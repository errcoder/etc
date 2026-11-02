좋다. 이번에는 **전체를 다시 처음부터 검산해서** 진행하자.

특히 방금 두 번 오류가 있었던 instruction encoding은 더 이상 추측하지 않고 **필드별 shift 계산**으로 고정한다.

## 1. 먼저 Encoding부터 확정

R-type:

```text
15       12 11     10 9      8 7      6 5        0
+-----------+---------+--------+--------+----------+
|  opcode   |   rd    |  rs1   |  rs2   | reserved |
+-----------+---------+--------+--------+----------+
```

### `LOAD R1, [R0 + 0]`

```text
opcode = 5
rd     = 1
rs1    = 0
offset = 0
```

계산:

```text
opcode = 5 << 12 = 0x5000
rd     = 1 << 10 = 0x0400
rs1    = 0 <<  8 = 0x0000
offset             0x0000
-------------------------
                   0x5400
```

따라서:

```scala
B"16'h5400"
```

### `ADD R2, R1, R3`

```text
opcode = 0
rd     = 2
rs1    = 1
rs2    = 3
```

계산:

```text
opcode = 0 << 12 = 0x0000
rd     = 2 << 10 = 0x0800
rs1    = 1 <<  8 = 0x0100
rs2    = 3 <<  6 = 0x00C0
-------------------------
                   0x09C0
```

따라서:

```scala
B"16'h09C0"
```

**이번에는 `0x5400`, `0x09C0`로 확정한다.**

---

# 2. 이번 CPU의 정확한 구조

이번에는 memory timing을 더 단순하고 명확하게 만든다.

```text
IF → ID → EX → MEM → WB
              │
              └── LOAD address
                     │
                     ▼
                  Data RAM
                     │
                     ▼
                   MEM/WB
                     │
                     ▼
                  Forwarding
                     │
                     ▼
                    EX
```

LOAD의 경우:

```text
Cycle N
    LOAD → EX
    address 계산

Cycle N+1
    LOAD → MEM
    memory data → MEM/WB

    ADD는 아직 ID에 있음
    → 1 cycle stall

Cycle N+2
    ADD → EX
    MEM/WB의 LOAD data → EX forwarding

Cycle N+3
    ADD → MEM

Cycle N+4
    ADD → WB
```

이 구조에서는 **별도의 `memReadData` 레지스터를 하나 더 만들지 않는다.**

`EX/MEM`의 주소가 한 cycle 동안 안정되어 있고:

```scala
val memReadData = dataMem.readAsync(exMemAluResult(...))
```

로 memory output을 만들고, 그 값을 `MEM/WB` 레지스터에 저장한다.

즉 교육적으로:

```text
EX/MEM address
      ↓
  async memory lookup
      ↓
   MEM/WB register
```

가 되어 **정확히 한 pipeline cycle의 memory latency**를 표현한다.

---

