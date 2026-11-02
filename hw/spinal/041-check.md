좋습니다. 이번에는 **Project 13-5 전체 코드를 한 번에 정리**하겠습니다.

아래 코드는 지금까지 확인한 사항을 모두 반영했습니다.

- SpinalHDL **1.15.0 / Release 34**
- 5-stage pipeline: `IF → ID → EX → MEM → WB`
- `LOAD`
- `ADD`
- Load-use hazard 검출
- 정확히 1-cycle stall
- `EX/MEM → EX` forwarding
- `MEM/WB → EX` forwarding
- `Bits`에 직접 `+`를 사용하지 않음
- LOAD address는 `UInt + UInt`
- instruction encoding 수정
- pipeline register valid 관리
- simulation에서 stall/WB 관찰 가능
- simulation timeout
- 최종 register 검증

단, **이 대화 환경에는 SpinalHDL 1.15.0 compiler가 설치되어 있지 않아서 실제 compile/run을 실행했다고 주장하지는 않겠습니다.** 아래 코드는 타입과 pipeline timing을 기준으로 전체를 다시 검토한 버전입니다.

---

# 1. `FiveStageLoadCpu.scala`

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  val io = new Bundle {
    // CPU control
    val run = in Bool()

    // Register initialization interface
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // Debug
    val pc = out UInt(4 bits)

    val stall = out Bool()

    val wbEnable  = out Bool()
    val wbAddress = out UInt(2 bits)
    val wbData    = out Bits(8 bits)

    val reg0 = out Bits(8 bits)
    val reg1 = out Bits(8 bits)
    val reg2 = out Bits(8 bits)
    val reg3 = out Bits(8 bits)
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
  // Instruction format
  //
  // R-type
  //
  // 15       12 11     10 9      8 7      6 5        0
  // +-----------+---------+--------+--------+----------+
  // |  opcode   |   rd    |  rs1   |  rs2   | reserved |
  // +-----------+---------+--------+--------+----------+
  //
  // LOAD
  //
  // 15       12 11     10 9      8 7            0
  // +-----------+---------+--------+--------------+
  // |  opcode   |   rd    |  rs1   |    offset    |
  // +-----------+---------+--------+--------------+
  //
  // opcode:
  //   0000 = ADD
  //   0101 = LOAD
  // ============================================================

  val rom = Mem(Bits(16 bits), wordCount = 16)

  rom.init(
    Seq(
      B"16'h5400", // PC 0 : LOAD R1, [R0 + 0]
      B"16'h09C0", // PC 1 : ADD  R2, R1, R3
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000",
      B"16'h0000"
    )
  )

  // ============================================================
  // Data RAM
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
  // Program Counter
  // ============================================================

  val pc = Reg(UInt(4 bits)) init(0)

  val fetchInstruction = rom.readAsync(pc)

  // ============================================================
  // IF / ID pipeline register
  // ============================================================

  val ifIdValid       = Reg(Bool()) init(False)
  val ifIdInstruction = Reg(Bits(16 bits)) init(0)

  // ============================================================
  // Decode
  // ============================================================

  val decOpcode = ifIdInstruction(15 downto 12)

  val decRd  = ifIdInstruction(11 downto 10).asUInt
  val decRs1 = ifIdInstruction(9 downto 8).asUInt
  val decRs2 = ifIdInstruction(7 downto 6).asUInt

  val decOffset = ifIdInstruction(7 downto 0)

  val decIsAdd  = decOpcode === B"4'b0000"
  val decIsLoad = decOpcode === B"4'b0101"

  val decUsesRs1 = decIsLoad || decIsAdd
  val decUsesRs2 = decIsAdd

  val decWriteEnable = decIsLoad || decIsAdd

  // Register File read
  val decRegA = registers(decRs1)
  val decRegB = registers(decRs2)

  // ============================================================
  // ID / EX pipeline register
  // ============================================================

  val idExValid = Reg(Bool()) init(False)

  val idExIsLoad      = Reg(Bool()) init(False)
  val idExWriteEnable = Reg(Bool()) init(False)

  val idExRd = Reg(UInt(2 bits)) init(0)

  val idExRegA = Reg(Bits(8 bits)) init(0)
  val idExRegB = Reg(Bits(8 bits)) init(0)

  val idExOffset = Reg(Bits(8 bits)) init(0)

  // ============================================================
  // Forwarding
  //
  // Priority:
  //
  //   1. EX/MEM
  //   2. MEM/WB
  //   3. Register File value
  //
  // LOAD in EX/MEM is NOT forwarded because its data
  // does not exist yet.
  // ============================================================

  val exMemValid       = Reg(Bool()) init(False)
  val exMemIsLoad      = Reg(Bool()) init(False)
  val exMemWriteEnable = Reg(Bool()) init(False)
  val exMemRd          = Reg(UInt(2 bits)) init(0)
  val exMemResult      = Reg(Bits(8 bits)) init(0)

  val memWbValid       = Reg(Bool()) init(False)
  val memWbIsLoad      = Reg(Bool()) init(False)
  val memWbWriteEnable = Reg(Bool()) init(False)
  val memWbRd          = Reg(UInt(2 bits)) init(0)
  val memWbResult      = Reg(Bits(8 bits)) init(0)

  val exForwardA = Bits(8 bits)
  val exForwardB = Bits(8 bits)

  exForwardA := idExRegA
  exForwardB := idExRegB

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRd)
  ) {
    exForwardA := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRd)
    ) {
      exForwardA := memWbResult
    }
  }

  // Operand B needs rs2 address, not rd.
  // Therefore forwarding conditions must use the actual
  // ID/EX source register numbers.
  //
  // We need to retain rs1/rs2 in ID/EX.

  val idExRs1 = Reg(UInt(2 bits)) init(0)
  val idExRs2 = Reg(UInt(2 bits)) init(0)

  exForwardA := idExRegA
  exForwardB := idExRegB

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs1)
  ) {
    exForwardA := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs1)
    ) {
      exForwardA := memWbResult
    }
  }

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs2)
  ) {
    exForwardB := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs2)
    ) {
      exForwardB := memWbResult
    }
  }

  // ============================================================
  // EX stage
  // ============================================================

  // Bits does NOT have '+'
  //
  // Therefore:
  //
  //   Bits -> UInt
  //   UInt + UInt
  //   UInt -> Bits
  //
  val exAluResult =
    (exForwardA.asUInt + exForwardB.asUInt).asBits

  // LOAD address:
  //
  // UInt + UInt
  //
  val exLoadAddress =
    exForwardA.asUInt + idExOffset.asUInt

  val exResult = Bits(8 bits)

  exResult := exAluResult

  when(idExIsLoad) {
    exResult := exLoadAddress.asBits
  }

  // ============================================================
  // Load-use hazard detection
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

  // ============================================================
  // EX / MEM pipeline
  // ============================================================

  // exMemResult:
  //
  //   ADD -> ALU result
  //   LOAD -> calculated memory address
  //
  // For LOAD, the actual data will be obtained in MEM.
  //
  // ============================================================

  // ============================================================
  // MEM stage
  // ============================================================

  val dataMemAddress =
    exMemResult(3 downto 0).asUInt

  val memReadData =
    dataMem.readAsync(dataMemAddress)

  // ============================================================
  // Debug stall pulse
  // ============================================================

  val stallPulse = Reg(Bool()) init(False)

  // ============================================================
  // Main sequential pipeline
  // ============================================================

  when(io.loadEnable) {
    registers(io.loadAddress) := io.loadData
  }

  when(io.run) {

    // ----------------------------------------------------------
    // WB
    // ----------------------------------------------------------

    when(memWbValid && memWbWriteEnable) {
      registers(memWbRd) := memWbResult
    }

    // ----------------------------------------------------------
    // MEM -> WB
    // ----------------------------------------------------------

    memWbValid       := exMemValid
    memWbIsLoad      := exMemIsLoad
    memWbWriteEnable := exMemWriteEnable
    memWbRd          := exMemRd

    when(exMemIsLoad) {
      memWbResult := memReadData
    } otherwise {
      memWbResult := exMemResult
    }

    // ----------------------------------------------------------
    // EX -> MEM
    // ----------------------------------------------------------

    exMemValid       := idExValid
    exMemIsLoad      := idExIsLoad
    exMemWriteEnable := idExWriteEnable
    exMemRd          := idExRd
    exMemResult      := exResult

    // ----------------------------------------------------------
    // Default: no stall pulse
    // ----------------------------------------------------------

    stallPulse := False

    // ----------------------------------------------------------
    // ID -> EX
    //
    // On load-use hazard:
    //
    //   - PC holds
    //   - IF/ID holds
    //   - ID/EX becomes bubble
    //
    // This creates exactly one stall cycle.
    // ----------------------------------------------------------

    when(loadUseHazard) {

      // Bubble into ID/EX
      idExValid       := False
      idExIsLoad      := False
      idExWriteEnable := False
      idExRd          := 0
      idExRegA        := 0
      idExRegB        := 0
      idExOffset      := 0
      idExRs1         := 0
      idExRs2         := 0

      // Hold IF/ID
      ifIdValid       := ifIdValid
      ifIdInstruction := ifIdInstruction

      // Hold PC
      pc := pc

      // Debug
      stallPulse := True

    } otherwise {

      // --------------------------------------------------------
      // Normal ID -> EX
      // --------------------------------------------------------

      idExValid       := ifIdValid
      idExIsLoad      := ifIdValid && decIsLoad
      idExWriteEnable := ifIdValid && decWriteEnable

      idExRd     := decRd
      idExRs1    := decRs1
      idExRs2    := decRs2
      idExRegA   := decRegA
      idExRegB   := decRegB
      idExOffset := decOffset

      // --------------------------------------------------------
      // IF -> ID
      // --------------------------------------------------------

      ifIdValid       := True
      ifIdInstruction := fetchInstruction

      // --------------------------------------------------------
      // PC
      // --------------------------------------------------------

      pc := pc + 1
    }
  }

  // ============================================================
  // Outputs
  // ============================================================

  io.pc := pc

  io.stall := stallPulse

  io.wbEnable  := memWbValid && memWbWriteEnable
  io.wbAddress := memWbRd
  io.wbData    := memWbResult

  io.reg0 := registers(0)
  io.reg1 := registers(1)
  io.reg2 := registers(2)
  io.reg3 := registers(3)
}
```

---

# 2. 중요한 부분 하나 수정

위 코드에서 `exForwardA`를 처음에 한 번 정의하고 forwarding을 지정한 뒤, `idExRs1/idExRs2`를 선언하고 다시 forwarding을 지정하는 형태가 되어 있으므로 **실제 제출용 코드는 중복을 제거하는 것이 맞다.**

따라서 위 코드를 그대로 복사하기보다는 아래의 **정리된 forwarding 부분**으로 교체해야 한다.

`ID/EX pipeline register` 부분부터:

```scala
  val idExValid = Reg(Bool()) init(False)

  val idExIsLoad      = Reg(Bool()) init(False)
  val idExWriteEnable = Reg(Bool()) init(False)

  val idExRd = Reg(UInt(2 bits)) init(0)

  val idExRs1 = Reg(UInt(2 bits)) init(0)
  val idExRs2 = Reg(UInt(2 bits)) init(0)

  val idExRegA = Reg(Bits(8 bits)) init(0)
  val idExRegB = Reg(Bits(8 bits)) init(0)

  val idExOffset = Reg(Bits(8 bits)) init(0)
