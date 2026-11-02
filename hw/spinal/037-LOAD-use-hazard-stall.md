좋습니다. 이제 **Project 13-5 — Load-Use Hazard + Pipeline Stall**로 넘어가겠습니다.

다만 현재 우리가 만든 CPU에는 아직 `LOAD` 명령과 Data Memory가 없으므로, 이번에는 **실제 CPU에 Data Memory를 추가하고 `LOAD → ALU` hazard를 stall로 해결** 하는 방향으로 바로 구현하겠습니다.

### 13-5에서 만들 구조

기존:

```text
IF → [IF/ID] → ID → [ID/EX] → EX → [EX/WB] → WB
```

여기에 LOAD를 넣으면:

```text
                    ┌───────────────┐
                    │  Data Memory  │
                    └───────┬───────┘
                            LOAD
                              ↓
IF → [IF/ID] → ID → [ID/EX] → EX → [EX/WB] → WB
                    ↑
                 Stall
```

핵심 상황은 이것입니다.

```text
LOAD R1, [R0]
ADD  R2, R1, R3
```

`ADD`가 바로 다음 cycle에 들어오면 `R1`의 새 값이 아직 준비되지 않습니다.

따라서 Hazard Detection Unit이:

```text
ID instruction의 source
        ↓
EX instruction이 LOAD인가?
        ↓
같은 register를 사용하는가?
        ↓
      YES
        ↓
   STALL + BUBBLE
```

를 수행합니다.

### 이번 단계에서 검증할 것

최종적으로 다음을 만족시키겠습니다.

```text
Cycle N     : LOAD R1, [R0]
Cycle N+1   : ADD가 hazard 감지
              ↓
              PC freeze
              IF/ID freeze
              ID/EX = bubble
Cycle N+2   : LOAD 결과 사용 가능
Cycle N+3   : ADD 실행
```

즉 **Forwarding으로 해결 가능한 RAW hazard와 Stall이 필요한 Load-Use hazard의 차이**를 실제 CPU waveform으로 확인하게 됩니다.

그리고 이번에도 처음부터 simulation까지 포함해서 **SpinalHDL 1.15.0 기준으로 작성하고, `waitSampling()` 타이밍까지 검토한 뒤 진행**하겠습니다.

---

---

좋습니다. 그런데 방금 단계에서 **LOAD/Data Memory를 추가한다고 했으므로**, 바로 코드를 만들기 전에 CPU 구조를 한 번 정리하고 구현하겠습니다.

## Project 13-5 — Load-Use Hazard + Stall

이번에는 기존 4-stage CPU에 **LOAD** 하나만 추가합니다.

### 1. 명령어

기존:

```text
0000 ADD
0001 SUB
0010 AND
0011 OR
0100 BEQ
```

추가:

```text
0101 LOAD
```

LOAD 형식은 단순하게 하겠습니다.

```text
15       12 11    10 9     8 7          0
+-----------+--------+-------+------------+
|   opcode  |   rd   |  rs1  |   offset   |
+-----------+--------+-------+------------+
```

의미:

```text
LOAD Rd, [Rs1 + offset]
```

예:

```text
LOAD R1, [R0 + 0]
```

---

## 2. Data Memory

8-bit 데이터, 16개 word로 충분합니다.

```text
Address    Data
  0        0x00
  1        0x00
  2        0x55
  ...
```

이번 테스트에서는:

```text
R0 = 0x02

Memory[2] = 0x55

LOAD R1, [R0 + 0]
```

그러면:

```text
R1 = 0x55
```

가 됩니다.

---

## 3. 핵심 Hazard

프로그램을:

```text
0: LOAD R1, [R0 + 0]
1: ADD  R2, R1, R3
```

로 만듭니다.

초기:

```text
R0 = 0x02
R1 = 0x00
R3 = 0x10
```

Memory:

```text
M[2] = 0x55
```

정상 결과:

```text
R1 = 0x55

R2 = 0x55 + 0x10
   = 0x65
```

그런데 pipeline에서는:

