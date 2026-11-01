좋습니다. **Project 13-1 — 가장 단순한 2-Stage Pipeline**부터 시작하겠습니다.

이번 단계에서는 CPU 전체를 한꺼번에 파이프라인화하지 않고, **Pipeline Register 하나가 왜 필요한지**를 정확히 이해하는 것이 목표입니다.

---

# Project 13-1 — 2-Stage Pipeline

현재 CPU는:

```text
Fetch → Decode/Execute → Write Back
```

처럼 조합논리들이 직접 연결되어 있습니다.

이번에는 가운데에 레지스터를 넣습니다.

```text
Stage 0                    Stage 1

Fetch
  │
  ▼
┌──────────────┐
│ Pipeline Reg │
└──────┬───────┘
       │
       ▼
Decode / Execute
```

핵심은 **clock edge가 데이터의 경계를 만든다**는 것입니다.

---

## 1. 아주 작은 예제로 시작

8-bit 데이터를 두 번 연산한다고 해보겠습니다.

```text
Input
  │
  ▼
ADD 1
  │
  ▼
[Pipeline Register]
  │
  ▼
ADD 2
  │
  ▼
Output
```

Pipeline register가 없으면:

```text
Input → ADD1 → ADD2 → Output
```

모든 것이 하나의 긴 combinational path입니다.

Pipeline register를 넣으면:

```text
Input → ADD1 → Reg → ADD2 → Output
```

이제 clock 기준으로 두 stage가 분리됩니다.

---

# 2. 코드

```scala
import spinal.core._

class TwoStagePipeline extends Component {
  val io = new Bundle {
    val input  = in UInt(8 bits)
    val output = out UInt(8 bits)
  }

  // ------------------------------------------------------------
  // Stage 0
  // ------------------------------------------------------------

  val stage0 = io.input + 1

  // ------------------------------------------------------------
  // Pipeline Register
  // ------------------------------------------------------------

  val stage1 = Reg(UInt(8 bits)) init(0)

  stage1 := stage0

  // ------------------------------------------------------------
  // Stage 1
  // ------------------------------------------------------------

  io.output := stage1 + 1
}
```

이 회로를 시간축으로 보면:

```text
Cycle 0

input
  │
  ▼
 +1
  │
  ▼
stage1
```

clock edge가 지나면:

```text
Cycle 1

stage1 = input + 1

stage1
  │
  ▼
 +1
  │
  ▼
output
```

따라서 입력 `10`을 넣으면 최종 결과는 `12`입니다.

하지만 중요한 것은 **입력과 출력이 같은 cycle에 완성되지 않는다**는 것입니다.

---

# 3. Simulation

이번에도 simulation timing을 명확하게 보겠습니다.

```scala
import spinal.core.sim._

object TwoStagePipelineSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new TwoStagePipeline)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        dut.io.input #= 10

        // 초기 pipeline register 확인
        sleep(1)

        println(
          s"Before clock: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        // 첫 번째 clock
        dut.clockDomain.waitSampling()

        sleep(1)

        println(
          s"After clock 1: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        assert(dut.io.output.toInt == 12)

        println("Two-stage pipeline test passed.")
      }
  }
}
```

여기서 예상 결과는:

```text
Before clock: input=10 output=1
After clock 1: input=10 output=12
Two-stage pipeline test passed.
```

초기 `stage1 = 0`이므로 첫 번째 clock 이전에는:

```text
stage1 = 0
output = 0 + 1 = 1
```

입니다.

첫 번째 clock에서:

```text
stage1 <= input + 1
        <= 11
```

그리고 clock 이후:

```text
output = stage1 + 1
       = 12
```

가 됩니다.

---

# 4. 여기서 가장 중요한 것

Pipeline register:

```scala
val stage1 = Reg(UInt(8 bits)) init(0)

stage1 := stage0
```

이 두 줄이 핵심입니다.

`stage0`은 **현재 입력에 즉시 반응하는 combinational 값**이고,

`stage1`은 **clock edge에서 값을 저장하는 값**입니다.

따라서:

```text
stage0
  │
  │ combinational
  ▼
stage1
  │
  │ clock boundary
  ▼
next stage
```

가 됩니다.