```

그리고 forwarding은 **한 번만** 다음처럼 작성한다.

```scala
  val exMemValid       = Reg(Bool()) init(False)
  val exMemIsLoad      = Reg(Bool()) init(False)
  val exMemWriteEnable = Reg(Bool()) init(False)
  val exMemRd          = Reg(UInt(2 bits)) init(0)
  val exMemResult      = Reg(Bits(8 bits)) init(0)

  val memWbValid       = Reg(Bool()) init(False)
  val memWbIsLoad      = Reg(Bool()) init(False)
  val memWbWriteEnable = Reg(Bool()) init(False)
  val memWbRd          = Reg(UInt(2 bits)) init(0)
  val memWbResult      = Reg(Bits(8 bits)) init(0)

  val exForwardA = Bits(8 bits)
  val exForwardB = Bits(8 bits)

  exForwardA := idExRegA
  exForwardB := idExRegB

  // ------------------------------------------------------------
  // Forward A : rs1
  // ------------------------------------------------------------

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs1)
  ) {
    exForwardA := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs1)
    ) {
      exForwardA := memWbResult
    }
  }

  // ------------------------------------------------------------
  // Forward B : rs2
  // ------------------------------------------------------------

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs2)
  ) {
    exForwardB := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs2)
    ) {
      exForwardB := memWbResult
    }
  }