```text
Cycle       IF        ID        EX       WB

N           LOAD
N+1         ADD       LOAD
N+2                   ADD       LOAD
```

이 되어 버립니다.

`ADD`가 EX에 들어가는 시점에는 `LOAD` 결과가 아직 없습니다.

따라서:

```text
LOAD R1
     ↓
     ↓  dependency
     ↓
ADD R1,...
```

에서는 **일반적인 EX forwarding만으로 해결할 수 없습니다.**

---

# 4. Hazard Detection

ID 단계에서 다음을 검사합니다.

```scala
val loadUseHazard =
  idExIsLoad &&
  idExWriteEnable &&
  (
    idExRd === decRs1 ||
    idExRd === decRs2
  )
```

그리고 hazard가 발생하면:

```text
PC       → HOLD
IF/ID    → HOLD
ID/EX    → BUBBLE
```

즉:

```text
          PC
          │
          │ HOLD
          ▼
       [IF/ID]
          │
          │ HOLD
          ▼
       Hazard
          │
          ▼
       [ID/EX]
          │
          └── BUBBLE
```

이것이 **stall**입니다.

---

# 5. 왜 ID/EX에 Bubble을 넣는가?

중요합니다.

다음 명령:

```text
ADD R2,R1,R3
```

을 ID에서 멈춰놓고,

현재 LOAD는 계속 pipeline을 진행시켜야 합니다.

따라서:

```text
Cycle N

ID/EX = LOAD
IF/ID = ADD
```

다음:

```text
Cycle N+1

ID/EX = BUBBLE
IF/ID = ADD
```

그리고:

```text
Cycle N+2

ID/EX = ADD
```

가 됩니다.

즉 한 cycle을 비워주는 것입니다.

```text
LOAD → BUBBLE → ADD
```

---

## 6. 이번 단계에서 확인할 waveform

이번 simulation에서는 단순히 최종값만 검사하지 않고 다음을 확인하는 것이 좋습니다.

```text
Cycle
       IF/ID       ID/EX       EX/WB

N      LOAD
N+1    ADD         LOAD
N+2    ADD         BUBBLE      LOAD
N+3                 ADD
```

그리고 최종:

```text
R1 = 0x55
R2 = 0x65
```

가 되어야 합니다.

또한 **stall이 정확히 1 cycle 발생했는지**를 별도의 `debugStall` 신호로 노출해서 검증하겠습니다.

---

### 이번 단계의 핵심

지금까지:

```text
RAW hazard
    ↓
Forwarding
```

이번에는:

```text
LOAD-USE hazard
    ↓
Hazard Detection
    ↓
Stall
    ↓
Bubble
```

입니다.

이걸 통과하면 현재 CPU는 상당히 실제 CPU에 가까워집니다.

```text
4-stage pipeline
├── RAW forwarding        ✅
├── Control hazard        ✅
├── Branch flush          ✅
└── Load-use stall        ← 이번 단계
```

다음 구현에서는 **`PipelinedLoadCpu.scala`와 `PipelinedLoadCpuSim.scala` 전체를 한 번에 작성하고**, 
특히 stall 시 `PC`와 `IF/ID`가 정말 freeze되는지까지 simulation으로 검증하면 됩니다.

---

---

그런데, cycle N 에서 LOAD 가 실행될 때, cycle N+1 에서 register 값이 준비되지 않는 이유는?

---

핵심은 **LOAD의 데이터가 어디서 나오느냐**입니다.

`ADD` 같은 ALU 연산과 `LOAD`는 결과가 만들어지는 위치가 다릅니다.

### 1. ADD는 EX에서 바로 결과가 만들어짐

예를 들어:

```text
ADD R2, R0, R1
```

pipeline:

```text
Cycle N
        ID/EX
          ↓
Cycle N+1
          EX
          ↓
      ALU result
```

ALU는 조합논리이므로 EX 동안:

```text
R0 + R1 → ALU → result
```

가 바로 계산됩니다.

그래서 다음 명령이 필요하면 **EX forwarding**으로 바로 가져올 수 있습니다.

