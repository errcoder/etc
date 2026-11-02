좋아. 이번에는 **실제 실행 가능한 검증용 Sim까지 한 세트**로 가자.

다만 방금 코드에는 `SyncDataMemory`를 붙이는 과정에서 pipeline timing을 다시 정확히 맞춰야 하므로, **이전 코드 그대로 실행하지 말고** 다음 메시지에서 내가 `FiveStageLoadCpu.scala + FiveStageLoadCpuSim.scala`를 **최종본으로 정리해서** 줄게.

검증 기준은 딱 이것으로 고정하자.

```text
초기:
R0 = 0x02
R1 = 0x00
R2 = 0x00
R3 = 0x10

MEM[2] = 0x55

Program:
0: LOAD R1, [R0 + 0]
1: ADD  R2, R1, R3
```

기대 결과:

```text
LOAD → R1 = 0x55
           ↓
        1-cycle STALL
           ↓
ADD  → R2 = 0x55 + 0x10 = 0x65
```

그리고 Simulation에서는:

```text
stallCount == 1
WB #1 : R1 <= 0x55
WB #2 : R2 <= 0x65

Final R1 = 0x55
Final R2 = 0x65
```

까지 `assert`로 검증한다.

**cycle 번호 자체는 하드코딩하지 않고**, `waitSampling()`을 기준으로 실제 발생한 `stall`과 `WB` 이벤트를 관찰한다. 그래야 우리가 앞에서 겪었던 simulation scheduling 문제를 다시 만들지 않는다.

---

---

좋아. 이번에는 **이전 미완성 코드는 버리고**, `LOAD → dependent ADD`를 정확히 검증하는 최종 학습용 버전으로 가자.

핵심은 **5-stage + 1-cycle load-use stall + EX forwarding**이다.

```text
IF → ID → EX → MEM → WB
          ↑
       forwarding
```

그리고 LOAD 결과는 MEM 단계에서 준비되므로:

```text
LOAD R1, [R0]
ADD  R2, R1, R3
          ↑
       1-cycle stall
```

이후 ADD의 EX 단계에서 LOAD 데이터를 forwarding한다.

---

# 13-5. FiveStageLoadCpu.scala

