좋습니다. **12-6 — Instruction → Register File → ALU → Write Back**으로 갑니다.

이번 단계에서 드디어 **명령어 하나가 실제 연산을 수행**하게 만듭니다.

다만 앞 단계의 Decoder는 `rs1`까지만 추출했기 때문에, 이번에는 `rs2`까지 포함한 **R-type 명령어 형식**으로 확장하겠습니다.

## 12-6 R-Type Mini CPU

명령어를 다음처럼 정의합니다.

```text
15       12 11    10 9     8 7     6 5          0
+-----------+--------+-------+-------+------------+
|  opcode   |   rd   |  rs1  |  rs2  |  reserved |
+-----------+--------+-------+-------+------------+
   4 bits     2 bits  2 bits  2 bits    6 bits
```

예를 들어:

```text
ADD R2, R0, R1
```

은 의미상:

```text
R2 = R0 + R1
```

입니다.

---

# 1. Instruction Decoder 확장

```scala
import spinal.core._

class InstructionDecoder extends Component {
  val io = new Bundle {
    val instruction = in Bits(16 bits)

    val opcode = out UInt(2 bits)
    val rd     = out UInt(2 bits)
    val rs1    = out UInt(2 bits)
    val rs2    = out UInt(2 bits)

    val valid  = out Bool()
  }

  val opField = io.instruction(15 downto 12)

  io.opcode := opField(1 downto 0).asUInt
  io.rd     := io.instruction(11 downto 10).asUInt
  io.rs1    := io.instruction(9 downto 8).asUInt
  io.rs2    := io.instruction(7 downto 6).asUInt

  io.valid := False

  switch(opField) {
    is(B"4'b0000") { io.valid := True }
    is(B"4'b0001") { io.valid := True }
    is(B"4'b0010") { io.valid := True }
    is(B"4'b0011") { io.valid := True }
  }
}
```

이제 CPU가:

```text
rs1 ──┐
      ├──> ALU
rs2 ──┘
```

두 레지스터를 동시에 읽을 수 있습니다.

---

# 2. Mini CPU

이번에는 이전 `MiniCpuDatapath`에 Decoder를 연결합니다.

중요한 점은 **초기 레지스터 값을 넣는 경로와 CPU의 write-back 경로를 분리**하는 것입니다.

```scala
import spinal.core._

class MiniCpu extends Component {
  val io = new Bundle {
    val instruction = in Bits(16 bits)

    // 외부에서 Register File 초기화
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    val result = out Bits(8 bits)
  }

  val decoder = new InstructionDecoder
  val regFile = new DualReadRegisterFile
  val alu     = new SimpleAlu

  // --------------------------------------------------
  // Instruction Decode
  // --------------------------------------------------

  decoder.io.instruction := io.instruction

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
  // Register File Write
  //
  // loadEnable이 1이면 외부 초기화
  // valid 명령이면 ALU 결과를 write-back
  // --------------------------------------------------

  regFile.io.writeEnable := io.loadEnable | decoder.io.valid

  regFile.io.writeAddress := io.loadAddress
  regFile.io.writeData    := io.loadData

  when(decoder.io.valid) {
    regFile.io.writeAddress := decoder.io.rd
    regFile.io.writeData    := alu.io.result.asBits
  }
}
```

여기서 핵심은:

```text
                  instruction
                       │
                       ▼
                  ┌─────────┐
                  │ Decoder │
                  └────┬────┘
                       │
                 rs1   │   rs2
                  │    │    │
                  ▼    │    ▼
               ┌───────────────┐
               │  RegisterFile │
               │  A         B  │
               └──────┬───┬────┘
                      │   │
                      ▼   ▼
                    ┌───────┐
                    │  ALU  │
                    └───┬───┘
                        │
                     result
                        │
                        ▼
                       rd
```

입니다.

---

# 3. 명령어 인코딩

이번에는 실제 명령어를 하나 만들겠습니다.

```text
ADD R2, R0, R1
```

필드는:

```text
opcode = 0000
rd     = 10
rs1    = 00
rs2    = 01
```

따라서:

```text
0000 10 00 01 000000
```

이고 이것은:

```text
0x0810
```

입니다.

---

# 4. Simulation

