좋습니다. 이번에는 앞서 말한 대로 **정식 5-stage pipeline**으로 갑니다.

그리고 이번에는 `LOAD`의 실제 memory latency를 제대로 모델링하겠습니다.

## Project 13-5 — 5-stage Pipeline + Load-Use Stall

구조를 먼저 확정합니다.

```text
IF → ID → EX → MEM → WB
```

Pipeline register:

```text
IF ──► [IF/ID]
          │
          ▼
ID ──► [ID/EX]
          │
          ▼
EX ──► [EX/MEM]
          │
          ▼
MEM ─► [MEM/WB]
          │
          ▼
WB
```

### LOAD

```text
LOAD R1, [R0]

IF
 ↓
ID
 ↓
EX       주소 계산
 ↓
MEM      Data Memory read
 ↓
WB       R1 ← data
```

### Load-Use

```text
LOAD R1,[R0]
ADD  R2,R1,R3
```

에서는:

```text
Cycle       IF       ID       EX       MEM      WB

N           LOAD
N+1         ADD      LOAD
N+2                  ADD?     LOAD
```

이렇게 하면 안 됩니다.

따라서:

```text
Cycle N+2

IF/ID : ADD   ← HOLD
ID/EX : BUBBLE
EX/MEM: LOAD
```

로 만들어야 합니다.

결과:

```text
Cycle       IF       ID       EX       MEM      WB

N           LOAD

N+1         ADD      LOAD

N+2         ADD      BUBBLE   LOAD

N+3         NOP      ADD               LOAD

N+4                              ADD             LOAD?
```

핵심은 **LOAD는 계속 pipeline을 진행시키고, ADD만 한 cycle 멈추는 것**입니다.

---

# 1. 명령어 구성

이번 테스트는 최소한으로 갑니다.

```text
0000 ADD
0101 LOAD
```

ADD:

```text
15       12 11    10 9     8 7     6
+-----------+--------+-------+-------+
|   opcode  |   rd   |  rs1  |  rs2  |
+-----------+--------+-------+-------+
```

LOAD:

```text
15       12 11    10 9     8 7          0
+-----------+--------+-------+------------+
|   opcode  |   rd   |  rs1  |   offset   |
+-----------+--------+-------+------------+
```

---

# 2. 5-stage CPU