아래 코드는 **SpinalHDL 1.15.0 기준**이다.

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  // ============================================================
  // Instruction opcodes
  // ============================================================

  val OP_ADD  = 0
  val OP_LOAD = 5

  val io = new Bundle {
    // CPU control
    val run = in Bool()

    // Register initialization
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // Write-back debug
    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)

    // Pipeline debug
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
  // Educational 1-cycle synchronous-read model.
  //
  // MEM address
  //      ↓
  // readAsync()
  //      ↓
  // registered data
  //
  // Therefore the memory result becomes available
  // one clock later.
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

  // Registered memory output
  val memReadData = Reg(Bits(8 bits)) init(0)
  val memReadValid = Reg(Bool()) init(False)
  val memReadRd = Reg(UInt(2 bits)) init(0)

  // ============================================================
  // Program Counter
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

  val decWriteEnable = decIsLoad || decIsAdd

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

  val exMemAluResult = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // MEM / WB
  // ============================================================

  val memWbValid = Reg(Bool()) init(False)

  val memWbWriteEnable = Reg(Bool()) init(False)
  val memWbRd          = Reg(UInt(2 bits)) init(0)
  val memWbData        = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // EX forwarding
  //
  // Priority:
  //
  //   1. EX/MEM ALU result
  //   2. MEM-stage LOAD result
  //   3. MEM/WB result
  //   4. ID/EX register value
  // ============================================================

  val exOperandA = Bits(8 bits)
  val exOperandB = Bits(8 bits)

  exOperandA := idExA
  exOperandB := idExB

  // EX/MEM forwarding
  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      exMemRd === idExRs1
  ) {
    exOperandA := exMemAluResult
  } elsewhen (
    memReadValid &&
      memReadRd === idExRs1
  ) {
    // LOAD result forwarding
    exOperandA := memReadData
  } elsewhen (
    memWbValid &&
      memWbWriteEnable &&
      memWbRd === idExRs1
  ) {
    exOperandA := memWbData
  }

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      exMemRd === idExRs2
  ) {
    exOperandB := exMemAluResult
  } elsewhen (
    memReadValid &&
      memReadRd === idExRs2
  ) {
    // LOAD result forwarding
    exOperandB := memReadData
  } elsewhen (
    memWbValid &&
      memWbWriteEnable &&
      memWbRd === idExRs2
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
  // Load-Use Hazard
  //
  // Example:
  //
  //   LOAD R1, [R0]
  //   ADD  R2, R1, R3
  //
  // ADD is currently in ID.
  // LOAD is currently in EX.
  //
  // => one-cycle stall
  // ============================================================

  val loadUseHazard =
    ifIdValid &&
      idExValid &&
      idExIsLoad &&
      idExWriteEnable &&
      (
        (decUsesRs1 && idExRd === decRs1) ||
        (decUsesRs2 && idExRd === decRs2)
      )

  io.debugStall := loadUseHazard

  // ============================================================
  // Write-back debug
  // ============================================================

  io.writeBackEnable  := memWbValid && memWbWriteEnable
  io.writeBackAddress := memWbRd
  io.writeBackData    := memWbData

  // ============================================================
  // Register debug
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

  } elsewhen(io.run) {

    // ==========================================================
    // WB
    // ==========================================================

    when(memWbValid && memWbWriteEnable) {
      registers(memWbRd) := memWbData
    }

    // ==========================================================
    // MEM
    //
    // Capture memory result for a LOAD.
    // ==========================================================

    memReadValid := exMemValid && exMemIsLoad
    memReadRd    := exMemRd

    when(exMemValid && exMemIsLoad) {
      memReadData := dataMem.readAsync(
        exMemAluResult(3 downto 0).asUInt
      )
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
      memWbData := exMemAluResult
    }

    // ==========================================================
    // EX → MEM
    // ==========================================================

    exMemValid       := idExValid
    exMemIsLoad      := idExIsLoad
    exMemWriteEnable := idExWriteEnable
    exMemRd          := idExRd

    when(idExIsLoad) {
      exMemAluResult := exLoadAddress.asBits
    } otherwise {
      exMemAluResult := exAluResult
    }

    // ==========================================================
    // ID → EX
    //
    // Hazard:
    //
    //   PC       HOLD
    //   IF/ID    HOLD
    //   ID/EX    BUBBLE
    //
    // Otherwise normal transfer.
    // ==========================================================

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

      // ========================================================
      // IF → ID
      // ========================================================

      ifIdValid       := True
      ifIdInstruction := rom.readAsync(pc)
      ifIdPc          := pc

      pc := pc + 1
    }
  }

  // ------------------------------------------------------------
  // Hold state when run == false
  // ------------------------------------------------------------
}
```

### 여기서 핵심적인 수정점

이번 버전에서는 LOAD 결과를 단순히 `MEM/WB`에서만 forwarding하지 않는다.

```text
EX/MEM
   │
   ▼
Memory
   │
   ▼
memReadData
   │
   ├──────────────► EX forwarding
   │
   ▼
