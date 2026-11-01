좋습니다. 이제 **12-9 — Mini CPU 전체 연결**로 갑니다.

이번 단계가 Project 12의 핵심입니다.

지금까지 검증한 블록들을 연결합니다.

```text
        ┌────────┐
        │   PC   │
        └───┬────┘
            │
            ▼
     ┌─────────────┐
     │ Instruction │
     │     ROM     │
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │   Decoder   │
     └──────┬──────┘
            │
       ┌────┴─────┐
       ▼          ▼
    rs1/rs2       rd
       │
       ▼
 ┌──────────────┐
 │ RegisterFile │
 └──────┬───────┘
        │
        ▼
     ┌─────┐
     │ ALU │
     └──┬──┘
        │
        └──────> Write Back
```

## 12-9 MiniCpu

기존 블록들을 그대로 재사용합니다.

```scala
import spinal.core._

class MiniCpu extends Component {
  val io = new Bundle {
    // 외부에서 register 초기화
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    val pc          = out UInt(4 bits)
    val instruction = out Bits(16 bits)
    val result      = out Bits(8 bits)
  }

  val fetch   = new FetchUnit
  val decoder = new InstructionDecoder
  val regFile = new DualReadRegisterFile
  val alu     = new SimpleAlu

  // --------------------------------------------------
  // Fetch
  // --------------------------------------------------

  fetch.io.enable := True

  io.pc          := fetch.io.pc
  io.instruction := fetch.io.instruction

  // --------------------------------------------------
  // Decode
  // --------------------------------------------------

  decoder.io.instruction := fetch.io.instruction

  // --------------------------------------------------
  // Register File Read
  // --------------------------------------------------

  regFile.io.readAddressA := decoder.io.rs1
  regFile.io.readAddressB := decoder.io.rs2

  // --------------------------------------------------
  // ALU
  // --------------------------------------------------

  alu.io.a      := regFile.io.readDataA.asUInt
  alu.io.b      := regFile.io.readDataB.asUInt
  alu.io.opcode := decoder.io.opcode

  io.result := alu.io.result.asBits

  // --------------------------------------------------
  // Register Write
  // --------------------------------------------------

  regFile.io.writeEnable  := io.loadEnable
  regFile.io.writeAddress := io.loadAddress
  regFile.io.writeData    := io.loadData

  when(decoder.io.valid) {
    regFile.io.writeEnable  := True
    regFile.io.writeAddress := decoder.io.rd
    regFile.io.writeData    := alu.io.result.asBits
  }
}
```

### 여기서 중요한 점

이제 CPU 외부에서 instruction을 직접 넣지 않습니다.

```text
❌ 이전

외부 → instruction → CPU
```

가 아니라,

```text
✅ 현재

PC → ROM → instruction → CPU
```

입니다.

---

# Simulation

우선 R0와 R1을 초기화합니다.

그 다음 ROM에 들어 있는 명령어를 CPU가 실제로 가져오는지 확인합니다.

```scala
import spinal.core.sim._

object MiniCpuSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new MiniCpu
    ).doSim { dut =>

      dut.clockDomain.forkStimulus(10)

      // ----------------------------------------------
      // R0 = 0x10
      // ----------------------------------------------

      dut.io.loadEnable  #= true
      dut.io.loadAddress #= 0
      dut.io.loadData    #= 0x10

      dut.clockDomain.waitSampling()

      // ----------------------------------------------
      // R1 = 0x20
      // ----------------------------------------------

      dut.io.loadAddress #= 1
      dut.io.loadData    #= 0x20

      dut.clockDomain.waitSampling()

      dut.io.loadEnable #= false

      // ----------------------------------------------
      // PC = 2
      //
      // FetchUnit은 매 clock마다 PC를 증가시키므로
      // 현재 PC 위치는 초기화 clock 수만큼 진행됨.
      //
      // 여기서는 현재 fetch 상태를 직접 확인한다.
      // ----------------------------------------------

      sleep(1)

      println(
        f"PC=${dut.io.pc.toInt} " +
        f"instruction=0x${dut.io.instruction.toBigInt}%04X " +
        f"result=0x${dut.io.result.toBigInt}%02X"
      )

      println("Mini CPU integration test completed.")
    }
  }
}
```