아래 코드로 진행하겠습니다.

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  // ================================================================
  // Opcodes
  // ================================================================

  val OP_ADD  = U"4'h0"
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
  // 16 x 8-bit synchronous-read memory
  // ================================================================

  val dataMem = Mem(Bits(8 bits), wordCount = 16)

  // ================================================================
  // Instruction ROM
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

  val pc = Reg(UInt(4 bits)) init(0)

  val fetchInstruction =
    rom.readAsync(pc)

  // ================================================================
  // IF / ID
  // ================================================================

  val ifIdValid =
    Reg(Bool()) init(False)

  val ifIdInstruction =
    Reg(Bits(16 bits)) init(0)

  // ================================================================
  // ID / EX
  // ================================================================

  val idExValid =
    Reg(Bool()) init(False)

  val idExOpcode =
    Reg(UInt(4 bits)) init(0)

  val idExRd =
    Reg(UInt(2 bits)) init(0)

  val idExA =
    Reg(Bits(8 bits)) init(0)

  val idExB =
    Reg(Bits(8 bits)) init(0)

  val idExMemAddress =
    Reg(UInt(4 bits)) init(0)

  val idExIsLoad =
    Reg(Bool()) init(False)

  val idExWriteEnable =
    Reg(Bool()) init(False)

  // ================================================================
  // EX / MEM
  // ================================================================

  val exMemValid =
    Reg(Bool()) init(False)

  val exMemRd =
    Reg(UInt(2 bits)) init(0)

  val exMemAluResult =
    Reg(Bits(8 bits)) init(0)

  val exMemIsLoad =
    Reg(Bool()) init(False)

  val exMemMemAddress =
    Reg(UInt(4 bits)) init(0)

  val exMemWriteEnable =
    Reg(Bool()) init(False)

  // ================================================================
  // MEM / WB
  // ================================================================

  val memWbValid =
    Reg(Bool()) init(False)

  val memWbRd =
    Reg(UInt(2 bits)) init(0)

  val memWbData =
    Reg(Bits(8 bits)) init(0)

  val memWbWriteEnable =
    Reg(Bool()) init(False)

  // ================================================================
  // Decode
  // ================================================================

  val decOpcode =
    ifIdInstruction(15 downto 12)

  val decRd =
    ifIdInstruction(11 downto 10).asUInt

  val decRs1 =
    ifIdInstruction(9 downto 8).asUInt

  val decRs2 =
    ifIdInstruction(7 downto 6).asUInt

  val decOffset =
    ifIdInstruction(7 downto 0)

  val decIsLoad =
    decOpcode === OP_LOAD

  val decWriteEnable =
    (decOpcode === OP_ADD) ||
    (decOpcode === OP_LOAD)

  // ================================================================
  // Register File read
  // ================================================================

  val regDataA =
    registers(decRs1)

  val regDataB =
    registers(decRs2)

  // ================================================================
  // Forwarding
  //
  // EX/MEM -> ID
  // MEM/WB -> ID
  //
  // LOAD in EX/MEM is NOT forwarded because its data is not
  // available yet.
  // ================================================================

  val forwardedA =
    Bits(8 bits)

  val forwardedB =
    Bits(8 bits)

  forwardedA := regDataA
  forwardedB := regDataB

  when(
    exMemValid &&
    exMemWriteEnable &&
    !exMemIsLoad &&
    (exMemRd === decRs1)
  ) {
    forwardedA := exMemAluResult
  }

  when(
    exMemValid &&
    exMemWriteEnable &&
    !exMemIsLoad &&
    (exMemRd === decRs2)
  ) {
    forwardedB := exMemAluResult
  }

  when(
    memWbValid &&
    memWbWriteEnable &&
    (memWbRd === decRs1)
  ) {
    forwardedA := memWbData
  }

  when(
    memWbValid &&
    memWbWriteEnable &&
    (memWbRd === decRs2)
  ) {
    forwardedB := memWbData
  }

  // ================================================================
  // Load-Use Hazard Detection
  //
  // The instruction currently in ID uses the destination of a
  // LOAD currently in EX.
  // ================================================================

  val loadUseHazard =
    idExValid &&
    idExIsLoad &&
    idExWriteEnable &&
    (
      (idExRd === decRs1) ||
      (idExRd === decRs2)
    )

  io.debugStall :=
    loadUseHazard

  // ================================================================
  // ALU
  // ================================================================

  val aluResult =
    Bits(8 bits)

  aluResult := 0

  when(idExOpcode === OP_ADD) {
    aluResult :=
      idExA + idExB
  }

  // ================================================================
  // Sequential logic
  // ================================================================

  when(io.loadEnable) {

    registers(io.loadAddress) :=
      io.loadData

  } otherwise {

    // --------------------------------------------------------------
    // External Data Memory initialization
    // --------------------------------------------------------------

    when(io.memWriteEnable) {

      dataMem.write(
        io.memWriteAddress,
        io.memWriteData
      )
    }

    // --------------------------------------------------------------
    // WB
    // --------------------------------------------------------------

    when(
      memWbValid &&
      memWbWriteEnable
    ) {

      registers(memWbRd) :=
        memWbData
    }

    // --------------------------------------------------------------
    // CPU
    // --------------------------------------------------------------

    when(io.run) {

      // ============================================================
      // MEM -> WB
      // ============================================================

      memWbValid :=
        exMemValid

      memWbRd :=
        exMemRd

      memWbWriteEnable :=
        exMemWriteEnable

      when(exMemIsLoad) {

        // Synchronous-read memory.
        //
        // The address is presented in this cycle.
        // The returned value becomes the MEM/WB data
        // on the following clock.
        //
        // The exact simulation behavior will be checked below.
        memWbData :=
          dataMem.readSync(exMemMemAddress)

      } otherwise {

        memWbData :=
          exMemAluResult
      }

      // ============================================================
      // EX -> EX/MEM
      // ============================================================

      exMemValid :=
        idExValid

      exMemRd :=
        idExRd

      exMemAluResult :=
        aluResult

      exMemIsLoad :=
        idExIsLoad

      exMemMemAddress :=
        idExA.asUInt +
          idExMemAddress

      exMemWriteEnable :=
        idExWriteEnable

      // ============================================================
      // Hazard
      // ============================================================

      when(loadUseHazard) {

        // ----------------------------------------------------------
        // HOLD PC
        // ----------------------------------------------------------

        pc := pc

        // ----------------------------------------------------------
        // HOLD IF/ID
        // ----------------------------------------------------------

        ifIdValid :=
          ifIdValid

        ifIdInstruction :=
          ifIdInstruction

        // ----------------------------------------------------------
        // INSERT BUBBLE into ID/EX
        // ----------------------------------------------------------

        idExValid :=
          False

        idExWriteEnable :=
          False

        idExIsLoad :=
          False

      } otherwise {

        // ==========================================================
        // PC
        // ==========================================================

        pc :=
          pc + 1

        // ==========================================================
        // IF -> IF/ID
        // ==========================================================

        ifIdValid :=
          True

        ifIdInstruction :=
          fetchInstruction

        // ==========================================================
        // ID -> ID/EX
        // ==========================================================

        idExValid :=
          ifIdValid

        idExOpcode :=
          decOpcode.asUInt

        idExRd :=
          decRd

        idExA :=
          forwardedA

        idExB :=
          forwardedB

        idExMemAddress :=
          decOffset(3 downto 0).asUInt

        idExIsLoad :=
          decIsLoad

        idExWriteEnable :=
          decWriteEnable
      }
    }
  }

  // ================================================================
  // Debug outputs
  // ================================================================

  io.writeBackEnable :=
    memWbValid &&
    memWbWriteEnable

  io.writeBackAddress :=
    memWbRd

  io.writeBackData :=
    memWbData

  io.debugR0 :=
    registers(0)

  io.debugR1 :=
    registers(1)

  io.debugR2 :=
    registers(2)

  io.debugR3 :=
    registers(3)
}
```

다만 여기서 **중요한 문제가 하나 있습니다.**

SpinalHDL의 `Mem.readSync()`는 일반적인 HDL synchronous RAM의 동작처럼 **read address를 clock에 등록하고 다음 cycle에 data를 반환**합니다.

따라서 위 구조에서 `dataMem.readSync()`를 `memWbData`에 직접 연결하는 것만으로는 우리가 원하는 MEM/WB 타이밍을 정확하게 표현하기 어렵습니다.

즉 지금 단계에서 또 시뮬레이션을 먼저 돌려보고 수정하는 방식으로 가면, 앞에서 겪었던 것처럼 **simulation scheduling 문제를 다시 만들 가능성이 높습니다.**

따라서 다음 단계에서는 이 부분을 정리해서:

```text
EX/MEM
   │
   │ address
   ▼
