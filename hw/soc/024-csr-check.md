PASS — Unmapped Load Error 처리 성공

좋습니다! 이번 결과는 기대한 동작과 정확히 일치합니다.

## 1. 실행 결과 분석

| Cycle | 동작             | 결과                             |
| ----- | -------------- | ------------------------------ |
| 9     | RAM Store      | `RAM[0] = 1`                   |
| 15    | Unmapped Load  | `0x20000000` 요청                |
| 16    | Error Response | `ready=1`, `error=1`, `data=0` |

특히 다음 두 가지가 중요합니다.

① Decoder의 Error Response 생성 성공

```
Cycle  15 : LOAD addr=0x20000000
Cycle  16 : LOAD RSP addr=0x20000000 data=0x00000000 error=true
```

요청 다음 Cycle에 정확히 Error Response가 반환되었습니다.

② CPU의 후속 명령 실행 차단 확인

```
[PASS] Store after Load Access Fault was not executed
```

Load Access Fault 이후의 Store가 실행되지 않았습니다. 따라서 CPU가 오류 응답을 정상적인 Load 데이터로 처리하지 않았다는 것을 확인했습니다.

## 2. 현재까지 완성된 DBus 구조

```
VexRiscv CPU(DBusSimplePlugin · catchAccessFault = true)
    |
    v
DBusDecoder(주소 디코딩 및 응답 라우팅)
    |                      |
    v                      v
RAM Hit                 Unmapped Load
SimpleDataRam           Error Response
정상 Read / Write        Access Fault
```

## 3. 다음 단계: CPU Trap 정보 검증

이제 단순히 후속 Store가 실행되지 않았다는 것에서 한 단계 더 나아가겠습니다.

RISC-V Load Access Fault가 발생했을 때 CPU의 CSR에 실제로 다음 정보가 기록되는지 확인하는 단계입니다.

| CSR      | 기대값          | 의미                   |
| -------- | ------------ | -------------------- |
| `mcause` | `5`          | Load Access Fault    |
| `mtval`  | `0x20000000` | 오류가 발생한 주소           |
| `mepc`   | `0x8000000C` | 오류를 발생시킨 Load 명령어 주소 |

이 검증이 성공하면 Decoder의 Error Response가 CPU의 RISC-V 예외 처리 경로까지 정상적으로 전달되었다고 훨씬 강하게 확인할 수 있습니다.

다음은 CSR Trap 검증으로 진행하겠습니다. 현재 `CsrPluginConfig.smallest`의 설정과 VexRiscv의 CSR 접근 방법을 확인한 다음, 기존 코드를 최대한 유지하면서 시뮬레이션을 추가하겠습니다.

---

---

좋습니다. 이번 단계는 RISC-V Trap 진입 후 CSR에 기록된 예외 정보를 실제 CPU 명령어로 읽어 검증하겠습니다.

먼저 현재 설정에서 중요한 사실을 확인했습니다.

`CsrPluginConfig.smallest`의 기본 구성은 다음과 같습니다.

| CSR                | 현재 접근 설정  | 이번 변경     |
| ------------------ | --------- | --------- |
| `mcause` (`0x342`) | READ_ONLY | 유지        |
| `mepc` (`0x341`)   | NONE      | READ_ONLY |
| `mtval` (`0x343`)  | NONE      | READ_ONLY |

VexRiscv 소스에서 `mbadaddrAccess`는 내부 `mtval` 레지스터에 연결됩니다. 이름은 예전 명칭이지만, 이번 검증에서는 RISC-V의 `mtval`로 이해하면 됩니다.

또 한 가지 변경이 필요합니다. 기본 `mtvecInit`은 `0x20`이므로, Trap Handler를 ROM의 `0x80000020`에 배치하고 Trap 진입 주소도 그곳으로 지정하겠습니다.

## 1. CpuConfig.scala 수정

`smallest()`를 그대로 사용하는 대신, 그 설정을 복사해서 필요한 CSR 접근 권한과 Trap Vector만 변경합니다.