```text
          ADD
           │
           ▼
        ALU result
           │
           ├──────► forwarding ───► 다음 명령 ALU
           │
           ▼
          WB
```

---

# 2. LOAD는 데이터가 Memory에서 나옴

```text
LOAD R1, [R0]
```

는:

```text
주소 계산
   ↓
Data Memory 접근
   ↓
Memory data
   ↓
R1
```

이라는 추가 단계가 필요합니다.

우리가 만들고 있는 CPU에서는 **동기식 Data Memory**를 사용할 것이므로 핵심이 여기 있습니다.

```text
Cycle N
    EX
    │
    ├── 주소 계산
    │
    └── Memory read 요청
             │
             ▼
        Data Memory
```

그런데 synchronous memory는 요청한 주소의 데이터를 **다음 clock edge에서 결과로 내놓습니다.**

즉:

```text
Cycle N
    주소 = 2
       │
       │ memory request
       ▼
    [Data Memory]
       │
       │
Cycle N+1
       ▼
    data = 0x55
```

따라서 `Cycle N+1`이 되어서야 LOAD 데이터가 준비됩니다.

---

# 3. 그런데 왜 ADD는 N+1에 실행하면 안 되는가?

다음 명령을 생각해 봅시다.

```text
0: LOAD R1, [R0]
1: ADD  R2, R1, R3
```

pipeline을 단순화하면:

```text
Cycle       LOAD             ADD
────────────────────────────────────
N           EX
N+1         MEM              EX
N+2         WB               ...
```

Cycle N의 LOAD가:

```text
Memory[2] → 0x55
```

를 요청합니다.

Cycle N+1에는 Data Memory가:

```text
0x55
```

를 내놓습니다.

**그런데 동시에 ADD는 EX에서 `R1`을 필요로 합니다.**

문제는:

```text
ADD의 EX
     │
     ├── R1이 필요
     │
     └── 그런데 LOAD data가 이제 막 Memory에서 나옴
```

입니다.

---

# 4. 더 정확하게 말하면

제가 앞에서 "`Cycle N+1에서 register 값이 준비되지 않는다`"라고 표현한 것은 조금 부정확했습니다.

정확히는:

> **Cycle N+1에는 LOAD의 memory data가 준비되기 시작하지만, 기존 4-stage 구조에서 ADD가 EX 단계에서 필요로 하는 시점과 맞지 않기 때문에 일반적인 EX forwarding만으로는 해결할 수 없다.**

입니다.

즉 **R1 register 자체가 반드시 아직 0이라는 뜻은 아닙니다.**

실제 문제는 **데이터의 생산 시점과 소비 시점**입니다.

---

# 5. 그래서 Stall이 필요하다

한 cycle 기다리면:

```text
Cycle       LOAD             ADD
────────────────────────────────────
N           EX
N+1         MEM              STALL
N+2         WB               EX
```

또는 구현에 따라:

```text
Cycle N
LOAD → Memory request

Cycle N+1
LOAD → memory data available
ADD  → bubble/stall

Cycle N+2
LOAD → WB
ADD  → EX
```

이렇게 됩니다.

그러면 ADD가 EX에 들어갈 때:

```text
R1 = 0x55
```

가 준비되어 있습니다.

---

## 6. Forwarding과 Stall의 차이

이 차이가 이번 단계에서 가장 중요합니다.

### ALU → ALU

```text
ADD R1, R0, R2
ADD R3, R1, R4
```

첫 번째 ADD 결과는 EX에서 즉시 생성됩니다.

```text
ALU result
    │
    └──► forwarding ──► 다음 ALU
```

따라서 **stall 불필요**.

---

### LOAD → ALU

```text
LOAD R1, [R0]
ADD  R3, R1, R4
```

LOAD 결과는 Memory에서 나옵니다.

```text
Memory
   │
   │ 1 cycle latency
   ▼
LOAD data
   │
   └──► R1
```

따라서 다음 명령이 너무 빨리 들어오면:

```text
LOAD data
     │
     └──── 아직 준비 안 됨
                 ↑
                 │
             ADD가 필요로 함
```

**stall이 필요합니다.**