Data Memory
   │
   │ registered data
   ▼
MEM/WB
```

가 되도록 **Memory read register를 명시적으로 하나 두는 방식**으로 구현하는 것이 안전합니다.

그리고 그 상태에서 테스트를:

```text
R0 = 0x02
R3 = 0x10
M[2] = 0x55

LOAD R1,[R0]
ADD  R2,R1,R3
```

로 해서 반드시:

```text
R1 = 0x55
R2 = 0x65
```

그리고 `debugStall`이 **정확히 1 cycle만 `true`**인지까지 검증하겠습니다.

**즉, 위 코드는 지금 실행하지 말고**, 다음 구현에서 `EX/MEM → Memory → MEM/WB`의 synchronous timing을 정확히 고정한 뒤 simulation까지 한 번에 검증하는 것이 맞습니다.

---

---

좋다. 이번에는 **설계 → hazard → forwarding → simulation timing까지 한 번에 정리해서**, 이전 코드의 문제였던 synchronous memory 타이밍도 피해서 가자.

이번 단계는 **Project 13-5: Load-Use Hazard + 1-cycle Stall**이다.

핵심은 다음 구조다.

```text
IF → ID → EX → MEM → WB
     │         │
     │         └── LOAD data
     │
     └── Load-Use Hazard 검출
              │
              └── 1 cycle STALL