기존:

```scala
new CsrPlugin(
  CsrPluginConfig.smallest
),
```

다음으로 교체합니다.

```scala
val csrConfig =
  CsrPluginConfig
    .smallest(BigInt("80000020", 16))
    .copy(
      mepcAccess = CsrAccess.READ_ONLY,
      mbadaddrAccess = CsrAccess.READ_ONLY
    )
```

그리고 Plugin 목록에서는 다음과 같이 사용합니다.

```
new CsrPlugin(csrConfig),
```

`mcauseAccess`는 이미 `READ_ONLY`이므로 변경할 필요가 없습니다.

`mtvecAccess`도 `NONE` 그대로 둡니다. 소프트웨어에서 `mtvec`를 읽거나 쓸 필요는 없고, 초기값만 지정하면 되기 때문입니다.

VexRiscv는 `mtvecInit`을 내부적으로 `mtvec.base`에 설정하고 Trap 발생 시 해당 Base 주소로 이동합니다.

## 2. InstructionRom.scala 수정

이제 프로그램이 다음 순서로 실행됩니다.

Main Program

1. RAM[0]에 1 저장
2. `0x20000000`에서 Load 시도
3. Load Access Fault 발생
4. Trap Vector `0x80000020`으로 이동

Trap Handler

1. `mcause` 읽기 → x6
2. `mepc` 읽기 → x7
3. `mtval` 읽기 → x8
4. 각 값을 RAM에 저장
5. 무한 루프

다음 코드로 전체 교체합니다.

```scala
package myvexsoc

import spinal.core._
import spinal.lib._
import vexriscv.plugin._

class InstructionRom extends Component {

  val io = new Bundle {
    val cmd = slave(Stream(IBusSimpleCmd()))
    val rsp = master(Flow(IBusSimpleRsp()))
  }

  // Main program
  val ADDI_X1_X0_1 = B"x00100093"
  val SW_X1_0_X0   = B"x00102023"

  val LUI_X4_20000 = B"x20000237"
  val LW_X3_0_X4   = B"x00022183"

  val SW_X3_8_X0   = B"x00302423"

  // Trap handler: CSR read
  // CSRRS x6, mcause, x0
  val CSRRS_X6_MCAUSE = B"x34202373"

  // CSRRS x7, mepc, x0
  val CSRRS_X7_MEPC = B"x341023F3"

  // CSRRS x8, mtval, x0
  val CSRRS_X8_MTVAL = B"x34302473"

  // Trap handler: Store CSR values to RAM
  val SW_X6_16_X0 = B"x00602823"
  val SW_X7_20_X0 = B"x00702A23"
  val SW_X8_24_X0 = B"x00802C23"

  val JAL_X0_0 = B"x0000006F"
  val NOP      = B"x00000013"

  io.cmd.ready := True

  val instruction = Bits(32 bits)
  instruction := NOP

  switch(io.cmd.pc) {

    // --------------------------------
    // Main program
    // --------------------------------

    // x1 = 1
    is(U"32'h80000000") {
      instruction := ADDI_X1_X0_1
    }

    // RAM[0] = 1
    is(U"32'h80000004") {
      instruction := SW_X1_0_X0
    }

    // x4 = 0x20000000
    is(U"32'h80000008") {
      instruction := LUI_X4_20000
    }

    // Load from unmapped address
    is(U"32'h8000000C") {
      instruction := LW_X3_0_X4
    }

    // Must not execute after Load Access Fault
    is(U"32'h80000010") {
      instruction := SW_X3_8_X0
    }

    is(U"32'h80000014") {
      instruction := JAL_X0_0
    }

    // --------------------------------
    // Trap handler: 0x80000020
    // --------------------------------

    // x6 = mcause
    is(U"32'h80000020") {
      instruction := CSRRS_X6_MCAUSE
    }

    // x7 = mepc
    is(U"32'h80000024") {
      instruction := CSRRS_X7_MEPC
    }

    // x8 = mtval
    is(U"32'h80000028") {
      instruction := CSRRS_X8_MTVAL
    }

    // RAM[4] = mcause
    is(U"32'h8000002C") {
      instruction := SW_X6_16_X0
    }

    // RAM[5] = mepc
    is(U"32'h80000030") {
      instruction := SW_X7_20_X0
    }

    // RAM[6] = mtval
    is(U"32'h80000034") {
      instruction := SW_X8_24_X0
    }

    // Infinite loop
    is(U"32'h80000038") {
      instruction := JAL_X0_0
    }
  }

  // One-cycle instruction response
  io.rsp.valid := RegNext(io.cmd.fire) init (False)

  io.rsp.inst :=
    RegNextWhen(instruction, io.cmd.fire) init (NOP)

  io.rsp.error := False
}
```