---

### 한 줄로 정리

> **Forwarding은 "결과가 이미 계산됐는데 Register File에 아직 쓰이지 않은 경우"를 해결하고, Stall은 "결과 자체가 아직 생성되지 않은 경우"를 해결합니다.**

그리고 `LOAD → ALU`가 대표적인 **stall이 필요한 경우**입니다.

---

---

좋습니다. 이제 설명을 반복하지 않고 **실제로 `LOAD + Load-Use Stall` CPU를 구현하고 시뮬레이션**하겠습니다.

이번 구현의 핵심은 **동기식 Data Memory의 1-cycle latency**를 명시적으로 pipeline에 반영하는 것입니다.

## 13-5 구조

이번에는 pipeline을 다음처럼 잡겠습니다.

```text
IF → [IF/ID] → ID → [ID/EX] → EX → [EX/WB] → WB
                              │
                              ▼
                         Data Memory
```

LOAD의 경우:

```text
LOAD
  │
  ▼
EX : address 계산
  │
  ▼
MEM : Data Memory read
  │
  ▼
EX/WB
  │
  ▼
WB : Register File
```

그리고:

```text
LOAD R1,[R0]
ADD  R2,R1,R3
```

가 들어오면:

```text
Cycle N     LOAD : EX
Cycle N+1   LOAD : MEM
            ADD  : STALL/BUBBLE
Cycle N+2   LOAD : WB
            ADD  : EX
```

가 되도록 하겠습니다.

---

# 1. 전체 CPU

SpinalHDL 1.15.0 기준입니다.