초기화는 CPU의 외부 `load` 경로를 사용합니다.

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
      // ADD R2, R0, R1
      //
      // R2 = R0 + R1
      //    = 0x10 + 0x20
      //    = 0x30
      // ----------------------------------------------

      dut.io.instruction #= 0x0810

      // ALU 결과는 combinational
      sleep(1)

      println(
        f"ADD R2,R0,R1 -> result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30
      )

      // ----------------------------------------------
      // 명령어의 write-back
      // ----------------------------------------------

      dut.clockDomain.waitSampling()

      // ----------------------------------------------
      // R2가 실제로 0x30이 되었는지 확인
      //
      // OR R2, R0
      // 0x30 | 0x10 = 0x30
      //
      // OR 명령:
      // opcode=0011
      // rd=R2
      // rs1=R2
      // rs2=R0
      // ----------------------------------------------

      dut.io.instruction #= 0x3A00

      sleep(1)

      println(
        f"R2 OR R0 -> result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30
      )

      println("Mini CPU instruction execution test passed.")
    }
  }
}
```

### 한 가지 중요한 부분

이번 구조에서는:

```text
clock edge
    ↓
ADD 결과
    ↓
R2에 write
```

가 이루어집니다.

따라서 **ADD 결과를 확인하는 것**과 **R2에 결과가 저장되는 것**은 별개의 사건입니다.

```text
sleep(1)
    ↓
ALU 결과 확인

waitSampling()
    ↓
R2 write-back
```

이 구분이 이번 단계에서 아주 중요합니다.

예상 결과:

```text
ADD R2,R0,R1 -> result=0x30
R2 OR R0 -> result=0x30
Mini CPU instruction execution test passed.
```

이번 단계가 통과하면 다음은 **12-7 — Program Counter(PC) + Instruction Memory**입니다.

그때부터는 외부에서 매번 `instruction`을 넣는 것이 아니라,

```text
PC
 ↓
Instruction ROM
 ↓
Decoder
 ↓
Register File
 ↓
ALU
 ↓
Write Back
 ↓
PC + 1
```

구조로 발전시키면서 **작은 CPU가 스스로 명령어를 순차 실행**하도록 만들겠습니다.

---

좋습니다. **12-6까지 통과**했습니다. ✅

현재까지:

```text
12-1  Simple Bus + Address Decode       ✅
12-2  Register File                     ✅
12-3  Dual-Read Register File           ✅
12-4  Mini CPU Datapath                 ✅
12-5  Instruction Decoder               ✅
12-6  Instruction Execution             ✅
```

이제부터가 꽤 재미있는 부분입니다.

## 12-7 — Program Counter + Instruction ROM

지금까지는 CPU 외부에서 매번:

```text
instruction = 0x0810
```

처럼 명령어를 직접 넣었습니다.

다음부터는 CPU가 스스로:

```text
PC
 ↓
Instruction ROM
 ↓
Instruction
 ↓
Decoder
 ↓
Register File
 ↓
ALU
 ↓
Write Back
```

하도록 바꿉니다.

그리고 가장 먼저 **Program Counter(PC)**를 별도 블록으로 만들어 보겠습니다.

핵심은 단순합니다.

```text
reset → PC = 0

clock
  ↓
PC = PC + 1
```

그 다음 Instruction ROM을 붙여:

```text
PC = 0 → ADD ...
PC = 1 → SUB ...
PC = 2 → AND ...
PC = 3 → OR  ...
```

처럼 **프로그램을 순차적으로 실행하는 CPU**로 발전시키겠습니다.

다음 단계에서는 **PC + Instruction ROM만 먼저 만들고 시뮬레이션으로 검증**하겠습니다.  
한 번에 CPU 전체를 만들지 않고, 지금까지처럼 **각 블록을 확실하게 통과시키면서 연결**하겠습니다.

---

좋습니다. **12-7 — Program Counter + Instruction ROM**을 진행하겠습니다.

이번 단계는 CPU의 **Fetch 단계**입니다.

```text
        ┌─────────────┐
        │     PC      │
        └──────┬──────┘
               │ address
               ▼
        ┌─────────────┐
        │ Instruction │
        │     ROM     │
        └──────┬──────┘
               │
          instruction
```

이번에는 ALU나 Register File을 아직 연결하지 않고 **PC와 ROM을 각각 검증**합니다.

---

# 12-7-1 Program Counter

```scala
import spinal.core._

class ProgramCounter extends Component {
  val io = new Bundle {
    val enable = in Bool()
    val value  = out UInt(4 bits)
  }

  val pc = Reg(UInt(4 bits)) init(0)

  when(io.enable) {
    pc := pc + 1
  }

  io.value := pc
}
```

4-bit PC이므로:

```text
0 → 1 → 2 → ... → 15 → 0 → ...
```

으로 동작합니다.

### Simulation

```scala
import spinal.core.sim._

object ProgramCounterSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new ProgramCounter
    ).doSim { dut =>

      dut.clockDomain.forkStimulus(10)

      dut.io.enable #= false

      dut.clockDomain.waitSampling()

      assert(dut.io.value.toInt == 0)
      println(s"PC = ${dut.io.value.toInt}")

      // PC 0 -> 1
      dut.io.enable #= true
      dut.clockDomain.waitSampling()

      assert(dut.io.value.toInt == 1)
      println(s"PC = ${dut.io.value.toInt}")

      // PC 1 -> 2
      dut.clockDomain.waitSampling()

      assert(dut.io.value.toInt == 2)
      println(s"PC = ${dut.io.value.toInt}")

      // enable=false
      dut.io.enable #= false
      dut.clockDomain.waitSampling()

      assert(dut.io.value.toInt == 2)
      println(s"PC held = ${dut.io.value.toInt}")

      println("ProgramCounter test passed.")
    }
  }
}
```

여기서는 PC가 **레지스터**이므로 `waitSampling()`을 사용합니다.

---

# 12-7-2 Instruction ROM

이번에는 PC가 가리키는 위치에서 명령어를 가져오는 ROM입니다.

```scala
import spinal.core._

class InstructionRom extends Component {
  val io = new Bundle {
    val address     = in UInt(2 bits)
    val instruction = out Bits(16 bits)
  }

  val rom = Mem(Bits(16 bits), wordCount = 4)

  rom.init(
    Seq(
      B"16'x0810", // 0: ADD R2, R0, R1
      B"16'x1940", // 1: SUB R1, R1, R0
      B"16'x2D40", // 2: AND R3, R1, R1
      B"16'x3A00"  // 3: OR  R2, R2, R0
    )
  )

  io.instruction := rom.readAsync(io.address)
}
```

여기서 ROM은 **조합식 read**로 만들었습니다.

따라서:

```text
address 변경
     ↓
ROM data 변경
```

이고 별도의 clock edge가 필요하지 않습니다.

---

# Simulation

```scala
import spinal.core.sim._

object InstructionRomSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new InstructionRom
    ).doSim { dut =>

      for (address <- 0 until 4) {

        dut.io.address #= address

        sleep(1)

        val instruction = dut.io.instruction.toBigInt

        println(
          f"address=$address instruction=0x$instruction%04X"
        )

        address match {
          case 0 =>
            assert(instruction == 0x0810)

          case 1 =>
            assert(instruction == 0x1940)

          case 2 =>
            assert(instruction == 0x2D40)

          case 3 =>
            assert(instruction == 0x3A00)
        }
      }

      println("Instruction ROM test passed.")
    }
  }
}
```

예상 결과:

```text
address=0 instruction=0x0810
address=1 instruction=0x1940
address=2 instruction=0x2D40
address=3 instruction=0x3A00
Instruction ROM test passed.
```

## 이번 단계의 핵심

여기서 CPU의 **Fetch**가 처음 등장합니다.

```text
PC = 0
 ↓
ROM[0]
 ↓
0x0810
```

다음 clock:

```text
PC = 1
 ↓
ROM[1]
 ↓
0x1940
```

다음 clock:

```text
PC = 2
 ↓
ROM[2]
 ↓
0x2D40
```

즉 지금까지의 구조가:

```text
             ┌─────────────┐
             │     PC      │
             └──────┬──────┘
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
                    ▼
             Register + ALU
```

로 확장되고 있습니다.

**PC simulation과 Instruction ROM simulation이 둘 다 통과하면 다음은 12-8에서 이 둘을 연결한 `FetchUnit`을 만들고, 그 다음에 CPU 전체에 붙이겠습니다.**

---

좋습니다. **12-7 PC + Instruction ROM까지 통과**했습니다. ✅

현재 Project 12 진행 상황:

```text
12-1  Simple Bus + Address Decode       ✅
12-2  Register File                     ✅
12-3  Dual-Read Register File           ✅
12-4  Mini CPU Datapath                 ✅
12-5  Instruction Decoder               ✅
12-6  Instruction Execution             ✅
12-7  Program Counter + Instruction ROM  ✅
```

이제 **12-8 Fetch Unit**으로 갑니다.

이번에는 두 블록을 단순히 나열하는 것이 아니라:

```text
        ┌──────┐
        │  PC  │
        └──┬───┘
           │
           ▼
      Instruction ROM
           │
           ▼
      instruction