```

**최종 코드에서는 이 정리된 버전을 사용해야 한다.**

---

# 3. Simulation

파일명:

```text
FiveStageLoadCpuSim.scala
```

```scala
import spinal.core.sim._

object FiveStageLoadCpuSim {

  def main(args: Array[String]): Unit = {

    SimConfig
      .withWave
      .compile(new FiveStageLoadCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(period = 10)

        // ------------------------------------------------------
        // Initial values
        // ------------------------------------------------------

        dut.io.run #= false

        dut.io.loadEnable  #= false
        dut.io.loadAddress #= 0
        dut.io.loadData    #= 0

        // ------------------------------------------------------
        // Load initial register values
        //
        // R0 = 0x02
        // R1 = 0x00
        // R2 = 0x00
        // R3 = 0x10
        // ------------------------------------------------------

        def loadRegister(address: Int, value: Int): Unit = {
          dut.io.loadEnable  #= true
          dut.io.loadAddress #= address
          dut.io.loadData    #= value

          dut.clockDomain.waitSampling()
          sleep(1)

          dut.io.loadEnable #= false
        }

        loadRegister(0, 0x02)
        loadRegister(1, 0x00)
        loadRegister(2, 0x00)
        loadRegister(3, 0x10)

        // ------------------------------------------------------
        // Start CPU
        // ------------------------------------------------------

        dut.io.run #= true

        var cycle = 0
        var wbCount = 0
        var stallCount = 0

        var wb1Address = -1
        var wb1Data = -1

        var wb2Address = -1
        var wb2Data = -1

        // ------------------------------------------------------
        // Observe pipeline
        // ------------------------------------------------------

        while (wbCount < 2 && cycle < 20) {

          dut.clockDomain.waitSampling()
          sleep(1)

          cycle += 1

          val pc =
            dut.io.pc.toInt

          val stall =
            dut.io.stall.toBoolean

          val wbEnable =
            dut.io.wbEnable.toBoolean

          val wbAddress =
            dut.io.wbAddress.toInt

          val wbData =
            dut.io.wbData.toInt

          if (stall) {
            stallCount += 1

            println(
              f"Cycle $cycle%2d : STALL"
            )
          }

          if (wbEnable) {

            wbCount += 1

            println(
              f"Cycle $cycle%2d : " +
                f"WB #$wbCount%d : " +
                f"R$wbAddress%d <= 0x$wbData%02X"
            )

            if (wbCount == 1) {
              wb1Address = wbAddress
              wb1Data = wbData
            }

            if (wbCount == 2) {
              wb2Address = wbAddress
              wb2Data = wbData
            }
          } else {
            println(
              f"Cycle $cycle%2d : PC=$pc%d"
            )
          }
        }

        // ------------------------------------------------------
        // Stop CPU
        // ------------------------------------------------------

        dut.io.run #= false

        dut.clockDomain.waitSampling()
        sleep(1)

        // ------------------------------------------------------
        // Final register values
        // ------------------------------------------------------

        val r0 = dut.io.reg0.toInt
        val r1 = dut.io.reg1.toInt
        val r2 = dut.io.reg2.toInt
        val r3 = dut.io.reg3.toInt

        println()
        println(f"Final R0 = 0x$r0%02X")
        println(f"Final R1 = 0x$r1%02X")
        println(f"Final R2 = 0x$r2%02X")
        println(f"Final R3 = 0x$r3%02X")

        println()
        println(s"Stall count = $stallCount")
        println(s"WB count    = $wbCount")

        // ------------------------------------------------------
        // Verification
        // ------------------------------------------------------

        assert(
          stallCount == 1,
          s"Expected exactly one stall, got $stallCount"
        )

        assert(
          wbCount == 2,
          s"Expected exactly two write-backs, got $wbCount"
        )

        assert(
          wb1Address == 1 &&
            wb1Data == 0x55,
          f"Expected WB #1 = R1 <= 0x55, " +
            f"got R$wb1Address%d <= 0x$wb1Data%02X"
        )

        assert(
          wb2Address == 2 &&
            wb2Data == 0x65,
          f"Expected WB #2 = R2 <= 0x65, " +
            f"got R$wb2Address%d <= 0x$wb2Data%02X"
        )

        assert(
          r0 == 0x02,
          f"Expected R0 = 0x02, got 0x$r0%02X"
        )

        assert(
          r1 == 0x55,
          f"Expected R1 = 0x55, got 0x$r1%02X"
        )

        assert(
          r2 == 0x65,
          f"Expected R2 = 0x65, got 0x$r2%02X"
        )

        assert(
          r3 == 0x10,
          f"Expected R3 = 0x10, got 0x$r3%02X"
        )

        println()
        println("Load-use hazard / stall test passed.")
      }
  }
}
```

## 4. 이번 코드에서 반드시 확인해야 하는 핵심

특히 이전에 문제가 됐던 부분은 이제 다음과 같이 되어 있다.

### ALU

잘못된 코드:

```scala
exForwardA + exForwardB
```

`Bits + Bits` ❌

현재:

```scala
(exForwardA.asUInt + exForwardB.asUInt).asBits
```

`UInt + UInt` → `Bits` ✅

---

### LOAD 주소

```scala
val exLoadAddress =
  exForwardA.asUInt + idExOffset.asUInt