그런데 여기서 **중요한 문제가 하나 있습니다.**

현재 `MiniCpu`에서:

```scala
fetch.io.enable := True
```

이므로 **CPU가 매 clock마다 계속 실행됩니다.**

따라서 단순히 `R0/R1`을 load하는 동안에도 PC가 진행됩니다.

그리고 더 중요한 문제가 있습니다.

현재 ROM의 명령어는:

```text
0x0810 = ADD R2, R0, R1
```

이므로 R0/R1 초기화 직후 바로 실행될 수 있지만, 그 다음 clock에는 계속 다른 명령어가 실행됩니다.

즉 지금 구조는 **“프로그램 실행”과 “외부 register 초기화”가 동시에 존재하는 과도기 구조**입니다.

이 상태에서 무리하게 전체 CPU simulation을 작성하면 이전처럼 **clock scheduling 때문에 결과를 잘못 해석할 가능성**이 있습니다.

따라서 이번 단계에서는 먼저 **MiniCpu의 wiring이 맞는지 확인하고**, 다음 단계에서 `load`와 CPU 실행을 명확히 분리하는 **Reset/Run 구조**를 넣는 것이 올바릅니다.

### 그래서 이번 12-9의 목표

현재는 코드 연결 자체를 이해하는 것이 목적이고, **아직 “test passed”를 선언하지 않습니다.**

다음 **12-10**에서:

```text
RESET
  ↓
Register 초기화
  ↓
RUN
  ↓
PC → ROM → Decode → Execute → Write Back
```

구조를 정리하고 **실제로 ADD → SUB → AND → OR 명령어를 순차 실행하는 simulation**을 만들겠습니다.

이렇게 해야 Project 12를 깔끔하게 마무리할 수 있습니다.

---

좋습니다. **현재 12-9 Mini CPU 통합 테스트는 정상 실행된 것**으로 보입니다.

출력:

```text
PC=2 instruction=0x2D40 result=0x00
Mini CPU integration test completed.
```

다만 이 테스트는 **“시뮬레이션이 돌았다”는 것만 확인한 상태**이고, 아직 CPU가 각 명령을 실제로 순서대로 실행하고 레지스터에 write-back했는지는 충분히 검증하지 않았습니다.

특히 현재 구조에서는:

```text
PC=0  ADD
PC=1  SUB
PC=2  AND
PC=3  OR
```

가 진행되므로 `PC=2`, `0x2D40`까지 도달했다는 것은 **Fetch → ROM → Decode → ALU 경로가 연결되어 동작하고 있음**을 보여줍니다.

하지만 다음 단계에서는 중요한 문제 하나를 정리해야 합니다.

### 12-10 — CPU 실행 제어 정리

현재는 `fetch.io.enable := True`라서 CPU가 항상 실행됩니다.

다음에는:

```text
             ┌─────────────┐
loadEnable → │ RegisterFile│
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
PC ────────→ │ Instruction │
             │    ROM      │
             └──────┬──────┘
                    ▼
                 Decode
                    ▼
                  ALU
                    │
                    ▼
               Write Back
```

에 **`run` 제어**를 넣겠습니다.

즉,

```text
run = false
    ↓
레지스터 초기화
    ↓
run = true
    ↓
PC → Fetch → Decode → ALU → Write Back
```

이렇게 분리합니다.

그리고 다음 테스트에서는 단순히

```text
PC=2 ...
```

만 출력하지 않고,

```text
ADD → R2 확인
SUB → R1 확인
AND → R3 확인
OR  → R2 확인
```

처럼 **실제로 register write-back까지 검증**하도록 만들겠습니다.

이 단계가 끝나면 **Project 12의 핵심적인 CPU 구조가 완성**되고, 그 다음은 Project 13의 **Pipeline / Stage**로 넘어가면 됩니다.