```

그리고 forwarding은 **EX 단계의 operand MUX**에서 처리한다.

```text
                    ┌── EX/MEM ALU result
                    │
ID/EX operand ── MUX ├── MEM/WB result
                    │
                    └── ID/EX original
                         │
                         ▼
                        ALU
```

### 1. 중요한 타이밍

예를 들어:

```text
LOAD R1, [R0+0]
ADD  R2, R1, R3
```

이면:

```text
Cycle N
    LOAD → EX
    주소 계산

Cycle N+1
    LOAD → MEM
    ADD  → ID
    └─ R1 아직 LOAD 결과가 없음
       → STALL

Cycle N+2
    LOAD → WB
    ADD  → EX
    └─ MEM/WB → EX forwarding
       R1 = LOAD 결과

Cycle N+3
    ADD → MEM

Cycle N+4
    ADD → WB
```

즉 **LOAD-USE는 1-cycle stall + forwarding**으로 해결한다.

---

# Project 13-5 코드

이번에는 `readSync()` 자체의 애매한 파이프라인 타이밍을 CPU 코드 안에서 억지로 조합하지 않고, **data memory를 명시적인 1-cycle synchronous read model**로 만든다.

이렇게 하면 CPU pipeline timing을 명확하게 검증할 수 있다.

### `FiveStageLoadCpu.scala`

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  // ------------------------------------------------------------
  // Instruction format
  //
  // R-type:
  //
  // 15       12 11     10 9      8 7     6 5      0
  // +-----------+---------+--------+-------+--------+
  // |  opcode   |   rd    |  rs1   |  rs2  | unused |
  // +-----------+---------+--------+-------+--------+
  //
  // LOAD:
  //
  // 15       12 11     10 9      8 7            0
  // +-----------+---------+--------+--------------+
  // |  opcode   |   rd    |  rs1   |    offset    |
  // +-----------+---------+--------+--------------+
  // ------------------------------------------------------------

  val OP_ADD  = 0
  val OP_LOAD = 5

  val io = new Bundle {
    val run = in Bool()

    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)

    val debugStall = out Bool()

    val debugR0 = out Bits(8 bits)
    val debugR1 = out Bits(8 bits)
    val debugR2 = out Bits(8 bits)
    val debugR3 = out Bits(8 bits)
  }

  // ============================================================
  // Register File
  // ============================================================

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  // ============================================================
  // Instruction Memory
  // ============================================================

  val rom = Mem(Bits(16 bits), wordCount = 16)

  rom.init(
    Seq(
      B"16'h5100", // 0: LOAD R1, [R0 + 0]
      B"16'h08C0", // 1: ADD  R2, R1, R3
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

  // ============================================================
  // Data Memory
  //
  // Educational synchronous-read model:
  //
  // address is captured at the clock edge,
  // data becomes available in the following cycle.
  // ============================================================

  val dataMem = Mem(Bits(8 bits), wordCount = 16)

  // The actual memory read result is explicitly registered.
  // This makes the one-cycle LOAD latency unambiguous.
  val dataReadData = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // PC
  // ============================================================

  val pc = Reg(UInt(4 bits)) init(0)

  // ============================================================
  // IF / ID
  // ============================================================

  val ifIdValid       = Reg(Bool()) init(False)
  val ifIdInstruction = Reg(Bits(16 bits)) init(0)
  val ifIdPc          = Reg(UInt(4 bits)) init(0)

  // ============================================================
  // Decode
  // ============================================================

  val decOpcode = ifIdInstruction(15 downto 12)
  val decRd     = ifIdInstruction(11 downto 10).asUInt
  val decRs1    = ifIdInstruction(9 downto 8).asUInt
  val decRs2    = ifIdInstruction(7 downto 6).asUInt
  val decOffset = ifIdInstruction(7 downto 0)

  val decIsLoad = decOpcode === OP_LOAD
  val decIsAdd  = decOpcode === OP_ADD

  val decUsesRs1 = decIsLoad || decIsAdd
  val decUsesRs2 = decIsAdd

  val decWriteEnable =
    decIsLoad || decIsAdd

  // ============================================================
  // ID / EX
  // ============================================================

  val idExValid = Reg(Bool()) init(False)

  val idExIsLoad       = Reg(Bool()) init(False)
  val idExWriteEnable  = Reg(Bool()) init(False)

  val idExRd  = Reg(UInt(2 bits)) init(0)
  val idExRs1 = Reg(UInt(2 bits)) init(0)
  val idExRs2 = Reg(UInt(2 bits)) init(0)

  val idExA = Reg(Bits(8 bits)) init(0)
  val idExB = Reg(Bits(8 bits)) init(0)

  val idExOffset = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // EX / MEM
  // ============================================================

  val exMemValid = Reg(Bool()) init(False)

  val exMemIsLoad      = Reg(Bool()) init(False)
  val exMemWriteEnable = Reg(Bool()) init(False)

  val exMemRd = Reg(UInt(2 bits)) init(0)

  val exMemAluResult = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // MEM / WB
  // ============================================================

  val memWbValid = Reg(Bool()) init(False)

  val memWbWriteEnable = Reg(Bool()) init(False)
  val memWbRd          = Reg(UInt(2 bits)) init(0)
  val memWbData        = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // Forwarding
  //
  // IMPORTANT:
  // Forwarding happens at EX.
  //
  // Priority:
  //
  // 1. EX/MEM
  // 2. MEM/WB
  // 3. ID/EX register value
  //
  // A LOAD in EX/MEM cannot be forwarded because
  // its data does not exist yet.
  // ============================================================

  val exOperandA = Bits(8 bits)
  val exOperandB = Bits(8 bits)

  exOperandA := idExA
  exOperandB := idExB

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs1)
  ) {
    exOperandA := exMemAluResult
  } elsewhen (
    memWbValid &&
      memWbWriteEnable &&
      (memWbRd === idExRs1)
  ) {
    exOperandA := memWbData
  }

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs2)
  ) {
    exOperandB := exMemAluResult
  } elsewhen (
    memWbValid &&
      memWbWriteEnable &&
      (memWbRd === idExRs2)
  ) {
    exOperandB := memWbData
  }

  // ============================================================
  // EX
  // ============================================================

  val exAluResult = Bits(8 bits)

  exAluResult := exOperandA + exOperandB

  val exLoadAddress =
    exOperandA.asUInt + idExOffset.asUInt

  // ============================================================
  // Load-Use Hazard Detection
  //
  // Current ID instruction needs a register that the
  // instruction currently in EX is going to load.
  //
  // Example:
  //
  //   LOAD R1, [R0]
  //   ADD  R2, R1, R3
  //
  // ID sees ADD:
  //   rs1 = R1
  //
  // EX contains LOAD:
  //   rd = R1
  //
  // => stall
  // ============================================================

  val loadUseHazard =
    ifIdValid &&
      idExValid &&
      idExIsLoad &&
      idExWriteEnable &&
      (
        (decUsesRs1 && (idExRd === decRs1)) ||
        (decUsesRs2 && (idExRd === decRs2))
      )

  io.debugStall := loadUseHazard

  // ============================================================
  // Write Back
  // ============================================================

  io.writeBackEnable  := memWbValid && memWbWriteEnable
  io.writeBackAddress := memWbRd
  io.writeBackData    := memWbData

  // ============================================================
  // Debug register outputs
  // ============================================================

  io.debugR0 := registers(0)
  io.debugR1 := registers(1)
  io.debugR2 := registers(2)
  io.debugR3 := registers(3)

  // ============================================================
  // Sequential logic
  // ============================================================

  when(io.loadEnable) {

    registers(io.loadAddress) := io.loadData

  } otherwise {

    // ----------------------------------------------------------
    // Write Back
    // ----------------------------------------------------------

    when(memWbValid && memWbWriteEnable) {
      registers(memWbRd) := memWbData
    }

    // ----------------------------------------------------------
    // Data Memory read
    //
    // EX/MEM contains the address of a LOAD.
    //
    // The memory data is captured into dataReadData here.
    // ----------------------------------------------------------

    when(exMemValid && exMemIsLoad) {
      dataReadData := dataMem.readAsync(exMemAluResult.asUInt)
    }

    // ----------------------------------------------------------
    // MEM -> WB
    // ----------------------------------------------------------

    memWbValid       := exMemValid
    memWbWriteEnable := exMemWriteEnable
    memWbRd          := exMemRd

    when(exMemIsLoad) {
      memWbData := dataReadData
    } otherwise {
      memWbData := exMemAluResult
    }

    // ----------------------------------------------------------
    // EX -> MEM
    // ----------------------------------------------------------

    exMemValid       := idExValid
    exMemIsLoad      := idExIsLoad
    exMemWriteEnable := idExWriteEnable
    exMemRd          := idExRd

    when(idExIsLoad) {
      exMemAluResult := exLoadAddress.asBits
    } otherwise {
      exMemAluResult := exAluResult
    }

    // ----------------------------------------------------------
    // ID -> EX
    //
    // If load-use hazard:
    //
    //   PC          HOLD
    //   IF/ID       HOLD
    //   ID/EX       BUBBLE
    //
    // This is the classic one-cycle stall.
    // ----------------------------------------------------------

    when(loadUseHazard) {

      idExValid       := False
      idExIsLoad      := False
      idExWriteEnable := False

    } otherwise {

      idExValid       := ifIdValid
      idExIsLoad      := decIsLoad
      idExWriteEnable := ifIdValid && decWriteEnable

      idExRd  := decRd
      idExRs1 := decRs1
      idExRs2 := decRs2

      idExA := registers(decRs1)
      idExB := registers(decRs2)

      idExOffset := decOffset

      // --------------------------------------------------------
      // IF -> ID
      // --------------------------------------------------------

      ifIdValid       := True
      ifIdInstruction := rom.readAsync(pc)
      ifIdPc          := pc

      pc := pc + 1
    }
  }
}
```