```scala
import spinal.core._

class PipelinedLoadCpu extends Component {

  // ================================================================
  // Opcodes
  // ================================================================

  val OP_ADD  = U"4'h0"
  val OP_SUB  = U"4'h1"
  val OP_AND  = U"4'h2"
  val OP_OR   = U"4'h3"
  val OP_BEQ  = U"4'h4"
  val OP_LOAD = U"4'h5"

  val io = new Bundle {

    // --------------------------------------------------------------
    // CPU control
    // --------------------------------------------------------------

    val run = in Bool()

    // --------------------------------------------------------------
    // Register initialization
    // --------------------------------------------------------------

    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // --------------------------------------------------------------
    // Data memory initialization
    // --------------------------------------------------------------

    val memWriteEnable  = in Bool()
    val memWriteAddress = in UInt(4 bits)
    val memWriteData    = in Bits(8 bits)

    // --------------------------------------------------------------
    // Debug
    // --------------------------------------------------------------

    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)

    val debugR0 = out Bits(8 bits)
    val debugR1 = out Bits(8 bits)
    val debugR2 = out Bits(8 bits)
    val debugR3 = out Bits(8 bits)

    val debugStall = out Bool()
  }

  // ================================================================
  // Register File
  // ================================================================

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  // ================================================================
  // Data Memory
  //
  // 16 words x 8 bits
  // ================================================================

  val dataMem = Mem(Bits(8 bits), wordCount = 16)

  // ================================================================
  // Program Counter
  // ================================================================

  val pc = Reg(UInt(4 bits)) init(0)

  // ================================================================
  // Instruction ROM
  //
  // 0: LOAD R1, [R0 + 0]
  // 1: ADD  R2, R1, R3
  // 2+: NOP
  // ================================================================

  val rom = Mem(Bits(16 bits), wordCount = 16)

  rom.init(
    Seq(
      B"16'h5100", // 0: LOAD R1, [R0 + 0]
      B"16'h0880", // 1: ADD  R2, R1, R3
      B"16'h0000", // 2
      B"16'h0000", // 3
      B"16'h0000", // 4
      B"16'h0000", // 5
      B"16'h0000", // 6
      B"16'h0000", // 7
      B"16'h0000", // 8
      B"16'h0000", // 9
      B"16'h0000", // 10
      B"16'h0000", // 11
      B"16'h0000", // 12
      B"16'h0000", // 13
      B"16'h0000", // 14
      B"16'h0000"  // 15
    )
  )

  val fetchInstruction = rom.readAsync(pc)

  // ================================================================
  // IF / ID
  // ================================================================

  val ifIdValid       = Reg(Bool()) init(False)
  val ifIdInstruction = Reg(Bits(16 bits)) init(0)
  val ifIdPc          = Reg(UInt(4 bits)) init(0)

  // ================================================================
  // ID / EX
  // ================================================================

  val idExValid = Reg(Bool()) init(False)

  val idExPc = Reg(UInt(4 bits)) init(0)

  val idExRd = Reg(UInt(2 bits)) init(0)

  val idExA = Reg(Bits(8 bits)) init(0)
  val idExB = Reg(Bits(8 bits)) init(0)

  val idExOpcode = Reg(UInt(4 bits)) init(0)

  val idExWriteEnable = Reg(Bool()) init(False)

  val idExIsLoad = Reg(Bool()) init(False)

  val idExMemAddress = Reg(UInt(4 bits)) init(0)

  // ================================================================
  // EX / WB
  // ================================================================

  val exWbValid = Reg(Bool()) init(False)

  val exWbWriteEnable = Reg(Bool()) init(False)

  val exWbWriteAddress = Reg(UInt(2 bits)) init(0)

  val exWbWriteData = Reg(Bits(8 bits)) init(0)

  // ================================================================
  // Decode
  // ================================================================

  val decOpcode = ifIdInstruction(15 downto 12)

  val decRd =
    ifIdInstruction(11 downto 10).asUInt

  val decRs1 =
    ifIdInstruction(9 downto 8).asUInt

  val decRs2 =
    ifIdInstruction(7 downto 6).asUInt

  val decOffset =
    ifIdInstruction(7 downto 0).asUInt

  val decWriteEnable =
    (decOpcode === OP_ADD) ||
    (decOpcode === OP_SUB) ||
    (decOpcode === OP_AND) ||
    (decOpcode === OP_OR) ||
    (decOpcode === OP_LOAD)

  val decIsLoad =
    decOpcode === OP_LOAD

  // ================================================================
  // Register File read
  // ================================================================

  val regDataA = registers(decRs1)
  val regDataB = registers(decRs2)

  // ================================================================
  // EX result
  // ================================================================

  val exAluResult = Bits(8 bits)

  exAluResult := 0

  switch(idExOpcode) {

    is(OP_ADD) {
      exAluResult := idExA + idExB
    }

    is(OP_SUB) {
      exAluResult := idExA - idExB
    }

    is(OP_AND) {
      exAluResult := idExA & idExB
    }

    is(OP_OR) {
      exAluResult := idExA | idExB
    }
  }

  // ================================================================
  // LOAD address
  // ================================================================

  val loadAddress =
    idExA.asUInt + idExMemAddress

  // ================================================================
  // Load-use hazard detection
  //
  // Current instruction in ID depends on the destination register
  // of a LOAD currently in ID/EX.
  // ================================================================

  val loadUseHazard =
    idExValid &&
    idExIsLoad &&
    idExWriteEnable &&
    (
      (idExRd === decRs1) ||
      (idExRd === decRs2)
    )

  io.debugStall := loadUseHazard

  // ================================================================
  // Forwarding
  //
  // EX/WB -> ID
  // ================================================================

  val forwardedA = Bits(8 bits)
  val forwardedB = Bits(8 bits)

  forwardedA := regDataA
  forwardedB := regDataB

  when(
    exWbValid &&
    exWbWriteEnable &&
    (exWbWriteAddress === decRs1)
  ) {
    forwardedA := exWbWriteData
  }

  when(
    exWbValid &&
    exWbWriteEnable &&
    (exWbWriteAddress === decRs2)
  ) {
    forwardedB := exWbWriteData
  }

  // ================================================================
  // Sequential logic
  // ================================================================

  // ---------------------------------------------------------------
  // Register loading
  // ---------------------------------------------------------------

  when(io.loadEnable) {

    registers(io.loadAddress) := io.loadData

  } otherwise {

    // -------------------------------------------------------------
    // Data memory initialization
    // -------------------------------------------------------------

    when(io.memWriteEnable) {
      dataMem.write(
        io.memWriteAddress,
        io.memWriteData
      )
    }

    // -------------------------------------------------------------
    // Write Back
    // -------------------------------------------------------------

    when(
      exWbValid &&
      exWbWriteEnable
    ) {
      registers(exWbWriteAddress) := exWbWriteData
    }

    // -------------------------------------------------------------
    // CPU execution
    // -------------------------------------------------------------

    when(io.run) {

      // ===========================================================
      // Load-use hazard
      //
      // Keep PC and IF/ID unchanged.
      // Insert bubble into ID/EX.
      // ===========================================================

      when(loadUseHazard) {

        pc := pc

        ifIdValid := ifIdValid

        ifIdInstruction := ifIdInstruction

        ifIdPc := ifIdPc

        idExValid := False

        idExWriteEnable := False

        idExIsLoad := False

      } otherwise {

        // =========================================================
        // Normal pipeline advance
        // =========================================================

        pc := pc + 1

        // ---------------------------------------------------------
        // IF -> IF/ID
        // ---------------------------------------------------------

        ifIdValid       := True
        ifIdInstruction := fetchInstruction
        ifIdPc          := pc

        // ---------------------------------------------------------
        // ID -> ID/EX
        // ---------------------------------------------------------

        idExValid := ifIdValid

        idExPc := ifIdPc

        idExRd := decRd

        idExOpcode := decOpcode.asUInt

        idExWriteEnable := decWriteEnable

        idExIsLoad := decIsLoad

        idExMemAddress := decOffset(3 downto 0).asUInt

        idExA := forwardedA
        idExB := forwardedB
      }
    }
  }

  // ================================================================
  // EX / Memory / WB
  //
  // For this first version:
  //
  //   ALU instruction -> ALU result -> EX/WB
  //
  //   LOAD             -> Data Memory -> EX/WB
  //
  // ================================================================

  exWbValid := idExValid

  exWbWriteEnable :=
    idExValid &&
    idExWriteEnable

  exWbWriteAddress :=
    idExRd

  when(idExIsLoad) {

    exWbWriteData :=
      dataMem.readAsync(loadAddress)

  } otherwise {

    exWbWriteData :=
      exAluResult
  }

  // ================================================================
  // Debug outputs
  // ================================================================

  io.writeBackEnable :=
    exWbValid &&
    exWbWriteEnable

  io.writeBackAddress :=
    exWbWriteAddress

  io.writeBackData :=
    exWbWriteData

  io.debugR0 := registers(0)
  io.debugR1 := registers(1)
  io.debugR2 := registers(2)
  io.debugR3 := registers(3)
}
```