---

좋습니다. 이제 **Project 12-10 — Mini CPU 실행 제어 + Write-Back 검증**으로 갑니다.

이번 단계의 핵심은 하나입니다.

> **레지스터 초기화(`load`)와 CPU 실행(`run`)을 분리한다.**

그리고 이번에는 단순히 `result`만 보는 것이 아니라 **실제로 어느 레지스터에 어떤 값이 write-back 되었는지**까지 확인합니다.

---

# Project 12-10 — Mini CPU Run Control

## 1. 먼저 CPU 구조를 정리

이번 구조는 다음과 같습니다.

```text
                  run
                   │
                   ▼
             ┌──────────┐
             │    PC    │
             └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │   ROM    │
             └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │ Decoder  │
             └────┬─────┘
                  │
             ┌────┴────┐
             ▼         ▼
           rs1         rs2
             │         │
             └────┬────┘
                  ▼
             ┌──────────┐
             │Register  │
             │  File    │
             └────┬─────┘
                  │
                  ▼
                ALU
                  │
                  ▼
             Write Back
```

그리고 초기화와 실행을 분리합니다.

```text
loadEnable = 1
run        = 0
       ↓
   Register 초기화

loadEnable = 0
run        = 1
       ↓
     CPU 실행
```

---

# 2. 수정된 MiniCpu

```scala
import spinal.core._

class MiniCpu extends Component {
  val io = new Bundle {
    // CPU 실행
    val run = in Bool()

    // 외부 Register 초기화
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // 관찰용 출력
    val pc          = out UInt(4 bits)
    val instruction = out Bits(16 bits)

    val result = out Bits(8 bits)

    // Write-back 관찰용
    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)
  }

  val fetch   = new FetchUnit
  val decoder = new InstructionDecoder
  val regFile = new DualReadRegisterFile
  val alu     = new SimpleAlu

  // ------------------------------------------------------------
  // Fetch
  // ------------------------------------------------------------

  fetch.io.enable := io.run

  io.pc          := fetch.io.pc
  io.instruction := fetch.io.instruction

  // ------------------------------------------------------------
  // Decode
  // ------------------------------------------------------------

  decoder.io.instruction := fetch.io.instruction

  // ------------------------------------------------------------
  // Register File Read
  // ------------------------------------------------------------

  regFile.io.readAddressA := decoder.io.rs1
  regFile.io.readAddressB := decoder.io.rs2

  // ------------------------------------------------------------
  // ALU
  // ------------------------------------------------------------

  alu.io.a      := regFile.io.readDataA.asUInt
  alu.io.b      := regFile.io.readDataB.asUInt
  alu.io.opcode := decoder.io.opcode

  io.result := alu.io.result.asBits

  // ------------------------------------------------------------
  // Register Write
  // ------------------------------------------------------------

  regFile.io.writeEnable  := False
  regFile.io.writeAddress := 0
  regFile.io.writeData    := 0

  io.writeBackEnable  := False
  io.writeBackAddress := 0
  io.writeBackData    := 0

  // 외부 초기화가 우선
  when(io.loadEnable) {

    regFile.io.writeEnable  := True
    regFile.io.writeAddress := io.loadAddress
    regFile.io.writeData    := io.loadData

    io.writeBackEnable  := True
    io.writeBackAddress := io.loadAddress
    io.writeBackData    := io.loadData

  } elsewhen(io.run && decoder.io.valid) {

    regFile.io.writeEnable  := True
    regFile.io.writeAddress := decoder.io.rd
    regFile.io.writeData    := alu.io.result.asBits

    io.writeBackEnable  := True
    io.writeBackAddress := decoder.io.rd
    io.writeBackData    := alu.io.result.asBits
  }
}
```

### 중요한 변경점

이전 코드와 달리 이제:

```scala
fetch.io.enable := io.run
```

입니다.

따라서:

```text
run = false → PC 정지
run = true  → PC 증가
```

가 됩니다.

또한 write-back도 명확합니다.