```

을 하나의 `FetchUnit`으로 묶겠습니다.

핵심은 **한 clock마다 PC가 증가하면서 다음 instruction을 가져오는 것**입니다.

그리고 여기서부터는 실제 CPU의 기본 동작인

```text
Fetch → Decode → Execute → Write Back
```

의 **Fetch 단계가 독립된 블록으로 완성**됩니다.

다음 단계에서 `FetchUnit` 코드와 시뮬레이션을 바로 진행하겠습니다.

---

좋습니다. **12-8 — Fetch Unit**으로 진행합니다.

이번 단계에서는 이미 검증한 `ProgramCounter`와 `InstructionRom`을 하나로 묶습니다.

## 12-8 Fetch Unit

구조는 아주 단순합니다.

```text
             ┌──────────────┐
   enable ──>│     PC       │
             └──────┬───────┘
                    │ address
                    ▼
             ┌──────────────┐
             │ Instruction  │
             │     ROM      │
             └──────┬───────┘
                    │
                    ▼
               instruction
```

### 코드

```scala
import spinal.core._

class FetchUnit extends Component {
  val io = new Bundle {
    val enable      = in Bool()
    val instruction = out Bits(16 bits)
    val pc          = out UInt(4 bits)
  }

  val pc = new ProgramCounter
  val rom = new InstructionRom

  pc.io.enable := io.enable

  rom.io.address := pc.io.value(1 downto 0)

  io.pc          := pc.io.value
  io.instruction := rom.io.instruction
}
```

여기서 한 가지 중요한 점:

`ProgramCounter`는 4-bit인데 `InstructionRom`은 4개 entry이므로 주소가 2-bit입니다.

따라서:

```scala
rom.io.address := pc.io.value(1 downto 0)
```

로 하위 2비트만 사용합니다.

즉:

```text
PC 0 → ROM 0
PC 1 → ROM 1
PC 2 → ROM 2
PC 3 → ROM 3
PC 4 → ROM 0
...
```

가 됩니다.

---

# Simulation

이번에는 **clock edge 전후의 관계**를 명확하게 확인하겠습니다.

```scala
import spinal.core.sim._

object FetchUnitSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new FetchUnit
    ).doSim { dut =>

      dut.clockDomain.forkStimulus(10)

      dut.io.enable #= true

      // 초기 PC = 0
      sleep(1)

      assert(dut.io.pc.toInt == 0)
      assert(dut.io.instruction.toBigInt == 0x0810)

      println(
        f"PC=${dut.io.pc.toInt} instruction=0x${dut.io.instruction.toBigInt}%04X"
      )

      // clock edge → PC 1
      dut.clockDomain.waitSampling()
      sleep(1)

      assert(dut.io.pc.toInt == 1)
      assert(dut.io.instruction.toBigInt == 0x1940)

      println(
        f"PC=${dut.io.pc.toInt} instruction=0x${dut.io.instruction.toBigInt}%04X"
      )

      // clock edge → PC 2
      dut.clockDomain.waitSampling()
      sleep(1)

      assert(dut.io.pc.toInt == 2)
      assert(dut.io.instruction.toBigInt == 0x2D40)

      println(
        f"PC=${dut.io.pc.toInt} instruction=0x${dut.io.instruction.toBigInt}%04X"
      )

      // clock edge → PC 3
      dut.clockDomain.waitSampling()
      sleep(1)

      assert(dut.io.pc.toInt == 3)
      assert(dut.io.instruction.toBigInt == 0x3A00)

      println(
        f"PC=${dut.io.pc.toInt} instruction=0x${dut.io.instruction.toBigInt}%04X"
      )

      println("FetchUnit test passed.")
    }
  }
}
```

예상 결과:

```text
PC=0 instruction=0x0810
PC=1 instruction=0x1940
PC=2 instruction=0x2D40
PC=3 instruction=0x3A00
FetchUnit test passed.
```

### 이번 단계에서 중요한 타이밍

```text
현재 PC = 0
ROM → instruction 0x0810

waitSampling()
        ↓
PC = 1
        ↓
ROM 주소 변경
        ↓
instruction = 0x1940
```

즉 **PC는 동기식**, **ROM read는 조합식**입니다.

이 조합이 실제 CPU의 instruction fetch에서 상당히 중요한 패턴입니다.

---

다음 **12-9**에서는 이 `FetchUnit`을 지금까지 만든 `InstructionDecoder + RegisterFile + ALU`에 연결해서,

```text
PC
 ↓
ROM
 ↓
Decoder
 ↓
Register File
 ↓
ALU
 ↓
Write Back
```

**Mini CPU 전체 datapath**를 완성하겠습니다.