참고로 CSR 명령어는 `CSRRS rd, csr, x0` 형식입니다. `rs1=x0`이므로 CSR 값을 읽기만 하고 변경하지 않습니다.

## 3. VexRiscvTopSim.scala 전체 코드

기존 테스트벤치를 다음 코드로 교체합니다.

이번에는 단순히 Error Response만 확인하지 않습니다. Trap Handler가 실행한 세 번의 Store를 통해 CPU 내부 CSR 값까지 확인합니다.

```scala
package myvexsoc

import spinal.core.sim._
import vexriscv.plugin._

import scala.collection.mutable

object VexRiscvTopSim extends App {

  SimConfig.withWave
    .compile(new VexRiscvTop)
    .doSim { dut =>

      println("========================================")
      println(" VexRiscv Trap CSR Verification Test")
      println("========================================")

      dut.clockDomain.forkStimulus(period = 10)

      val dbusPlugin = dut.system.cpu.plugins
        .collectFirst {
          case p: DBusSimplePlugin => p
        }
        .getOrElse {
          throw new Exception("DBusSimplePlugin not found")
        }

      val dbus = dbusPlugin.dBus

      val stores = mutable.Map.empty[Long, Long]

      var storeCount = 0
      var loadCount = 0
      var errorResponseCount = 0

      var loadPending = false
      var pendingLoadAddress = 0L

      for (cycle <- 1 to 120) {

        dut.clockDomain.waitFallingEdge()

        // --------------------------------
        // 1. Load response
        // --------------------------------

        val rspValid = dbus.rsp.ready.toBoolean

        if (rspValid) {

          assert(
            loadPending,
            s"Unexpected Load response at cycle $cycle"
          )

          val rspError = dbus.rsp.error.toBoolean
          val rspData = dbus.rsp.data.toLong

          println(
            f"Cycle $cycle%3d : LOAD RSP " +
              f"addr=0x$pendingLoadAddress%08X " +
              f"data=0x$rspData%08X error=$rspError"
          )

          assert(pendingLoadAddress == 0x20000000L)
          assert(rspError, "Expected Load Access Fault response")
          assert(rspData == 0L)

          errorResponseCount += 1
          loadPending = false
        }

        // --------------------------------
        // 2. DBus command
        // --------------------------------

        val cmdValid = dbus.cmd.valid.toBoolean
        val cmdReady = dbus.cmd.ready.toBoolean

        if (cmdValid && cmdReady) {

          val wr = dbus.cmd.wr.toBoolean
          val address = dbus.cmd.address.toLong
          val data = dbus.cmd.payload.data.toLong
          val mask = dbus.cmd.mask.toInt

          if (wr) {

            storeCount += 1
            stores(address) = data

            println(
              f"Cycle $cycle%3d : STORE " +
                f"addr=0x$address%08X " +
                f"data=0x$data%08X mask=0x$mask%X"
            )

            assert(mask == 0xF, f"Unexpected Store mask: 0x$mask%X")

            assert(
              Set(0x00000000L, 0x00000010L, 0x00000014L, 0x00000018L)
                .contains(address),
              f"Unexpected Store address: 0x$address%08X"
            )

          } else {

            loadCount += 1
            loadPending = true
            pendingLoadAddress = address

            println(
              f"Cycle $cycle%3d : LOAD addr=0x$address%08X"
            )

            assert(
              address == 0x20000000L,
              f"Unexpected Load address: 0x$address%08X"
            )
          }
        }
      }

      // --------------------------------
      // 3. Final verification
      // --------------------------------

      println()
      println("========================================")
      println(" Trap CSR Verification Result")
      println("========================================")

      assert(!loadPending, "Load response is still pending")

      assert(storeCount == 4,
        s"Expected 4 Stores, observed $storeCount")

      assert(loadCount == 1,
        s"Expected 1 Load, observed $loadCount")

      assert(errorResponseCount == 1,
        s"Expected 1 Error Response, observed $errorResponseCount")

      // Initial RAM Store
      assert(
        stores.getOrElse(0x00000000L, -1L) == 1L,
        "RAM[0] verification failed"
      )

      // Trap CSR values
      val mcause = stores.getOrElse(0x00000010L, -1L)
      val mepc   = stores.getOrElse(0x00000014L, -1L)
      val mtval  = stores.getOrElse(0x00000018L, -1L)

      println(f"mcause = 0x$mcause%08X")
      println(f"mepc   = 0x$mepc%08X")
      println(f"mtval  = 0x$mtval%08X")

      assert(
        mcause == 5L,
        f"mcause mismatch: expected 5, got 0x$mcause%08X"
      )

      assert(
        mepc == 0x8000000CL,
        f"mepc mismatch: expected 0x8000000C, got 0x$mepc%08X"
      )

      assert(
        mtval == 0x20000000L,
        f"mtval mismatch: expected 0x20000000, got 0x$mtval%08X"
      )

      // The instruction after the fault must not execute.
      assert(
        !stores.contains(0x00000008L),
        "Instruction after Load Access Fault was executed!"
      )

      println()
      println("[PASS] RAM[0] = 0x00000001")
      println("[PASS] Load Access Fault response")
      println("[PASS] mcause = 5 (Load Access Fault)")
      println("[PASS] mepc = 0x8000000C")
      println("[PASS] mtval = 0x20000000")
      println("[PASS] Trap handler executed")
      println("[PASS] Faulting instruction was not re-executed")
      println("[PASS] Store after fault was not executed")

      println("========================================")
      println(" Trap CSR Verification finished")
      println("========================================")
    }
}
```