```

둘 다 `UInt`이므로 정상적인 산술 연산이다.

---

### Memory address

```scala
val dataMemAddress =
  exMemResult(3 downto 0).asUInt

val memReadData =
  dataMem.readAsync(dataMemAddress)
```

`16` word memory이므로 주소는 4 bit이다.

---

### 정확한 instruction

```text
LOAD R1,[R0+0]
```

계산:

```text
opcode = 5
rd     = 1
rs1    = 0
offset = 0

5 << 12 = 0x5000
1 << 10 = 0x0400

----------------
             0x5400
```

따라서:

```scala
B"16'h5400"
```

이다.

그리고:

```text
ADD R2,R1,R3

opcode = 0
rd     = 2 -> 0x0800
rs1    = 1 -> 0x0100
rs2    = 3 -> 0x00C0

----------------
             0x09C0
```

따라서:

```scala
B"16'h09C0"
```

이다.

---

## 5. 최종 pipeline 동작

이 테스트의 핵심은 이것이다.

```text
Cycle       1       2       3       4       5       6
---------------------------------------------------------
LOAD        IF      ID      EX      MEM     WB
ADD                 IF      ID      ID      EX      MEM
                            ↑       ↑
                            │       │
                         hazard   stall
```

Cycle 4에서 LOAD는 `MEM/WB`에 있고 `0x55`가 준비된다.

동시에 ADD가 EX에 들어가면서:

```text
MEM/WB
  R1 = 0x55
      │
      ▼
   Forwarding
      │
      ▼