---

# 5. 기존 CPU와 연결하면

지금 만든 것은 CPU Pipeline의 아주 작은 축소판입니다.

현재:

```text
Fetch → Decode → Execute → Write Back
```

을 앞으로:

```text
Fetch
  │
  ▼
[IF/ID Register]
  │
  ▼
Decode
  │
  ▼
[ID/EX Register]
  │
  ▼
Execute
  │
  ▼
[EX/WB Register]
  │
  ▼
Write Back
```

형태로 발전시킬 것입니다.

다만 **아직 바로 4-stage CPU로 가지 않습니다.**

먼저 이번 2-stage 예제에서 반드시 확인해야 할 것은:

> **Pipeline register 때문에 데이터가 clock edge를 기준으로 한 stage씩 이동한다.**

입니다.

### 이번 단계에서 실행할 것

위의 `TwoStagePipeline`과 `TwoStagePipelineSim`을 그대로 실행해 주세요.

성공 로그가

```text
Two-stage pipeline test passed.
```

까지 나오면 다음은 **Project 13-2 — 두 개의 입력을 연속으로 넣어 pipeline이 동시에 두 데이터를 처리하는 것**으로 넘어가겠습니다.

---

좋습니다. ✅ **Project 13-1 통과**입니다.

이번 단계에서 핵심인 **clock boundary를 통한 stage 분리**를 실제 simulation으로 확인했습니다.

다음은 훨씬 중요한 단계입니다.

# Project 13-2 — Pipeline Overlap

지금까지는 입력 하나만 넣었습니다.

```text
Cycle 0
Input A → Stage 0

Cycle 1
          A → Stage 1
          Input B → Stage 0

Cycle 2
                    A → Output
                    B → Stage 1
                    Input C → Stage 0
```

이것이 pipeline의 핵심인 **overlap**입니다.

즉,

> **Pipeline은 한 instruction의 실행을 반드시 빠르게 만드는 것이 아니라, 여러 instruction을 동시에 서로 다른 stage에서 처리해서 throughput을 높이는 구조입니다.**

---

## 1. Pipeline을 조금 확장

이번에는 입력마다 `+2`를 수행하는 2-stage pipeline을 만들겠습니다.

```text
Stage 0                 Stage 1

input + 1               + 1
   │                      ▲
   ▼                      │
[Pipeline Register] ──────┘
```

결과적으로:

```text
output = input + 2
```

입니다.

### 코드

```scala
import spinal.core._

class TwoStagePipeline extends Component {
  val io = new Bundle {
    val input  = in UInt(8 bits)
    val output = out UInt(8 bits)
  }

  // Stage 0
  val stage0 = io.input + 1

  // Pipeline register
  val stage1 = Reg(UInt(8 bits)) init(0)

  stage1 := stage0

  // Stage 1
  io.output := stage1 + 1
}
```

---

# 2. 연속 입력 Simulation

이번에는 입력을 계속 변경합니다.

중요한 점은 **첫 번째 결과가 나온 뒤 두 번째 결과를 넣는 것이 아니라**, 매 clock마다 새로운 데이터를 넣는다는 것입니다.

```scala
import spinal.core.sim._

object TwoStagePipelineSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new TwoStagePipeline)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        // 초기값
        dut.io.input #= 10

        sleep(1)

        println(
          s"Before clock: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        // --------------------------------------------------------
        // Cycle 1
        // --------------------------------------------------------

        dut.clockDomain.waitSampling()
        sleep(1)

        println(
          s"Cycle 1: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        assert(dut.io.output.toInt == 12)

        // 다음 데이터
        dut.io.input #= 20

        // --------------------------------------------------------
        // Cycle 2
        // --------------------------------------------------------

        dut.clockDomain.waitSampling()
        sleep(1)

        println(
          s"Cycle 2: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        assert(dut.io.output.toInt == 22)

        // 다음 데이터
        dut.io.input #= 30

        // --------------------------------------------------------
        // Cycle 3
        // --------------------------------------------------------

        dut.clockDomain.waitSampling()
        sleep(1)

        println(
          s"Cycle 3: input=${dut.io.input.toInt} " +
          s"output=${dut.io.output.toInt}"
        )

        assert(dut.io.output.toInt == 32)

        println("Pipeline overlap test passed.")
      }
  }
}
```