## 4. 이번 테스트에서 기대하는 결과

기대값

| RAM 주소 | 저장될 값        | 의미          |
| ------ | ------------ | ----------- |
| `0x00` | `0x00000001` | 기존 정상 Store |
| `0x10` | `0x00000005` | `mcause`    |
| `0x14` | `0x8000000C` | `mepc`      |
| `0x18` | `0x20000000` | `mtval`     |

Trap Handler의 실제 실행을 확인하는 핵심은 다음 세 값입니다.

$$ \begin{aligned} mcause &= 5\\\ mepc &= 0x8000000C\\\ mtval &= 0x20000000 \end{aligned} $$

VexRiscv의 DBus Plugin은 Access Fault가 발생하면 예외 코드 5와 오류 주소를 Exception Port에 전달합니다. CsrPlugin은 Trap 진입 시 그 정보를 `mcause`, `mepc`, `mtval`에 기록합니다.

주의: 위 코드는 아직 실제 실행 검증 전입니다. 특히 `mtval`의 주소 값과 CSR 명령어의 실행 결과를 이번 시뮬레이션에서 확인해야 합니다. 예상 로그를 실제 실행 결과처럼 간주하지 않겠습니다.

실행 후 전체 로그를 보내주세요. 세 CSR 값이 모두 일치하면 다음에는 Trap Handler에서 `MRET`으로 복귀하는 과정까지 구현하겠습니다.

---

---