ADD operand A = 0x55
```

가 된다.

따라서:

```text
0x55 + 0x10 = 0x65
```

이고 최종적으로:

```text
R0 = 0x02
R1 = 0x55
R2 = 0x65
R3 = 0x10
```

이 된다.

**여기까지가 Load-Use Hazard + 1-cycle Stall + Forwarding의 완성 형태다.**

---

---

좋습니다. 앞에서 중복되었던 forwarding 부분까지 제거하고, **Project 13-5의 최종본으로 사용할 `FiveStageLoadCpu.scala`**를 하나로 정리하겠습니다.

이번 버전의 기준은 다음과 같습니다.

- SpinalHDL **Release 34 / 1.15.0**
- 5-stage: `IF → ID → EX → MEM → WB`
- `LOAD`, `ADD`
- Load-use hazard 검출
- 정확히 1-cycle stall
- `EX/MEM → EX` forwarding
- `MEM/WB → EX` forwarding
- `Bits`에 산술 연산을 직접 적용하지 않음
- 정확한 instruction encoding
- pipeline register의 `valid` 관리
- `R0~R3` 초기화 인터페이스 포함
- debug용 `stall`, `write-back`, register 출력 포함

**실제 SpinalHDL 1.15.0 컴파일/시뮬레이션을 이 환경에서 실행한 것은 아니므로 “컴파일 통과”라고 주장하지는 않겠습니다.** 아래 코드는 앞서 발견된 타입/구조 오류를 제거한 최종 정리본입니다.

## `FiveStageLoadCpu.scala`

```scala
import spinal.core._