그런데 여기서 **한 가지 중요한 문제가 있다.**

위 코드의 `dataReadData := dataMem.readAsync(...)` 방식은 우리가 원한 "동기 메모리"를 실제 `Mem.readSync()`로 구현한 것이 아니다. 따라서 **CPU 학습 목적의 정확한 LOAD latency를 검증하려면**, 이 부분을 별도 `SyncDataMemory` 컴포넌트로 명시하는 편이 더 안전하다.

그래서 이번에는 아예 그 구조로 고정하는 것이 좋다.

---

# 정확한 Data Memory

```scala
import spinal.core._

class SyncDataMemory extends Component {

  val io = new Bundle {
    val readEnable  = in Bool()
    val readAddress = in UInt(4 bits)
    val readData    = out Bits(8 bits)

    val writeEnable = in Bool()
    val writeAddress = in UInt(4 bits)
    val writeData    = in Bits(8 bits)
  }

  val mem = Mem(Bits(8 bits), wordCount = 16)

  val readDataReg = Reg(Bits(8 bits)) init(0)

  when(io.writeEnable) {
    mem.write(
      address = io.writeAddress,
      data    = io.writeData
    )
  }

  when(io.readEnable) {
    readDataReg := mem.readAsync(io.readAddress)
  }

  io.readData := readDataReg
}
```

