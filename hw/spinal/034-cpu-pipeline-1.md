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

---

"Execute" 스테이지 에서 정상적인 data path 와 forwarding data path 간의 충돌은 어떻게 해결하나?

---

좋은 질문입니다. 결론부터 말하면 **정상 data path와 forwarding data path가 서로 충돌하는 것이 아니라, MUX가 둘 중 하나를 선택하도록 설계**합니다.

그리고 제가 방금 제시한 코드의 forwarding은 정확히 말하면 **Execute stage 자체가 아니라 ID/EX에 들어가는 operand를 선택하는 forwarding**입니다.

### 1. 기본 구조

예를 들어 `ADD R3, R1, R2`가 Execute에 들어간다고 합시다.

정상 경로는:

```text
Register File
     │
     ├── R1 ──────┐
     │            │
     │         ┌──▼──┐
     │         │ MUX │────► ALU A
     │         └──▲──┘
     │            │
     │       Forwarding
     │            │
     ▼            │
  EX/WB ──────────┘
```

즉 **ALU 입력 앞에 MUX가 존재**합니다.

---

## 2. MUX가 선택한다

예를 들어 `rs1`의 정상 Register File 값이:

```text
R1 = 0x20
```

인데 바로 앞 instruction의 결과가:

```text
EX/WB = R1 <= 0x30
```

이라면 Register File에는 아직 `0x20`이 있을 수 있습니다.

이때:

```text
Normal path:
RegisterFile.R1 = 0x20

Forward path:
EX/WB.R1 = 0x30
```

두 값이 모두 ALU 쪽으로 **동시에 물리적으로 존재**할 수 있습니다.

하지만 ALU에 들어가는 값은 하나뿐입니다.

```text
                    ┌──────────────┐
RegisterFile ──────►│              │
                    │     MUX      ├────► ALU
Forwarding ────────►│              │
                    └──────▲───────┘
                           │
                       select
```

---

# 3. 선택 조건

개념적으로는:

```scala
if (forwardingNeeded) {
    aluInputA := forwardedData
} else {
    aluInputA := registerFileData
}
```

입니다.

제가 앞에서 작성한 코드의:

```scala
forwardedA := regFile.io.readDataA

when(
  exWbValid &&
  exWbWriteEnable &&
  exWbWriteAddress === decoder.io.rs1
) {
  forwardedA := exWbWriteData
}
```

가 바로 이 MUX를 표현합니다.

즉:

```text
forwarding 조건 = false
        │
        ▼
Register File ──► forwardedA ──► ALU

forwarding 조건 = true
        │
        ▼
EX/WB ──────────► forwardedA ──► ALU
```

입니다.

---

# 4. 그래서 "충돌"이 아니다

이 부분이 Pipeline 설계에서 굉장히 중요합니다.

### 잘못된 생각

```text
Register File ─────┐
                   ├──► ALU
Forwarding ────────┘

두 값이 동시에 ALU에 들어간다?
```

이렇게 하지 않습니다.

### 실제 구조

```text
Register File ─────┐
                   ▼
                  MUX ─────► ALU
                   ▲
Forwarding ───────┘
```

**MUX가 선택합니다.**

따라서 이것은 충돌 해결이라기보다 **operand selection**입니다.

---

# 5. 그런데 더 중요한 문제가 하나 있다

현재 우리가 만든 CPU에서는 forwarding을:

```text
EX/WB → ID/EX
```

로 하고 있습니다.

이것은 교육용으로 이해하기 쉽지만, 일반적인 pipeline에서는 forwarding을 좀 더 명확하게:

```text
             ┌──────────────┐
             │ RegisterFile │
             └──────┬───────┘
                    │
                    ▼
                 ID/EX
                    │
                    ▼
                  Execute
                 ┌─────┐
                 │ ALU │
                 └─────┘
                    ▲
                    │
             ┌──────┴──────┐
             │ Forward MUX │
             └──────▲──────┘
                    │
              EX/MEM, MEM/WB
```

처럼 **ALU 입력 직전에 forwarding MUX를 두는 방식**이 일반적입니다.

특히 실제 5-stage CPU에서는:

```text
IF → ID → EX → MEM → WB
```

이고 forwarding이:

```text
EX/MEM ──────┐
             ├──► ALU input MUX
MEM/WB ──────┤
             │
Register ────┘
```

처럼 됩니다.

---

# 6. 우리 CPU에서는 어떻게 해야 하나?

우리 CPU는 현재:

```text
IF
 ↓
IF/ID
 ↓
ID
 ↓
ID/EX
 ↓
EX
 ↓
EX/WB
 ↓
WB
```

입니다.

따라서 지금 구조에서는 **ID/EX register에 값을 저장하기 전에 forwarding을 선택**하고 있습니다.

```text
Register File ───────┐
                     │
                     ▼
                  Forward
                    MUX
                     │
                     ▼
                  ID/EX
                     │
                     ▼
                    ALU
```

이 방식도 동작할 수 있습니다.

하지만 여기에는 중요한 차이가 있습니다.

> **forwarding을 ID 단계에서 할 것인가, EX 단계에서 할 것인가**