# 3. 최종 `FiveStageLoadCpu.scala`

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  // ============================================================
  // Opcodes
  // ============================================================

  val OP_ADD  = 0
  val OP_LOAD = 5

  val io = new Bundle {
    val run = in Bool()

    // Register initialization
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // Write-back debug
    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)

    // Hazard debug
    val debugStall = out Bool()

    // Register debug
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
  // Instruction ROM
  //
  // 0: LOAD R1, [R0 + 0]
  // 1: ADD  R2, R1, R3
  // ============================================================

  val rom = Mem(Bits(16 bits), wordCount = 16)

  rom.init(
    Seq(
      B"16'h5400", // 0: LOAD R1, [R0 + 0]
      B"16'h09C0", // 1: ADD  R2, R1, R3
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
  // MEM[2] = 0x55
  //
  // The EX/MEM address is stable for one clock cycle.
  // readAsync() observes that address during the MEM cycle.
  // The result is captured into MEM/WB at the next clock edge.
  //
  // This gives us one-cycle synchronous-read behavior at
  // the CPU pipeline level.
  // ============================================================

  val dataMem = Mem(Bits(8 bits), wordCount = 16)

  dataMem.init(
    Seq(
      B"8'h00", // 0
      B"8'h00", // 1
      B"8'h55", // 2
      B"8'h00", // 3
      B"8'h00", // 4
      B"8'h00", // 5
      B"8'h00", // 6
      B"8'h00", // 7
      B"8'h00", // 8
      B"8'h00", // 9
      B"8'h00", // 10
      B"8'h00", // 11
      B"8'h00", // 12
      B"8'h00", // 13
      B"8'h00", // 14
      B"8'h00"  // 15
    )
  )

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

  val decUsesRs1 =
    decIsLoad || decIsAdd

  val decUsesRs2 =
    decIsAdd

  val decWriteEnable =
    decIsLoad || decIsAdd

  // ============================================================
  // ID / EX
  // ============================================================

  val idExValid = Reg(Bool()) init(False)

  val idExIsLoad      = Reg(Bool()) init(False)
  val idExWriteEnable = Reg(Bool()) init(False)

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

  // For ADD:
  //   ALU result
  //
  // For LOAD:
  //   memory address
  //
  val exMemResult = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // MEM / WB
  // ============================================================

  val memWbValid = Reg(Bool()) init(False)

  val memWbWriteEnable = Reg(Bool()) init(False)
  val memWbRd          = Reg(UInt(2 bits)) init(0)
  val memWbData        = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // EX Forwarding
  //
  // Priority:
  //
  // 1. EX/MEM ALU result
  // 2. MEM/WB data
  // 3. ID/EX register value
  //
  // EX/MEM LOAD is deliberately NOT forwarded because
  // its actual data is not available yet.
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
    exOperandA := exMemResult
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
    exOperandB := exMemResult
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

  val exAluResult =
    exOperandA + exOperandB

  val exLoadAddress =
    exOperandA.asUInt + idExOffset.asUInt

  // ============================================================
  // Load-Use Hazard Detection
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
  // Data Memory read
  //
  // IMPORTANT:
  //
  // exMemResult contains the LOAD address during the MEM cycle.
  // readAsync() is only used as the memory's combinational lookup.
  // The result is captured into MEM/WB below.
  //
  // Therefore the CPU-level memory operation is one cycle.
  // ============================================================

  val memReadData =
    dataMem.readAsync(
      exMemResult(3 downto 0).asUInt
    )

  // ============================================================
  // Debug outputs
  // ============================================================

  io.writeBackEnable  := memWbValid && memWbWriteEnable
  io.writeBackAddress := memWbRd
  io.writeBackData    := memWbData

  io.debugR0 := registers(0)
  io.debugR1 := registers(1)
  io.debugR2 := registers(2)
  io.debugR3 := registers(3)

  // ============================================================
  // Sequential Pipeline
  // ============================================================

  when(io.loadEnable) {

    // ----------------------------------------------------------
    // Register initialization mode
    // ----------------------------------------------------------

    registers(io.loadAddress) := io.loadData

  } elsewhen(io.run) {

    // ==========================================================
    // WB
    // ==========================================================

    when(memWbValid && memWbWriteEnable) {
      registers(memWbRd) := memWbData
    }

    // ==========================================================
    // MEM → WB
    // ==========================================================

    memWbValid       := exMemValid
    memWbWriteEnable := exMemWriteEnable
    memWbRd          := exMemRd

    when(exMemIsLoad) {
      memWbData := memReadData
    } otherwise {
      memWbData := exMemResult
    }

    // ==========================================================
    // EX → MEM
    // ==========================================================

    exMemValid       := idExValid
    exMemIsLoad      := idExIsLoad
    exMemWriteEnable := idExWriteEnable
    exMemRd          := idExRd

    when(idExIsLoad) {
      exMemResult := exLoadAddress.asBits
    } otherwise {
      exMemResult := exAluResult
    }

    // ==========================================================
    // ID → EX
    // ==========================================================

    when(loadUseHazard) {

      // Insert bubble into ID/EX.
      //
      // IF/ID and PC are NOT changed.
      // Therefore the dependent instruction remains in ID.
      //

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

      // ========================================================
      // IF → ID
      // ========================================================

      ifIdValid       := True
      ifIdInstruction := rom.readAsync(pc)
      ifIdPc          := pc

      pc := pc + 1
    }
  }
}
```

---

# 4. 왜 이번 memory 구조가 이전 코드보다 낫나?

이번에는 이 부분이 중요하다.

### LOAD가 EX에 있을 때

```text
idEx
 LOAD R1
    │
    ▼
EX
 address = R0 + offset
    │
    ▼
EX/MEM
```

`EX/MEM`에 주소가 저장된다.

그 다음 cycle:

```text
EX/MEM address
      │
      ▼
dataMem.readAsync()
      │
      ▼
MEM/WB register
```

따라서 **MEM/WB에는 정확히 LOAD 결과가 들어간다.**

그리고 그 cycle에 dependent ADD는 stall되어 있다.

다음 cycle:

```text
MEM/WB
 R1 = 0x55
    │
    │ forwarding
    ▼
   EX
 ADD R2,R1,R3
```

그래서:

```text
0x55 + 0x10 = 0x65
```

가 된다.

---

# 5. Simulation

`FiveStageLoadCpuSim.scala`

```scala
import spinal.core._
import spinal.core.sim._

object FiveStageLoadCpuSim {

  def main(args: Array[String]): Unit = {

    SimConfig
      .withWave
      .compile(new FiveStageLoadCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(period = 10)

        // ======================================================
        // Initial control
        // ======================================================

        dut.io.run #= false

        dut.io.loadEnable #= false
        dut.io.loadAddress #= 0
        dut.io.loadData #= 0

        // ======================================================
        // Register initialization
        // ======================================================

        def loadRegister(
            address: Int,
            data: Int
        ): Unit = {

          dut.io.loadEnable #= true
          dut.io.loadAddress #= address
          dut.io.loadData #= data

          dut.clockDomain.waitSampling()
          sleep(1)
        }

        loadRegister(0, 0x02)
        loadRegister(1, 0x00)
        loadRegister(2, 0x00)
        loadRegister(3, 0x10)

        dut.io.loadEnable #= false

        // ======================================================
        // Start CPU
        // ======================================================

        dut.io.run #= true

        var cycle = 0

        var stallCount = 0
        var wbCount = 0

        var wb1Address = -1
        var wb1Data = -1

        var wb2Address = -1
        var wb2Data = -1

        // ======================================================
        // Run until two write-backs
        //
        // Timeout protects against an unexpected pipeline bug.
        // ======================================================

        while (wbCount < 2 && cycle < 20) {

          dut.clockDomain.waitSampling()
          sleep(1)

          cycle += 1

          // ----------------------------------------------------
          // Stall observation
          // ----------------------------------------------------

          if (dut.io.debugStall.toBoolean) {

            stallCount += 1

            println(
              f"Cycle $cycle%2d : LOAD-USE STALL"
            )
          }

          // ----------------------------------------------------
          // Write-back observation
          // ----------------------------------------------------

          if (dut.io.writeBackEnable.toBoolean) {

            val address =
              dut.io.writeBackAddress.toInt

            val data =
              dut.io.writeBackData.toInt

            wbCount += 1

            println(
              f"Cycle $cycle%2d : " +
                f"WB #$wbCount%d : " +
                f"R$address%d <= 0x$data%02X"
            )

            if (wbCount == 1) {
              wb1Address = address
              wb1Data = data
            }

            if (wbCount == 2) {
              wb2Address = address
              wb2Data = data
            }
          }
        }

        // ======================================================
        // Stop CPU
        // ======================================================

        dut.io.run #= false

        // Allow final WB register update to settle.
        dut.clockDomain.waitSampling()
        sleep(1)

        // ======================================================
        // Assertions
        // ======================================================

        assert(
          cycle < 20,
          s"CPU simulation timeout: cycle=$cycle"
        )

        assert(
          stallCount == 1,
          s"Expected exactly 1 stall, got $stallCount"
        )

        assert(
          wbCount == 2,
          s"Expected exactly 2 write-backs, got $wbCount"
        )

        assert(
          wb1Address == 1 &&
            wb1Data == 0x55,
          f"Unexpected WB #1: " +
            f"R$wb1Address%d <= 0x$wb1Data%02X"
        )

        assert(
          wb2Address == 2 &&
            wb2Data == 0x65,
          f"Unexpected WB #2: " +
            f"R$wb2Address%d <= 0x$wb2Data%02X"
        )

        // ======================================================
        // Final register values
        // ======================================================

        val r0 = dut.io.debugR0.toInt
        val r1 = dut.io.debugR1.toInt
        val r2 = dut.io.debugR2.toInt
        val r3 = dut.io.debugR3.toInt

        println()
        println(f"Final R0 = 0x$r0%02X")
        println(f"Final R1 = 0x$r1%02X")
        println(f"Final R2 = 0x$r2%02X")
        println(f"Final R3 = 0x$r3%02X")
        println()

        assert(
          r0 == 0x02,
          f"R0 expected 0x02, got 0x$r0%02X"
        )

        assert(
          r1 == 0x55,
          f"R1 expected 0x55, got 0x$r1%02X"
        )

        assert(
          r2 == 0x65,
          f"R2 expected 0x65, got 0x$r2%02X"
        )

        assert(
          r3 == 0x10,
          f"R3 expected 0x10, got 0x$r3%02X"
        )

        println(
          "Load-use hazard / 1-cycle stall / forwarding test passed."
        )
      }
  }
}
```

---

# 6. 이번 테스트에서 반드시 나와야 하는 결과

정확한 cycle 번호는 simulation scheduling에 따라 우리가 여기서 미리 박아 넣지 않는다.

하지만 **이벤트의 순서와 값은 고정**이다.

```text
LOAD-USE STALL          ← 정확히 1회

WB #1 : R1 <= 0x55

WB #2 : R2 <= 0x65

Final R0 = 0x02
Final R1 = 0x55
Final R2 = 0x65
Final R3 = 0x10

Load-use hazard / 1-cycle stall / forwarding test passed.
```

여기서 가장 중요한 검증 세 가지는:

```text
1. stallCount == 1
2. WB #1 = R1 <= 0x55
3. WB #2 = R2 <= 0x65
```

이다.

---

## 그리고 이번에는 Encoding을 코드에서도 명확히 확인할 수 있다

```text
LOAD R1,[R0+0]

  0101 01 00 00000000
  └──┘ └─┘ └─┘ └──────┘
   5    1   0     0

  = 0x5400
```

```text
ADD R2,R1,R3

  0000 10 01 11 000000
  └──┘ └─┘ └─┘ └─┘ └──┘
   0    2   1   3    0

  = 0x09C0
```

**이번에는 이 두 값이 확정이다.**

이 코드를 실제 프로젝트에서 실행해서 로그를 가져오면, 이번 Project 13-5에서는 **stall 발생 시점 → LOAD memory 결과 → forwarding → ADD WB**를 waveform까지 한 번 확인하고 넘어가자.