```
[info] ========================================
[info]  VexRiscv Trap CSR Verification Test
[info] ========================================
[info] Cycle   9 : STORE addr=0x00000000 data=0x00000001 mask=0xF
[info] Cycle  15 : LOAD addr=0x20000000
[info] Cycle  16 : LOAD RSP addr=0x20000000 data=0x00000000 error=true
[info] Cycle  29 : STORE addr=0x00000010 data=0x00000005 mask=0xF
[info] Cycle  30 : STORE addr=0x00000014 data=0x8000000C mask=0xF
[info] Cycle  33 : STORE addr=0x00000018 data=0x20000000 mask=0xF
[info] ========================================
[info]  Trap CSR Verification Result
[info] ========================================
[info] mcause = 0x00000005
[info] mepc   = 0x8000000C
[info] mtval  = 0x20000000
[info] [PASS] RAM[0] = 0x00000001
[info] [PASS] Load Access Fault response
[info] [PASS] mcause = 5 (Load Access Fault)
[info] [PASS] mepc = 0x8000000C
[info] [PASS] mtval = 0x20000000
[info] [PASS] Trap handler executed
[info] [PASS] Faulting instruction was not re-executed
[info] [PASS] Store after fault was not executed
[info] ========================================
[info]  Trap CSR Verification finished
[info] ========================================

```

---

PASS — Trap CSR 검증 성공
훌륭합니다. 이번에는 단순히 Error Response를 반환하는 수준을 넘어, VexRiscv CPU가 실제 Load Access Fault Trap에 진입하고 CSR에 올바른 예외 정보를 기록한 것까지 확인했습니다.
1. 실행 결과 분석
이번 로그에서 가장 중요한 부분입니다.
```
Cycle  15 : LOAD addr=0x20000000
Cycle  16 : LOAD RSP addr=0x20000000 data=0x00000000 error=true

Cycle  29 : STORE addr=0x00000010 data=0x00000005
Cycle  30 : STORE addr=0x00000014 data=0x8000000C
Cycle  33 : STORE addr=0x00000018 data=0x20000000
```

Trap Response 이후 Trap Handler가 실행되었고, CSR 세 개를 읽어 RAM에 저장했습니다.
```
CSR	실제 저장값	검증 결과
mcause	0x00000005	PASS
mepc	0x8000000C	PASS
mtval	0x20000000	PASS
```
특히 mepc가 오류를 일으킨 Load 명령어의 주소이고, mtval이 접근에 실패한 주소라는 점까지 정확하게 일치합니다.
다음 단계: Trap Handler에서 MRET으로 복귀하기
이번에는 Trap Handler가 예외 정보를 기록한 뒤, mepc를 수정하고 MRET을 실행하도록 만들겠습니다.

핵심은 오류가 발생한 Load 명령어를 다시 실행하지 않고, 그 다음 명령어부터 CPU 실행을 재개하는 것입니다.

앞서 검증한 예외 발생 주소는 다음과 같습니다.
```
mepc = 0x8000000C
```

이 주소를 그대로 두고 MRET을 실행하면 같은 Load 명령어가 다시 실행되어 Trap이 반복됩니다. 따라서 Handler에서 mepc += 4를 수행해야 합니다.

사용 중인 VexRiscv commit의 CsrPlugin 소스를 확인했습니다. 이 버전은 MRET을 XRET 명령어로 디코딩하고, mepc를 복귀 PC로 사용하며 mstatus.MPIE/MIE와 MPP도 갱신합니다. 

## 1. CpuConfig.scala 수정
기존 설정에서 mepcAccess를 READ_ONLY에서 READ_WRITE로 변경합니다.
```scala
val csrConfig =
  CsrPluginConfig
    .smallest(BigInt("80000020", 16))
    .copy(
      mepcAccess = CsrAccess.READ_WRITE,
      mbadaddrAccess = CsrAccess.READ_ONLY
    )
```

mepc를 읽는 것뿐 아니라 Handler에서 수정해야 하기 때문입니다.