여기서 한 가지 중요한 점이 있습니다.

**위 코드의 Data Memory는 `readAsync()`입니다.**

즉 이 버전에서는 실제 동기 RAM latency를 재현하지 않습니다. 따라서 이것만으로는 우리가 방금 이야기한 **진짜 Load-Use Stall을 제대로 검증할 수 없습니다.**

따라서 이 코드를 그대로 실행시키는 것은 권하지 않습니다.

이번 단계에서 정확하게 하려면 **Data Memory를 `readSync()` 기반으로 만들고, LOAD의 결과를 별도의 MEM/WB 단계에 저장하는 구조로 한 단계 확장해야 합니다.**

즉 실제로는:

```text
IF → IF/ID → ID → ID/EX → EX → EX/MEM → MEM → MEM/WB → WB
```

가 되어야 합니다.

그리고 그때 진짜로:

```text
LOAD
 ↓
EX
 ↓
MEM
 ↓
MEM/WB
 ↓
WB
```

의 latency가 생깁니다.

따라서 **방금 코드는 실행하지 마십시오.**  
제가 처음에 4-stage 구조를 그대로 유지하면서 synchronous LOAD를 넣으려 한 것이 구조적으로 맞지 않았습니다.

다음 구현에서는 이 부분을 바로잡아 **`EX/MEM`, `MEM/WB`를 추가한 5-stage CPU**로 가는 것이 정확합니다.