class FiveStageLoadCpu extends Component {

  val io = new Bundle {
    // ==========================================================
    // CPU control
    // ==========================================================

    val run = in Bool()

    // ==========================================================
    // Register initialization
    // ==========================================================

    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // ==========================================================
    // Debug outputs
    // ==========================================================

    val pc = out UInt(4 bits)

    val stall = out Bool()

    val wbEnable  = out Bool()
    val wbAddress = out UInt(2 bits)
    val wbData    = out Bits(8 bits)

    val reg0 = out Bits(8 bits)
    val reg1 = out Bits(8 bits)
    val reg2 = out Bits(8 bits)
    val reg3 = out Bits(8 bits)
  }

  // ==========================================================
  // Register File
  // ==========================================================

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  // ==========================================================
  // Instruction ROM
  //
  // R-type
  //
  // 15       12 11     10 9      8 7      6 5        0
  // +-----------+---------+--------+--------+----------+
  // |  opcode   |   rd    |  rs1   |  rs2   | reserved |
  // +-----------+---------+--------+--------+----------+
  //
  // LOAD
  //
  // 15       12 11     10 9      8 7            0
  // +-----------+---------+--------+--------------+
  // |  opcode   |   rd    |  rs1   |    offset    |
  // +-----------+---------+--------+--------------+
  //
  // opcode:
  //
  //   0000 = ADD
  //   0101 = LOAD
  //
  // Program:
  //
  //   PC 0 : LOAD R1, [R0 + 0]
  //   PC 1 : ADD  R2, R1, R3
  // ==========================================================

  val rom = Mem(
    Bits(16 bits),
    wordCount = 16
  )