기존 Plugin 목록에서는 그대로 다음과 같이 사용합니다.
```
new CsrPlugin(csrConfig),
```

다른 Plugin 설정은 변경하지 않습니다.

## 2. 변경할 프로그램 흐름

이번 ROM 프로그램은 다음과 같이 구성합니다.

Main Program
1. RAM[0]에 1 저장
2. 0x20000000에서 Load → Access Fault
3. Trap Handler 진입

Trap Handler (0x80000020)
1. mcause, mepc, mtval 읽기
2. 세 값을 RAM에 보관
3. mepc = mepc + 4로 수정
4. MRET 실행

복귀 후
1. 0x80000010의 ADDI 실행
2. RAM[2]에 0x66 저장
3. 무한 루프

복귀 후 RAM[2]에 0x66이 기록되는 것이 이번 MRET 검증의 핵심입니다.

## 3. InstructionRom.scala 전체 교체

```scala
package myvexsoc

import spinal.core._
import spinal.lib._
import vexriscv.plugin._

class InstructionRom extends Component {

  val io = new Bundle {
    val cmd = slave(Stream(IBusSimpleCmd()))
    val rsp = master(Flow(IBusSimpleRsp()))
  }

  // Main program
  val ADDI_X1_X0_1 = B"x00100093"
  val SW_X1_0_X0   = B"x00102023"

  val LUI_X4_20000 = B"x20000237"
  val LW_X3_0_X4   = B"x00022183"

  // Resume point after MRET
  val ADDI_X9_X0_66 = B"x06600493"
  val SW_X9_8_X0    = B"x00902423"

  // Trap handler: read CSR
  val CSRRS_X6_MCAUSE = B"x34202373"
  val CSRRS_X7_MEPC   = B"x341023F3"
  val CSRRS_X8_MTVAL  = B"x34302473"

  // Store original CSR values
  val SW_X6_16_X0 = B"x00602823"
  val SW_X7_20_X0 = B"x00702A23"
  val SW_X8_24_X0 = B"x00802C23"

  // mepc = mepc + 4
  val ADDI_X7_X7_4 = B"x00438393"

  // CSRRW x0, mepc, x7
  val CSRRW_X0_MEPC_X7 = B"x34139073"

  // Return from trap
  val MRET = B"x30200073"

  val JAL_X0_0 = B"x0000006F"
  val NOP      = B"x00000013"

  io.cmd.ready := True

  val instruction = Bits(32 bits)
  instruction := NOP

  switch(io.cmd.pc) {

    // --------------------------------
    // Main program
    // --------------------------------

    // x1 = 1
    is(U"32'h80000000") {
      instruction := ADDI_X1_X0_1
    }

    // RAM[0] = 1
    is(U"32'h80000004") {
      instruction := SW_X1_0_X0
    }

    // x4 = 0x20000000
    is(U"32'h80000008") {
      instruction := LUI_X4_20000
    }

    // Faulting Load
    is(U"32'h8000000C") {
      instruction := LW_X3_0_X4
    }

    // Resume point: x9 = 0x66
    is(U"32'h80000010") {
      instruction := ADDI_X9_X0_66
    }

    // RAM[2] = 0x66
    is(U"32'h80000014") {
      instruction := SW_X9_8_X0
    }

    // Main program loop
    is(U"32'h80000018") {
      instruction := JAL_X0_0
    }

    // --------------------------------
    // Trap handler: 0x80000020
    // --------------------------------

    // x6 = mcause
    is(U"32'h80000020") {
      instruction := CSRRS_X6_MCAUSE
    }

    // x7 = original mepc
    is(U"32'h80000024") {
      instruction := CSRRS_X7_MEPC
    }

    // x8 = mtval
    is(U"32'h80000028") {
      instruction := CSRRS_X8_MTVAL
    }

    // RAM[4] = mcause
    is(U"32'h8000002C") {
      instruction := SW_X6_16_X0
    }

    // RAM[5] = original mepc
    is(U"32'h80000030") {
      instruction := SW_X7_20_X0
    }

    // RAM[6] = mtval
    is(U"32'h80000034") {
      instruction := SW_X8_24_X0
    }

    // x7 = original mepc + 4
    is(U"32'h80000038") {
      instruction := ADDI_X7_X7_4
    }

    // mepc = x7
    is(U"32'h8000003C") {
      instruction := CSRRW_X0_MEPC_X7
    }

    // Return to 0x80000010
    is(U"32'h80000040") {
      instruction := MRET
    }

    // Unexpected fall-through protection
    is(U"32'h80000044") {
      instruction := JAL_X0_0
    }
  }

  // One-cycle instruction response
  io.rsp.valid := RegNext(io.cmd.fire) init (False)

  io.rsp.inst :=
    RegNextWhen(instruction, io.cmd.fire) init (NOP)

  io.rsp.error := False
}
```