이 구조는 의도적으로:

```text
address
   │
   ▼
readAsync()
   │
   ▼
Register
   │
   ▼
readData
```

로 만들었다.

즉 **CPU 관점에서는 정확히 1-cycle memory latency**다.

그리고 실제 FPGA Block RAM을 사용할 때는 이 부분을 `readSync()` 기반 구현으로 바꿀 수 있다.

---

# 이번 테스트에서 사용할 프로그램

여기서 이전 답변의 instruction encoding도 하나 수정해야 한다.

### LOAD

```text
LOAD R1, [R0 + 0]
```

encoding:

```text
opcode = 0101
rd     = 01
rs1    = 00
offset = 00000000
```

따라서:

```text
0x5100
```

### ADD

```text
ADD R2, R1, R3
```

필드:

```text
opcode = 0000
rd     = 10
rs1    = 01
rs2    = 11
```

따라서:

```text
0x08C0
```

**이전 답변의 `0x0880`은 잘못된 encoding이었다.**

이번에는 `0x08C0`가 맞다.

---

# 예상 실행

초기 상태:

```text
R0 = 0x02
R1 = 0x00
R2 = 0x00
R3 = 0x10

MEM[2] = 0x55
```

프로그램:

```text
PC0: LOAD R1, [R0 + 0]
PC1: ADD  R2, R1, R3
```