  rom.init(
    Seq(
      B"16'h5400", // 0 : LOAD R1, [R0 + 0]
      B"16'h09C0", // 1 : ADD  R2, R1, R3

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

  // ==========================================================
  // Data RAM
  //
  // MEM[2] = 0x55
  // ==========================================================

  val dataMem = Mem(
    Bits(8 bits),
    wordCount = 16
  )

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

  // ==========================================================
  // Program Counter
  // ==========================================================

  val pcReg = Reg(UInt(4 bits)) init(0)

  val fetchInstruction = rom.readAsync(pcReg)

  // ==========================================================
  // IF / ID
  // ==========================================================

  val ifIdValid =
    Reg(Bool()) init(False)

  val ifIdInstruction =
    Reg(Bits(16 bits)) init(0)

  // ==========================================================
  // Decode
  // ==========================================================

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

  val decIsAdd =
    decOpcode === B"4'b0000"

  val decIsLoad =
    decOpcode === B"4'b0101"

  // Which source operands does this instruction actually use?
  //
  // LOAD:
  //   uses rs1
  //   does NOT use rs2
  //
  // ADD:
  //   uses rs1
  //   uses rs2

  val decUsesRs1 =
    decIsLoad || decIsAdd

  val decUsesRs2 =
    decIsAdd

  val decWriteEnable =
    decIsLoad || decIsAdd

  // ==========================================================
  // Register File read
  // ==========================================================

  val decRegA =
    registers(decRs1)

  val decRegB =
    registers(decRs2)

  // ==========================================================
  // ID / EX
  // ==========================================================

  val idExValid =
    Reg(Bool()) init(False)

  val idExIsLoad =
    Reg(Bool()) init(False)

  val idExWriteEnable =
    Reg(Bool()) init(False)

  val idExRd =
    Reg(UInt(2 bits)) init(0)

  val idExRs1 =
    Reg(UInt(2 bits)) init(0)

  val idExRs2 =
    Reg(UInt(2 bits)) init(0)

  val idExRegA =
    Reg(Bits(8 bits)) init(0)

  val idExRegB =
    Reg(Bits(8 bits)) init(0)

  val idExOffset =
    Reg(Bits(8 bits)) init(0)

  // ==========================================================
  // EX / MEM
  // ==========================================================

  val exMemValid =
    Reg(Bool()) init(False)

  val exMemIsLoad =
    Reg(Bool()) init(False)

  val exMemWriteEnable =
    Reg(Bool()) init(False)

  val exMemRd =
    Reg(UInt(2 bits)) init(0)

  val exMemResult =
    Reg(Bits(8 bits)) init(0)

  // ==========================================================
  // MEM / WB
  // ==========================================================

  val memWbValid =
    Reg(Bool()) init(False)

  val memWbIsLoad =
    Reg(Bool()) init(False)

  val memWbWriteEnable =
    Reg(Bool()) init(False)

  val memWbRd =
    Reg(UInt(2 bits)) init(0)

  val memWbResult =
    Reg(Bits(8 bits)) init(0)

  // ==========================================================
  // Forwarding
  //
  // Priority:
  //
  //   EX/MEM
  //       ↓
  //   MEM/WB
  //       ↓
  //   Register File
  //
  // Important:
  //
  // EX/MEM LOAD cannot be forwarded.
  //
  // At EX/MEM stage a LOAD only has the memory address.
  // The actual memory data is obtained in MEM stage.
  // ==========================================================

  val exForwardA =
    Bits(8 bits)

  val exForwardB =
    Bits(8 bits)

  // Default:
  // Register File value

  exForwardA := idExRegA
  exForwardB := idExRegB

  // ----------------------------------------------------------
  // Forward operand A : rs1
  // ----------------------------------------------------------

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs1)
  ) {
    exForwardA := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs1)
    ) {
      exForwardA := memWbResult
    }
  }

  // ----------------------------------------------------------
  // Forward operand B : rs2
  // ----------------------------------------------------------

  when(
    exMemValid &&
      exMemWriteEnable &&
      !exMemIsLoad &&
      (exMemRd === idExRs2)
  ) {
    exForwardB := exMemResult
  } otherwise {
    when(
      memWbValid &&
        memWbWriteEnable &&
        (memWbRd === idExRs2)
    ) {
      exForwardB := memWbResult
    }
  }

  // ==========================================================
  // EX stage
  // ==========================================================

  // exForwardA / exForwardB are Bits.
  //
  // Bits does not have '+',
  // so convert to UInt before arithmetic.

  val exAluResult =
    (
      exForwardA.asUInt +
        exForwardB.asUInt
    ).asBits

  // LOAD effective address:
  //
  // UInt + UInt

  val exLoadAddress =
    exForwardA.asUInt +
      idExOffset.asUInt

  // Result sent to EX/MEM.
  //
  // ADD:
  //   ALU result
  //
  // LOAD:
  //   effective address

  val exResult =
    Bits(8 bits)

  exResult := exAluResult

  when(idExIsLoad) {
    exResult := exLoadAddress.asBits
  }

  // ==========================================================
  // Load-use hazard detection
  //
  // Example:
  //
  //   LOAD R1, [R0 + 0]
  //   ADD  R2, R1, R3
  //
  // LOAD is currently in ID/EX.
  // ADD is currently in IF/ID.
  //
  // The LOAD result is not available for EX yet.
  // Therefore insert one bubble into ID/EX.
  // ==========================================================

  val loadUseHazard =
    ifIdValid &&
      idExValid &&
      idExIsLoad &&
      idExWriteEnable &&
      (
        (
          decUsesRs1 &&
            (idExRd === decRs1)
        ) ||
        (
          decUsesRs2 &&
            (idExRd === decRs2)
        )
      )

  // ==========================================================
  // MEM stage
  // ==========================================================

  // exMemResult contains:
  //
  //   ADD  -> ALU result
  //   LOAD -> effective memory address
  //
  // Data memory has 16 words,
  // therefore use address bits [3:0].

  val dataMemAddress =
    exMemResult(3 downto 0).asUInt

  val memReadData =
    dataMem.readAsync(dataMemAddress)

  // ==========================================================
  // Stall debug pulse
  // ==========================================================

  val stallPulse =
    Reg(Bool()) init(False)

  // ==========================================================
  // Sequential pipeline update
  // ==========================================================

  // ----------------------------------------------------------
  // Register initialization
  // ----------------------------------------------------------

  when(io.loadEnable) {
    registers(io.loadAddress) := io.loadData
  }

  // ----------------------------------------------------------
  // CPU running
  // ----------------------------------------------------------

  when(io.run) {

    // ========================================================
    // WB
    // ========================================================

    when(
      memWbValid &&
        memWbWriteEnable
    ) {
      registers(memWbRd) :=
        memWbResult
    }

    // ========================================================
    // MEM -> WB
    // ========================================================

    memWbValid :=
      exMemValid

    memWbIsLoad :=
      exMemIsLoad

    memWbWriteEnable :=
      exMemWriteEnable

    memWbRd :=
      exMemRd

    when(exMemIsLoad) {
      memWbResult :=
        memReadData
    } otherwise {
      memWbResult :=
        exMemResult
    }

    // ========================================================
    // EX -> MEM
    // ========================================================

    exMemValid :=
      idExValid

    exMemIsLoad :=
      idExIsLoad

    exMemWriteEnable :=
      idExWriteEnable

    exMemRd :=
      idExRd

    exMemResult :=
      exResult

    // ========================================================
    // Default: no stall
    // ========================================================

    stallPulse := False

    // ========================================================
    // Hazard
    // ========================================================

    when(loadUseHazard) {

      // ------------------------------------------------------
      // Insert bubble into ID/EX
      // ------------------------------------------------------

      idExValid :=
        False

      idExIsLoad :=
        False

      idExWriteEnable :=
        False

      idExRd :=
        0

      idExRs1 :=
        0

      idExRs2 :=
        0

      idExRegA :=
        0

      idExRegB :=
        0

      idExOffset :=
        0

      // ------------------------------------------------------
      // HOLD IF/ID
      // ------------------------------------------------------

      // No assignment is actually necessary here because
      // registers naturally retain their values.
      //
      // Explicitly assigning a register to itself is avoided.

      // ------------------------------------------------------
      // HOLD PC
      // ------------------------------------------------------

      // No assignment:
      // PC naturally retains its value.

      // ------------------------------------------------------
      // Debug
      // ------------------------------------------------------

      stallPulse := True

    } otherwise {

      // ======================================================
      // ID -> EX
      // ======================================================

      idExValid :=
        ifIdValid

      idExIsLoad :=
        ifIdValid &&
          decIsLoad

      idExWriteEnable :=
        ifIdValid &&
          decWriteEnable

      idExRd :=
        decRd

      idExRs1 :=
        decRs1

      idExRs2 :=
        decRs2

      idExRegA :=
        decRegA

      idExRegB :=
        decRegB

      idExOffset :=
        decOffset

      // ======================================================
      // IF -> ID
      // ======================================================

      ifIdValid :=
        True

      ifIdInstruction :=
        fetchInstruction

      // ======================================================
      // PC
      // ======================================================

      pcReg :=
        pcReg + 1
    }
  }

  // ==========================================================
  // Outputs
  // ==========================================================

  io.pc :=
    pcReg

  io.stall :=
    stallPulse

  io.wbEnable :=
    memWbValid &&
      memWbWriteEnable

  io.wbAddress :=
    memWbRd

  io.wbData :=
    memWbResult

  io.reg0 :=
    registers(0)

  io.reg1 :=
    registers(1)

  io.reg2 :=
    registers(2)

  io.reg3 :=
    registers(3)
}
```

### 핵심적으로 이번 최종본에서 정리된 부분

특히 이전 오류와 관련해서는 다음처럼 확정했다.

```scala
val exAluResult =
  (
    exForwardA.asUInt +
      exForwardB.asUInt
  ).asBits
```

즉 **`Bits + Bits`가 아니다.**

LOAD 주소도:

```scala
val exLoadAddress =
  exForwardA.asUInt +
    idExOffset.asUInt
```

로 **`UInt + UInt`**이다.

그리고 정확한 명령어는:

```scala
B"16'h5400"   // LOAD R1, [R0 + 0]
B"16'h09C0"   // ADD  R2, R1, R3
```

이다.

이제 이 CPU의 핵심 동작은

```text
LOAD R1
    ↓
   stall 1 cycle
    ↓
MEM/WB = R1 = 0x55
    ↓
forwarding
    ↓
ADD R2 = 0x55 + R3(0x10)
    ↓
R2 = 0x65
```

가 된다.

즉 이 단계에서 배우는 핵심은 **“LOAD는 EX/MEM에서 바로 forwarding할 수 없기 때문에 1-cycle stall을 넣고, 그 다음 MEM/WB에서 EX로 forwarding한다”**는 것이다.