###  명령어 인코딩 확인
이번에 추가한 세 가지 명령어의 인코딩은 다음과 같습니다.

| 명령어 | 인코딩 |
|------|-------|
| ADDI x7,x7,4 | 0x00438393 |
| CSRRW x0,mepc,x7 | 0x34139073 |
| MRET | 0x30200073 |

CSRRW x0,mepc,x7은 mepc에 x7 값을 기록합니다. 목적 레지스터가 x0이므로 CSR의 이전 값은 일반 레지스터에 기록하지 않습니다.

## 4. VexRiscvTopSim.scala 전체 교체
이번에는 RAM에 기록된 원래 CSR 값과 MRET 이후의 실행 결과를 함께 검사합니다.

```scala
package myvexsoc

import spinal.core.sim._
import vexriscv.plugin._

import scala.collection.mutable

object VexRiscvTopSim extends App {

  SimConfig.withWave
    .compile(new VexRiscvTop)
    .doSim { dut =>

      println("========================================")
      println(" VexRiscv MRET Verification Test")
      println("========================================")

      dut.clockDomain.forkStimulus(period = 10)

      val dbusPlugin = dut.system.cpu.plugins
        .collectFirst {
          case p: DBusSimplePlugin => p
        }
        .getOrElse {
          throw new Exception("DBusSimplePlugin not found")
        }

      val dbus = dbusPlugin.dBus

      val stores = mutable.Map.empty[Long, Long]

      var storeCount = 0
      var loadCount = 0
      var errorResponseCount = 0

      var loadPending = false
      var pendingLoadAddress = 0L

      for (cycle <- 1 to 160) {

        dut.clockDomain.waitFallingEdge()

        // --------------------------------
        // 1. Load response
        // --------------------------------

        val rspValid = dbus.rsp.ready.toBoolean

        if (rspValid) {

          assert(
            loadPending,
            s"Unexpected Load response at cycle $cycle"
          )

          val rspError = dbus.rsp.error.toBoolean
          val rspData = dbus.rsp.data.toLong

          println(
            f"Cycle $cycle%3d : LOAD RSP " +
              f"addr=0x$pendingLoadAddress%08X " +
              f"data=0x$rspData%08X error=$rspError"
          )

          assert(pendingLoadAddress == 0x20000000L)
          assert(rspError, "Expected Load Access Fault response")
          assert(rspData == 0L)

          errorResponseCount += 1
          loadPending = false
        }

        // --------------------------------
        // 2. DBus command
        // --------------------------------

        val cmdValid = dbus.cmd.valid.toBoolean
        val cmdReady = dbus.cmd.ready.toBoolean

        if (cmdValid && cmdReady) {

          val wr = dbus.cmd.wr.toBoolean
          val address = dbus.cmd.address.toLong
          val data = dbus.cmd.payload.data.toLong
          val mask = dbus.cmd.mask.toInt

          if (wr) {

            storeCount += 1
            stores(address) = data

            println(
              f"Cycle $cycle%3d : STORE " +
                f"addr=0x$address%08X " +
                f"data=0x$data%08X mask=0x$mask%X"
            )

            assert(mask == 0xF)

            val allowedAddresses = Set(
              0x00000000L,
              0x00000008L,
              0x00000010L,
              0x00000014L,
              0x00000018L
            )

            assert(
              allowedAddresses.contains(address),
              f"Unexpected Store address: 0x$address%08X"
            )

          } else {

            loadCount += 1
            loadPending = true
            pendingLoadAddress = address

            println(
              f"Cycle $cycle%3d : LOAD addr=0x$address%08X"
            )

            assert(
              address == 0x20000000L,
              f"Unexpected Load address: 0x$address%08X"
            )
          }
        }
      }

      // --------------------------------
      // 3. Final verification
      // --------------------------------

      println()
      println("========================================")
      println(" MRET Verification Result")
      println("========================================")

      assert(!loadPending, "Load response is still pending")

      assert(
        storeCount == 5,
        s"Expected 5 Stores, observed $storeCount"
      )

      assert(
        loadCount == 1,
        s"Expected 1 Load, observed $loadCount"
      )

      assert(
        errorResponseCount == 1,
        s"Expected 1 Error Response, observed $errorResponseCount"
      )

      // Original normal Store
      assert(
        stores.getOrElse(0x00000000L, -1L) == 1L,
        "RAM[0] verification failed"
      )

      // Original Trap CSR values
      val mcause = stores.getOrElse(0x00000010L, -1L)
      val mepc   = stores.getOrElse(0x00000014L, -1L)
      val mtval  = stores.getOrElse(0x00000018L, -1L)

      println(f"mcause = 0x$mcause%08X")
      println(f"mepc   = 0x$mepc%08X")
      println(f"mtval  = 0x$mtval%08X")

      assert(mcause == 5L)
      assert(mepc == 0x8000000CL)
      assert(mtval == 0x20000000L)

      // MRET resume marker
      val resumeMarker = stores.getOrElse(0x00000008L, -1L)

      assert(
        resumeMarker == 0x00000066L,
        f"MRET resume marker mismatch: 0x$resumeMarker%08X"
      )

      println()
      println("[PASS] RAM[0] = 0x00000001")
      println("[PASS] Load Access Fault response")
      println("[PASS] mcause = 5")
      println("[PASS] Original mepc = 0x8000000C")
      println("[PASS] mtval = 0x20000000")
      println("[PASS] Trap handler updated mepc")
      println("[PASS] MRET executed")
      println("[PASS] CPU resumed at 0x80000010")
      println("[PASS] RAM[2] = 0x00000066")

      println("========================================")
      println(" MRET Verification finished")
      println("========================================")
    }
}
```