```scala
when(io.loadEnable) {
    // 외부 초기화
} elsewhen(io.run && decoder.io.valid) {
    // CPU 실행 결과
}
```

따라서 **초기화와 CPU 실행이 동시에 충돌하지 않습니다.**

---

# 3. 이번 ROM 프로그램

현재 ROM을 그대로 사용합니다.

```text
PC  Instruction

0   0x0810   ADD R2, R0, R1
1   0x1940   SUB R1, R1, R0
2   0x2D40   AND R3, R1, R1
3   0x3A00   OR  R2, R2, R0
```

초기 레지스터는:

```text
R0 = 0x10
R1 = 0x20
R2 = 0x00
R3 = 0x00
```

따라서 예상 결과는:

### Instruction 0

```text
ADD R2, R0, R1

R2 = 0x10 + 0x20
   = 0x30
```

### Instruction 1

```text
SUB R1, R1, R0

R1 = 0x20 - 0x10
   = 0x10
```

### Instruction 2

```text
AND R3, R1, R1

R3 = 0x10 & 0x10
   = 0x10
```

### Instruction 3

```text
OR R2, R2, R0

R2 = 0x30 | 0x10
   = 0x30
```

최종적으로:

```text
R0 = 0x10
R1 = 0x10
R2 = 0x30
R3 = 0x10
```

---

# 4. Simulation

이번에는 내부 register를 `simPublic()`으로 억지로 열지 않습니다.

CPU가 밖으로 내보내는:

```text
writeBackEnable
writeBackAddress
writeBackData
```

를 이용해서 **CPU가 실제로 무엇을 write-back 했는지** scoreboard로 확인합니다.

```scala
import spinal.core.sim._

object MiniCpuSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new MiniCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        dut.io.run #= false
        dut.io.loadEnable #= false
        dut.io.loadAddress #= 0
        dut.io.loadData #= 0

        // --------------------------------------------------------
        // Register 초기화
        // --------------------------------------------------------

        def loadRegister(address: Int, data: Int): Unit = {
          dut.io.loadEnable #= true
          dut.io.loadAddress #= address
          dut.io.loadData #= data

          dut.clockDomain.waitSampling()

          println(
            f"LOAD R$address%d <= 0x$data%02X"
          )
        }

        loadRegister(0, 0x10)
        loadRegister(1, 0x20)
        loadRegister(2, 0x00)
        loadRegister(3, 0x00)

        dut.io.loadEnable #= false

        // --------------------------------------------------------
        // CPU 실행
        // --------------------------------------------------------

        dut.io.run #= true

        // 현재 PC=0 instruction을 확인
        sleep(1)

        assert(dut.io.pc.toInt == 0)
        assert(dut.io.instruction.toBigInt == 0x0810)

        println(
          f"EXEC PC=${dut.io.pc.toInt}%d " +
          f"INST=0x${dut.io.instruction.toBigInt}%04X " +
          f"RESULT=0x${dut.io.result.toBigInt}%02X"
        )

        assert(dut.io.result.toBigInt == 0x30)

        // ADD R2,R0,R1 실행
        dut.clockDomain.waitSampling()

        assert(dut.io.writeBackEnable.toBoolean)
        assert(dut.io.writeBackAddress.toInt == 2)
        assert(dut.io.writeBackData.toBigInt == 0x30)

        println(
          f"WRITE R${dut.io.writeBackAddress.toInt}%d " +
          f"<= 0x${dut.io.writeBackData.toBigInt}%02X"
        )

        // --------------------------------------------------------
        // SUB R1,R1,R0
        // --------------------------------------------------------

        sleep(1)

        assert(dut.io.pc.toInt == 1)
        assert(dut.io.instruction.toBigInt == 0x1940)
        assert(dut.io.result.toBigInt == 0x10)

        println(
          f"EXEC PC=${dut.io.pc.toInt}%d " +
          f"INST=0x${dut.io.instruction.toBigInt}%04X " +
          f"RESULT=0x${dut.io.result.toBigInt}%02X"
        )

        dut.clockDomain.waitSampling()

        assert(dut.io.writeBackAddress.toInt == 1)
        assert(dut.io.writeBackData.toBigInt == 0x10)

        println(
          f"WRITE R${dut.io.writeBackAddress.toInt}%d " +
          f"<= 0x${dut.io.writeBackData.toBigInt}%02X"
        )

        // --------------------------------------------------------
        // AND R3,R1,R1
        // --------------------------------------------------------

        sleep(1)

        assert(dut.io.pc.toInt == 2)
        assert(dut.io.instruction.toBigInt == 0x2D40)
        assert(dut.io.result.toBigInt == 0x10)

        println(
          f"EXEC PC=${dut.io.pc.toInt}%d " +
          f"INST=0x${dut.io.instruction.toBigInt}%04X " +
          f"RESULT=0x${dut.io.result.toBigInt}%02X"
        )

        dut.clockDomain.waitSampling()

        assert(dut.io.writeBackAddress.toInt == 3)
        assert(dut.io.writeBackData.toBigInt == 0x10)

        println(
          f"WRITE R${dut.io.writeBackAddress.toInt}%d " +
          f"<= 0x${dut.io.writeBackData.toBigInt}%02X"
        )

        // --------------------------------------------------------
        // OR R2,R2,R0
        // --------------------------------------------------------

        sleep(1)

        assert(dut.io.pc.toInt == 3)
        assert(dut.io.instruction.toBigInt == 0x3A00)
        assert(dut.io.result.toBigInt == 0x30)

        println(
          f"EXEC PC=${dut.io.pc.toInt}%d " +
          f"INST=0x${dut.io.instruction.toBigInt}%04X " +
          f"RESULT=0x${dut.io.result.toBigInt}%02X"
        )

        dut.clockDomain.waitSampling()

        assert(dut.io.writeBackAddress.toInt == 2)
        assert(dut.io.writeBackData.toBigInt == 0x30)

        println(
          f"WRITE R${dut.io.writeBackAddress.toInt}%d " +
          f"<= 0x${dut.io.writeBackData.toBigInt}%02X"
        )

        // --------------------------------------------------------
        // 완료
        // --------------------------------------------------------

        dut.io.run #= false

        println("Mini CPU run/write-back test passed.")
      }
  }
}
```