예상:

```text
Cycle 1: input=10 output=12
Cycle 2: input=20 output=22
Cycle 3: input=30 output=32
Pipeline overlap test passed.
```

---

## 3. 이것이 왜 중요한가?

Cycle별로 보면:

```text
              Stage 0        Stage 1
------------------------------------------------
Cycle 0       A
Cycle 1       B              A
Cycle 2       C              B
Cycle 3       D              C
```

즉 **Cycle 2에는 A와 B와 C가 순차적으로 끝나는 것이 아니라 서로 다른 stage에서 동시에 존재**합니다.

이것이 Pipeline의 본질입니다.

### Non-pipeline

```text
A: ██████████
B:           ██████████
C:                      ██████████
```

### Pipeline

```text
A: █████
B:     █████
C:         █████
D:             █████
```

따라서 stage가 균형 있게 설계되면 **instruction throughput이 증가**합니다.

---

이번 테스트까지 통과하면 다음 단계에서는 이 개념을 실제 CPU에 적용해서:

```text
Fetch
  ↓
[IF/ID]
  ↓
Decode
  ↓
[ID/EX]
  ↓
Execute
  ↓
[EX/WB]
  ↓
Write Back
```

형태의 **실제 pipelined Mini CPU**를 만들겠습니다.

---

좋습니다. 이번에는 **설명으로 돌리지 않고 바로 CPU 자체를 4-stage pipeline으로 바꾸겠습니다.**

기존 Mini CPU를 다음처럼 구성합니다.

```text
                4-Stage Pipeline

        ┌────────┐
        │ Fetch  │
        └───┬────┘
            │
        [ IF / ID ]
            │
            ▼
        ┌────────┐
        │ Decode │
        └───┬────┘
            │
        [ ID / EX ]
            │
            ▼
        ┌────────┐
        │ Execute│
        └───┬────┘
            │
        [ EX / WB ]
            │
            ▼
        Write Back
```

그리고 이번에는 **pipeline register 3개를 실제 CPU 안에 넣습니다.**

추가로 현재 프로그램에는 실제 data dependency가 있으므로, 단순 pipeline만 넣으면 `SUB → AND`에서 hazard가 발생합니다. 따라서 **EX/WB → ID/EX forwarding**도 같이 넣겠습니다.

---

# 1. 4-Stage Pipeline CPU

SpinalHDL 1.15.0 기준입니다.