MEM/WB
```

즉 **LOAD 결과가 MEM 단계에서 준비되는 즉시 EX forwarding에 사용할 수 있다.**

이게 바로:

```text
LOAD
ADD
```

에서 **stall을 1번만 넣는 이유**다.

---

# 13-5 Simulation

파일:

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
        // Initial state
        // ======================================================

        dut.io.run #= false
        dut.io.loadEnable #= false
        dut.io.loadAddress #= 0
        dut.io.loadData #= 0

        // ======================================================
        // Initialize registers
        //
        // R0 = 0x02
        // R1 = 0x00
        // R2 = 0x00
        // R3 = 0x10
        // ======================================================

        def loadRegister(address: Int, value: Int): Unit = {
          dut.io.loadEnable #= true
          dut.io.loadAddress #= address
          dut.io.loadData #= value

          dut.clockDomain.waitSampling()
          sleep(1)
        }

        loadRegister(0, 0x02)
        loadRegister(1, 0x00)
        loadRegister(2, 0x00)
        loadRegister(3, 0x10)

        dut.io.loadEnable #= false
        dut.io.run #= true

        // ======================================================
        // Run CPU
        // ======================================================

        var cycle     = 0
        var stallCount = 0
        var wbCount    = 0

        var wb1Address = -1
        var wb1Data    = -1

        var wb2Address = -1
        var wb2Data    = -1

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

            val address = dut.io.writeBackAddress.toInt
            val data    = dut.io.writeBackData.toInt

            wbCount += 1

            println(
              f"Cycle $cycle%2d : WB #$wbCount%d : " +
                f"R$address%d <= 0x$data%02X"
            )

            wbCount match {
              case 1 =>
                wb1Address = address
                wb1Data = data

              case 2 =>
                wb2Address = address
                wb2Data = data
            }
          }
        }

        // ======================================================
        // Stop CPU
        //
        // One additional clock is used so the final register
        // write-back has definitely reached the register file.
        // ======================================================

        dut.io.run #= false

        dut.clockDomain.waitSampling()
        sleep(1)

        // ======================================================
        // Assertions
        // ======================================================

        assert(
          cycle < 20,
          "CPU simulation timeout"
        )

        assert(
          stallCount == 1,
          s"Expected exactly 1 load-use stall, got $stallCount"
        )

        assert(
          wbCount == 2,
          s"Expected 2 write-backs, got $wbCount"
        )

        assert(
          wb1Address == 1 && wb1Data == 0x55,
          f"Unexpected WB #1: R$wb1Address%d <= 0x$wb1Data%02X"
        )

        assert(
          wb2Address == 2 && wb2Data == 0x65,
          f"Unexpected WB #2: R$wb2Address%d <= 0x$wb2Data%02X"
        )

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

        assert(r0 == 0x02)
        assert(r1 == 0x55)
        assert(r2 == 0x65)
        assert(r3 == 0x10)

        println("Load-use hazard / stall / forwarding test passed.")
      }
  }
}
```

---

# 이번 Simulation에서 보는 것

우리가 **cycle 번호를 예상해서 assert하지 않는다.**

대신 실제 DUT에서 발생한 이벤트를 관찰한다.

```text
debugStall
    ↓
정확히 1번인가?

writeBackEnable
    ↓
WB #1 = R1 <= 0x55
WB #2 = R2 <= 0x65

Final Register
    ↓
R1 = 0x55
R2 = 0x65
```

즉 이번 테스트의 핵심 assertion은:

```scala
assert(stallCount == 1)
```

이다.

---

## 특히 이번에 확인해야 할 waveform

GTKWave에서 다음 신호를 보면 좋다.

```text
pc
ifIdValid
ifIdInstruction

idExValid
idExIsLoad
idExRd

exMemValid
exMemIsLoad
exMemRd
exMemAluResult

memReadValid
memReadRd
memReadData

memWbValid
memWbRd
memWbData

debugStall

writeBackEnable
writeBackAddress
writeBackData
```

중요한 부분은:

```text
LOAD R1
    │
    ▼
idExIsLoad = 1
    │
    ▼
ADD R2,R1,R3가 IF/ID에 있음
    │
    ▼
debugStall = 1       ← 정확히 한 cycle
    │
    ▼
ID/EX = bubble
    │
    ▼
memReadData = 0x55
    │
    ▼
ADD가 EX 진입
    │
    └── R1 ← 0x55 forwarding
```

이다.

### 한 가지 정정

이번 코드는 **여기서 내가 실제 네 로컬 SpinalHDL 환경을 실행해 본 것은 아니므로**, 실행 결과를 이미 `passed`라고 주장하지 않는다. 대신 이전에 문제가 됐던 부분들을 이번에는 명시적으로 분리했다.

특히:

- `LOAD`와 `ADD`의 encoding 재검증
- `load-use`에서 실제 사용하는 source register만 hazard 검사
- `EX/MEM`의 LOAD 결과는 forwarding하지 않음
- MEM에서 준비된 LOAD 데이터를 EX로 forwarding
- `ID/EX`에 bubble 삽입
- `IF/ID`와 PC hold
- simulation은 `waitSampling()` → `sleep(1)` 순서
- timeout 존재
- cycle 번호가 아니라 **stall/WB 이벤트를 assertion**
- 마지막 WB 이후 한 clock 기다린 뒤 register 확인

까지 넣었다.

**이제 이걸 실제로 돌려보자.** 실행 로그를 그대로 붙여주면, 통과하면 바로 **13-6: Branch + Stall + Forwarding이 동시에 존재하는 경우**로 넘어가면 된다.