## 5. 여기서 중요한 simulation 포인트

이번 테스트에서는 의도적으로:

```scala
sleep(1)
```

과

```scala
waitSampling()
```

을 구분했습니다.

### `sleep(1)`

현재 instruction/ALU 결과를 **clock edge 전에 관찰**합니다.

```text
PC=0
instruction=ADD
result=0x30
```

### `waitSampling()`

그 instruction의 **write-back이 발생하는 clock edge**를 통과합니다.

```text
clock edge
    ↓
R2 <= 0x30
PC  <= 1
```

따라서 이번에는 이전에 발생했던 것처럼 `waitSampling()` 이후의 `PC`를 보고 이전 instruction의 결과라고 착각하지 않도록 했습니다.

---

## 6. 성공하면 이런 형태

예상 로그:

```text
LOAD R0 <= 0x10
LOAD R1 <= 0x20
LOAD R2 <= 0x00
LOAD R3 <= 0x00

EXEC PC=0 INST=0x0810 RESULT=0x30
WRITE R2 <= 0x30

EXEC PC=1 INST=0x1940 RESULT=0x10
WRITE R1 <= 0x10

EXEC PC=2 INST=0x2D40 RESULT=0x10
WRITE R3 <= 0x10

EXEC PC=3 INST=0x3A00 RESULT=0x30
WRITE R2 <= 0x30

Mini CPU run/write-back test passed.
```

이 테스트가 통과하면 **Project 12의 Mini CPU 기본 구조는 사실상 완료**입니다.

그 다음 **Project 13에서 Pipeline을 시작**하면서 지금의

```text
Fetch → Decode → Execute → Write Back
```

을 실제 여러 stage로 분리하게 됩니다.

