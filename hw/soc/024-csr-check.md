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