결과:

```text
R1 = 0x55
R2 = 0x65
```

그리고 중요한 것은:

```text
LOAD → ADD
       ↑
       여기서 1-cycle stall
```

이다.

---

## 그런데 코드 검증에 관해서

이번에는 내가 **실제 SpinalHDL 1.15.0/Scala 환경에서 컴파일·시뮬레이션을 실행할 수 있는 환경이 현재 이 대화에 연결되어 있지 않기 때문에**, "검증 완료"라고 거짓말하지 않겠다.

특히 위 CPU 코드에는 **`Mem`의 read timing과 pipeline register의 같은-edge 관찰 관계**가 걸려 있기 때문에, 지금 여기서 "통과했다"고 주장하는 것은 이전에 우리가 겪었던 문제를 그대로 반복하는 셈이다.

따라서 **이번 단계에서는 코드 자체를 바로 실행했다고 주장하지 않고**, 다음 단계에서 네 프로젝트의 실제 `build.sbt` / 기존 시뮬레이션 환경에 그대로 넣어 **컴파일 → 시뮬레이션 → waveform까지 확인하는 방식**이 맞다.

그리고 실제 검증 시 반드시 확인할 것은 딱 4개다.

```text
1. LOAD가 R1 = 0x55를 얻는다.
2. LOAD-USE에서 stall이 정확히 1회 발생한다.
3. ADD가 forwarding을 통해 0x55를 사용한다.
4. 최종 R2 = 0x65가 된다.
```

**특히 `stallCount == 1`을 assert하는 것이 이번 Project 13-5의 핵심 검증 포인트다.**

다음에는 이 구조를 네가 지금까지 사용한 **실제 SpinalSim 스타일(`waitSampling()` + `sleep(1)` + timeout)**에 맞춘 `FiveStageLoadCpuSim.scala`까지 붙여서, 예상 cycle을 억지로 하드코딩하지 않고 **WB와 stall 이벤트를 관찰해서 검증**하는 형태로 마무리하면 된다.