입니다.

CPU 설계에서 일반적으로 더 깔끔한 방식은 **EX stage의 ALU 입력 앞에 forwarding MUX를 두는 것**입니다.

---

## 7. 다음 단계에서 우리가 할 개선

지금 만든 4-stage CPU를 한 단계 더 제대로 다듬는다면:

```text
IF
 │
 ▼
[IF/ID]
 │
 ▼
ID
 │
 ▼
[ID/EX]
 │
 ▼
┌─────────────────────┐
│ Forwarding MUX      │
│                     │
│ RegFile ───────┐    │
│ EX/WB ─────────┼──► ALU
│                │    │
└────────────────┘
```

로 만들고,

```text
ForwardA
ForwardB
```

라는 control signal을 만들어서

```text
00 = Register File
01 = EX/WB
10 = 다른 forwarding source
```

처럼 명시적으로 선택하도록 만들 수 있습니다.

**이게 실제 CPU의 forwarding unit 설계로 이어지는 핵심 개념입니다.**

그리고 지금 질문하신 **"정상 경로와 forwarding 경로가 충돌하면 어떻게 하지?"**에 대한 가장 중요한 한 문장은:

> **충돌시키지 않는다. ALU 입력 앞의 MUX와 forwarding control logic으로 정확히 하나의 source만 선택한다.**

입니다.

---

---

```
[info] [Progress] Start PipelinedMiniCpu test simulation with seed 725150158
[info] LOAD R0 <= 0x10
[info] LOAD R1 <= 0x20
[info] LOAD R2 <= 0x00
[info] LOAD R3 <= 0x00
[info] WB #1 : R2 <= 0x30
[info] WB #2 : R1 <= 0x10
[info] WB #3 : R3 <= 0x20
[info] [Error] Simulation failed at time=251
[error] Exception in thread "main" java.lang.AssertionError: assertion failed
```

---


네. **이 실패는 오히려 아주 좋은 결과입니다.** 지금 실제 pipeline hazard가 정확히 드러났습니다.

```text
WB #1 : R2 <= 0x30   ← ADD 정상
WB #2 : R1 <= 0x10   ← SUB 정상
WB #3 : R3 <= 0x20   ← ❌ AND가 이전 R1=0x20을 사용
```

즉, **제가 앞서 넣은 EX/WB → ID/EX forwarding만으로는 부족했습니다.**

---

# 1. 정확히 무슨 일이 발생했나?

명령어는:

```text
ADD R2, R0, R1
SUB R1, R1, R0
AND R3, R1, R1
OR  R2, R2, R0
```

입니다.

Pipeline을 보면:

```text
Cycle     IF       ID       EX       WB
------------------------------------------------
C0        ADD
C1        SUB      ADD
C2        AND      SUB      ADD
C3        OR       AND      SUB      ADD
C4                 OR       AND      SUB
C5                          OR       AND
C6                                   OR
```

문제는 **C3**입니다.

```text
C3

EX : SUB R1,R1,R0
     → 결과 0x10

ID : AND R3,R1,R1
     → R1이 필요함
```

그런데 우리의 forwarding은:

```text
EX/WB → ID
```

뿐입니다.

C3의 EX/WB에는 아직 **ADD의 결과 R2=0x30**이 들어 있습니다.

즉 AND가 보는 것은:

```text
Register File R1 = 0x20   ← 아직 SUB 결과가 WB되지 않음
EX/WB             = R2=0x30
```

그래서 AND는:

```text
0x20 & 0x20 = 0x20
```

을 계산한 것입니다.

정확히 지금 로그가 그걸 증명합니다.

---

# 2. 해결 방법

현재 4-stage 구조에서는 **두 종류의 forwarding**이 필요합니다.

```text
                         ┌──────────────┐
Register File ──────────►│              │
                         │ Forward MUX  ├──► ID/EX
EX result ──────────────►│              │
                         │              │
EX/WB result ───────────►│              │
                         └──────────────┘
```

즉:

```text
정상 Register File
        │
        │
        ▼
      MUX ───────► ID/EX
        ▲
        │
   ┌────┴────┐
   │         │
EX result   EX/WB result
```

### 우선순위

가장 최신 결과가 우선입니다.

```text
1. 현재 EX stage 결과
2. EX/WB 결과
3. Register File
```

왜냐하면 현재 EX의 결과가 EX/WB보다 **한 instruction 더 최신**이기 때문입니다.

---

# 3. 우리 CPU에 필요한 수정

기존의:

```scala
when(
  exWbValid &&
  exWbWriteEnable &&
  exWbWriteAddress === decoder.io.rs1
) {
  forwardedA := exWbWriteData
}
```

에 **현재 EX 결과 forwarding**을 추가해야 합니다.

핵심은:

```scala
when(
  idExValid &&
  idExRd === decoder.io.rs1
) {
  forwardedA := alu.io.result.asBits
} elsewhen(
  exWbValid &&
  exWbWriteEnable &&
  exWbWriteAddress === decoder.io.rs1
) {
  forwardedA := exWbWriteData
}
```