```scala
import spinal.core._

class PipelinedMiniCpu extends Component {
  val io = new Bundle {
    // CPU control
    val run = in Bool()

    // Register initialization
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // Debug
    val pc          = out UInt(4 bits)
    val instruction = out Bits(16 bits)

    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)
  }

  // ============================================================
  // Components
  // ============================================================

  val pc        = new ProgramCounter
  val rom       = new InstructionRom
  val decoder   = new InstructionDecoder
  val regFile   = new DualReadRegisterFile
  val alu       = new SimpleAlu

  // ============================================================
  // Stage 0 : FETCH
  // ============================================================

  pc.io.enable := io.run

  rom.io.address := pc.io.value(1 downto 0)

  // ------------------------------------------------------------
  // IF / ID pipeline register
  // ------------------------------------------------------------

  val ifIdValid       = Reg(Bool()) init(False)
  val ifIdInstruction = Reg(Bits(16 bits)) init(0)
  val ifIdPc          = Reg(UInt(4 bits)) init(0)

  when(io.run) {
    ifIdValid       := True
    ifIdInstruction := rom.io.instruction
    ifIdPc          := pc.io.value
  } otherwise {
    ifIdValid := False
  }

  // ============================================================
  // Stage 1 : DECODE
  // ============================================================

  decoder.io.instruction := ifIdInstruction

  // ------------------------------------------------------------
  // ID / EX pipeline register
  // ------------------------------------------------------------

  val idExValid  = Reg(Bool()) init(False)
  val idExPc     = Reg(UInt(4 bits)) init(0)
  val idExRd     = Reg(UInt(2 bits)) init(0)
  val idExA      = Reg(Bits(8 bits)) init(0)
  val idExB      = Reg(Bits(8 bits)) init(0)
  val idExOpcode = Reg(UInt(2 bits)) init(0)

  // ============================================================
  // Register File Read
  // ============================================================

  regFile.io.readAddressA := decoder.io.rs1
  regFile.io.readAddressB := decoder.io.rs2

  // ============================================================
  // Forwarding
  // ============================================================

  val forwardedA = Bits(8 bits)
  val forwardedB = Bits(8 bits)

  forwardedA := regFile.io.readDataA
  forwardedB := regFile.io.readDataB

  // EX/WB → ID/EX forwarding
  when(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decoder.io.rs1
  ) {
    forwardedA := exWbWriteData
  }

  when(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decoder.io.rs2
  ) {
    forwardedB := exWbWriteData
  }

  // ------------------------------------------------------------
  // ID / EX register
  // ------------------------------------------------------------

  when(ifIdValid && decoder.io.valid) {
    idExValid  := True
    idExPc     := ifIdPc
    idExRd     := decoder.io.rd
    idExA      := forwardedA
    idExB      := forwardedB
    idExOpcode := decoder.io.opcode
  } otherwise {
    idExValid := False
  }

  // ============================================================
  // Stage 2 : EXECUTE
  // ============================================================

  alu.io.a      := idExA.asUInt
  alu.io.b      := idExB.asUInt
  alu.io.opcode := idExOpcode

  // ------------------------------------------------------------
  // EX / WB pipeline register
  // ------------------------------------------------------------

  val exWbValid       = Reg(Bool()) init(False)
  val exWbWriteEnable = Reg(Bool()) init(False)
  val exWbWriteAddress = Reg(UInt(2 bits)) init(0)
  val exWbWriteData   = Reg(Bits(8 bits)) init(0)

  when(idExValid) {
    exWbValid        := True
    exWbWriteEnable  := True
    exWbWriteAddress := idExRd
    exWbWriteData    := alu.io.result.asBits
  } otherwise {
    exWbValid       := False
    exWbWriteEnable := False
  }

  // ============================================================
  // Stage 3 : WRITE BACK
  // ============================================================

  regFile.io.writeEnable  := False
  regFile.io.writeAddress := 0
  regFile.io.writeData    := 0

  // External initialization has priority
  when(io.loadEnable) {

    regFile.io.writeEnable  := True
    regFile.io.writeAddress := io.loadAddress
    regFile.io.writeData    := io.loadData

  } elsewhen(exWbValid && exWbWriteEnable) {

    regFile.io.writeEnable  := True
    regFile.io.writeAddress := exWbWriteAddress
    regFile.io.writeData    := exWbWriteData
  }

  // ============================================================
  // Debug outputs
  // ============================================================

  io.pc          := pc.io.value
  io.instruction := rom.io.instruction

  io.writeBackEnable  := exWbValid && exWbWriteEnable
  io.writeBackAddress := exWbWriteAddress
  io.writeBackData    := exWbWriteData
}
```

여기서 중요한 점이 하나 있습니다.

`exWbValid`, `exWbWriteEnable`, `exWbWriteAddress`, `exWbWriteData`는 **EX/WB register**이므로 forwarding에서 사용됩니다.

즉:

```text
                 EX/WB
                   │
                   │ forwarding
                   ▼
                ID/EX
```

구조입니다.

---

# 2. 한 가지 수정

위 코드에서는 Scala의 선언 순서상 `exWbValid` 등을 forwarding에서 사용하기 전에 선언해야 합니다.

따라서 **실제 파일에는 EX/WB register 선언을 forwarding보다 위로 이동**해야 합니다.

완성본에서는 순서를 이렇게 잡는 것이 안전합니다.

```scala
// IF/ID
// Decode
// EX/WB register 선언
// Register read
// Forwarding
// ID/EX
// Execute
// EX/WB 값 갱신
// Write Back
```

즉, 아래처럼 `EX/WB` 선언부를 Decode 직후로 옮기면 됩니다.