## 5. 기대 결과
다음은 실제 실행 결과가 아닌 기대값입니다.

| RAM 주소 | 기대 데이터 | 의미 |
|---------|-----------|-----|
| 0x00 | 0x00000001 | 정상 Store |
| 0x08 | 0x00000066 | MRET 복귀 후 실행 증명 |
| 0x10 | 0x00000005 | 원래 mcause |
| 0x14 | 0x8000000C | 원래 mepc |
| 0x18 | 0x20000000 | 원래 mtval |

최종적으로 다음 세 가지를 확인하면 됩니다.
- Trap Handler가 mepc를 0x80000010으로 수정했는가?
- MRET이 실제로 실행되었는가?
- CPU가 0x80000010부터 실행하여 RAM[2]에 0x66을 기록했는가?

### 로컬 실행 검증 대기
아직 제가 사용자의 프로젝트에서 컴파일하거나 실행한 코드는 아닙니다. 특히 MRET 이후의 PC 복귀와 CSR 쓰기 타이밍은 실제 시뮬레이션으로 확인해야 합니다.

실행 결과를 보내주세요. PASS가 확인되면 다음에는 Trap Handler에서 예외 원인을 분기 처리하고, 정상 복귀와 예외 종료를 구분하는 구조로 발전시키겠습니다.

---