그리고 B도 동일합니다.

```scala
when(
  idExValid &&
  idExRd === decoder.io.rs2
) {
  forwardedB := alu.io.result.asBits
} elsewhen(
  exWbValid &&
  exWbWriteEnable &&
  exWbWriteAddress === decoder.io.rs2
) {
  forwardedB := exWbWriteData
}
```

---

# 4. 그런데 이걸 그대로 넣으면 중요한 문제가 하나 있습니다

현재 코드 구조에서는 `alu.io.result`가:

```text
ID/EX register
       ↓
      ALU
       ↓
alu.result
       ↓
forwarding MUX
       ↓
ID/EX register
```

가 됩니다.

처음 보면 **feedback loop처럼 보이지만 실제로는 combinational loop가 아닙니다.**

왜냐하면 중간에:

```text
ID/EX register
```

라는 clocked element가 있기 때문입니다.

실제 경로는:

```text
       ┌───────────────┐
       │               │
       ▼               │
    ID/EX Reg          │
       │               │
       ▼               │
      ALU              │
       │               │
       ▼               │
 Forward MUX ──────────┘
       │
       ▼
  next ID/EX Reg value
```

입니다.

**현재 ID/EX 값 → ALU → forwarding → 다음 clock의 ID/EX 값**

이므로 정상적인 synchronous feedback 구조입니다.

---

# 5. 더 정확한 Forwarding 우선순위

따라서 우리가 만들 forwarding logic은 개념적으로:

```text
if (EX stage가 같은 register를 생성)
    use EX result
else if (EX/WB가 같은 register를 생성)
    use EX/WB result
else
    use Register File
```

입니다.

이를 표로 보면:

| Source | 우선순위 | 이유 |
|---|---:|---|
| EX result | 1 | 가장 최신 결과 |
| EX/WB | 2 | 그 다음 최신 결과 |
| Register File | 3 | 이미 commit된 값 |

---

# 6. 이번 실패는 정확히 이것을 검증했다

현재 로그:

```text
R2 <= 0x30
R1 <= 0x10
R3 <= 0x20
```

은 CPU가 완전히 망가진 것이 아닙니다.

오히려:

```text
ADD → 정상
SUB → 정상
AND → RAW hazard 발생
```

이라는 것을 정확하게 보여줍니다.

즉 우리가 지금 처음으로 **실제 CPU pipeline hazard를 simulation으로 발견한 것**입니다.

---

# 7. 다음 수정 후 기대값

Forwarding을 제대로 넣으면:

```text
WB #1 : R2 <= 0x30
WB #2 : R1 <= 0x10
WB #3 : R3 <= 0x10
WB #4 : R2 <= 0x30

4-stage pipelined Mini CPU test passed.
```

가 되어야 합니다.

그리고 이 구조에서 중요한 forwarding은:

```text
SUB R1,...
     │
     │ EX result = 0x10
     ▼
┌─────────────┐
│ Forward MUX │
└──────┬──────┘
       │
       ▼
AND R3,R1,R1
```

입니다.

**즉 이번 실패의 원인은 `EX/WB forwarding`만 있고 `EX → ID/EX forwarding`이 없었던 것입니다.**

이 수정은 단순한 테스트 수정이 아니라, 우리가 방금 질문했던 **"정상 datapath와 forwarding datapath를 어떻게 충돌 없이 선택하는가?"**에 대한 실제 CPU 구현입니다.

---

---

완벽합니다. ✅ **4-stage pipelined Mini CPU + RAW forwarding까지 통과**했습니다.

이번 테스트는 단순 pipeline 동작보다 훨씬 중요한 것을 검증했습니다.

```text
WB #1 : R2 <= 0x30
WB #2 : R1 <= 0x10
WB #3 : R3 <= 0x10
WB #4 : R2 <= 0x30
```

즉,

```text
ADD R2,R0,R1  → 0x30
SUB R1,R1,R0  → 0x10
AND R3,R1,R1  → 0x10
OR  R2,R2,R0  → 0x30
```

가 pipeline을 통과하면서 **RAW dependency까지 올바르게 해결**됐습니다.

### 현재 CPU

```text
          ┌───────┐
          │ Fetch │
          └───┬───┘
              │
          [ IF / ID ]
              │
              ▼
          ┌───────┐
          │Decode │
          └───┬───┘
              │
          [ ID / EX ]
              │
              ▼
          ┌─────────┐
          │ Execute │◄──── Forwarding
          └────┬────┘
               │
           [ EX / WB ]
               │
               ▼
           Write Back
```

그리고 forwarding은 개념적으로:

```text
Register File ───────┐
                     │
EX result ───────────┼──► Forward MUX ─► ALU operand
                     │
EX/WB result ────────┘
```

까지 구현됐습니다.

**Project 13의 핵심 기반은 이제 제대로 만들어졌습니다.**

다음 단계에서는 단순 ALU dependency를 넘어 **branch/control hazard**를 다루겠습니다. 여기서 처음으로 `PC`와 pipeline에 대한 문제가 생기므로, CPU가 실제 프로세서처럼 한 단계 더 발전합니다.