```scala
  // ============================================================
  // EX / WB pipeline register
  // ============================================================

  val exWbValid        = Reg(Bool()) init(False)
  val exWbWriteEnable  = Reg(Bool()) init(False)
  val exWbWriteAddress = Reg(UInt(2 bits)) init(0)
  val exWbWriteData    = Reg(Bits(8 bits)) init(0)
```

그리고 기존 아래쪽의 **중복 선언은 제거**합니다.

이건 Scala의 순방향 참조 문제를 피하기 위한 정리입니다.

---

# 3. Simulation

이번에는 단순히 마지막 PC만 확인하지 않고 **pipeline의 WB 결과를 전부 scoreboard로 검증**하겠습니다.

```scala
import spinal.core.sim._

object PipelinedMiniCpuSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new PipelinedMiniCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        dut.io.run #= false

        dut.io.loadEnable  #= false
        dut.io.loadAddress #= 0
        dut.io.loadData    #= 0

        // ========================================================
        // Register initialization
        // ========================================================

        def loadRegister(address: Int, data: Int): Unit = {
          dut.io.loadEnable  #= true
          dut.io.loadAddress #= address
          dut.io.loadData    #= data

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

        // ========================================================
        // Start CPU
        // ========================================================

        dut.io.run #= true

        // --------------------------------------------------------
        // Pipeline needs several cycles to fill.
        //
        // Expected WB sequence:
        //
        // ADD R2,R0,R1 -> R2 = 0x30
        // SUB R1,R1,R0 -> R1 = 0x10
        // AND R3,R1,R1 -> R3 = 0x10
        // OR  R2,R2,R0 -> R2 = 0x30
        // --------------------------------------------------------

        var writeBackCount = 0

        while (writeBackCount < 4) {

          dut.clockDomain.waitSampling()
          sleep(1)

          if (dut.io.writeBackEnable.toBoolean) {

            val address = dut.io.writeBackAddress.toInt
            val data    = dut.io.writeBackData.toBigInt.toInt

            println(
              f"WB #${writeBackCount + 1}%d : " +
              f"R$address%d <= 0x$data%02X"
            )

            writeBackCount match {

              case 0 =>
                assert(address == 2)
                assert(data == 0x30)

              case 1 =>
                assert(address == 1)
                assert(data == 0x10)

              case 2 =>
                assert(address == 3)
                assert(data == 0x10)

              case 3 =>
                assert(address == 2)
                assert(data == 0x30)
            }

            writeBackCount += 1
          }
        }

        dut.io.run #= false

        println("4-stage pipelined Mini CPU test passed.")
      }
  }
}
```

---

# 4. 예상 Pipeline 동작

정상적으로 동작하면 대략 이런 흐름입니다.

```text
Cycle       IF       ID       EX       WB
------------------------------------------------
  0         ADD
  1         SUB      ADD
  2         AND      SUB      ADD
  3         OR       AND      SUB      ADD
  4                  OR       AND      SUB
  5                           OR       AND
  6                                    OR
```

Write-back은:

```text
Cycle 3   → R2 = 0x30
Cycle 4   → R1 = 0x10
Cycle 5   → R3 = 0x10
Cycle 6   → R2 = 0x30
```

가 됩니다.

그리고 중요한 hazard:

```text
SUB R1, R1, R0
      │
      └──────► R1 = 0x10
                    │
                    ▼
             AND R3, R1, R1
```

때문에 `AND`가 이전 `SUB`의 결과를 사용해야 합니다.

그래서:

```text
EX/WB
  │
  │ 0x10
  ▼
ID/EX
```

forwarding을 넣었습니다.

---

## 이번 단계의 목표

이번에는 **pipeline register 3개가 실제 CPU의 clock boundary로 동작하는지**, 그리고 **hazard가 있는 실제 명령어 프로그램이 올바른 write-back sequence를 만드는지**를 한 번에 검증합니다.

성공 로그의 핵심은:

```text
WB #1 : R2 <= 0x30
WB #2 : R1 <= 0x10
WB #3 : R3 <= 0x10
WB #4 : R2 <= 0x30

4-stage pipelined Mini CPU test passed.
```

입니다.

이 테스트에서 문제가 생기면 **로그 전체를 그대로 보여주세요.** 이번에는 특히 pipeline cycle과 forwarding 타이밍을 기준으로 잡겠습니다.

---


