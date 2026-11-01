좋습니다. **Dual-Read Register File까지 정상 통과**했습니다. ✅

현재 Project 12 진행 상황:

```text
12-1 Simple Bus              ✓
12-2 Register File           ✓
12-3 Dual-Read RegisterFile  ✓
```

이제 드디어 **ALU + Register File을 연결**합니다.

# Project 12-4 — Mini CPU Datapath

목표는 아주 간단합니다.

```text
R2 = R0 + R1
```

하드웨어는:

```text
       ┌───────────────┐
R0 ───→│               │
       │ Register File │──→ A
R1 ───→│               │──→ B
       └───────────────┘
                 │
                 ▼
              ┌─────┐
       ADD ──→│ ALU │
              └──┬──┘
                 │
                 ▼
                R2
```

## MiniCpuDatapath

```scala id="t7g3wq"
import spinal.core._

class MiniCpuDatapath extends Component {
  val io = new Bundle {
    val readAddressA = in UInt(2 bits)
    val readAddressB = in UInt(2 bits)

    val writeEnable  = in Bool()
    val writeAddress = in UInt(2 bits)
    val writeData    = in Bits(8 bits)

    val opcode       = in UInt(2 bits)

    val result       = out Bits(8 bits)
  }

  val regFile = new DualReadRegisterFile
  val alu      = new SimpleAlu

  // Register File
  regFile.io.readAddressA := io.readAddressA
  regFile.io.readAddressB := io.readAddressB

  regFile.io.writeEnable  := io.writeEnable
  regFile.io.writeAddress := io.writeAddress
  regFile.io.writeData    := io.writeData

  // ALU operands
  alu.io.a := regFile.io.readDataA.asUInt
  alu.io.b := regFile.io.readDataB.asUInt
  alu.io.opcode := io.opcode

  // ALU result
  io.result := alu.io.result.asBits
}
```

여기서 중요한 것은 **ALU 결과를 바로 register에 저장하지 않고 외부 `writeData` 경로로 되돌릴 수 있도록 만든 것**입니다.

즉 testbench에서:

```text
R0 = 0x10
R1 = 0x20
```

을 만들어 놓고,

```text
readAddressA = 0
readAddressB = 1
opcode = ADD
```

를 설정하면:

```text
ALU result = 0x30
```

이 됩니다.

그리고 이 값을:

```text
writeAddress = 2
writeData = ALU result
writeEnable = true
```

로 다음 clock에 저장하면:

```text
R2 = 0x30
```

이 됩니다.

### 전체 데이터 흐름

```text
R0 ──┐
     │
     ▼
    ALU ─────────→ writeData
     ▲                  │
     │                  ▼
R1 ──┘                R2
```

이제 다음 단계에서 이 구조를 **한 번의 ADD instruction처럼 동작시키는 simulation**을 작성해서

```text
R0 = 0x10
R1 = 0x20
R2 = 0x30
```

을 실제로 검증하겠습니다.

---

---

좋습니다. **Project 12-4 Mini CPU Datapath 시뮬레이션**으로 진행하겠습니다.

이번에는 이전에 문제가 있었던 것처럼 시뮬레이션 스케줄링을 애매하게 하지 않고, **동기식 Register File 쓰기와 조합식 ALU 계산을 분리해서** 확인하겠습니다.

## 12-4 Mini CPU Datapath Simulation

현재 구조에서는:

```text
R0 ──┐
     ├──> ALU ──> result
R1 ──┘          │
                │
                └──> writeData ──> R2
```

즉,

```text
R2 = R0 + R1
```

을 수행합니다.

### Simulation

```scala
import spinal.core._
import spinal.core.sim._

object MiniCpuDatapathSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new MiniCpuDatapath
    ).doSim { dut =>

      dut.clockDomain.forkStimulus(10)

      // --------------------------------------------------
      // 1. R0 = 0x10
      // --------------------------------------------------

      dut.io.writeEnable  #= true
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0x10

      // Register write는 clock edge에서 발생
      dut.clockDomain.waitSampling()

      dut.io.writeEnable #= false

      // --------------------------------------------------
      // 2. R1 = 0x20
      // --------------------------------------------------

      dut.io.writeEnable  #= true
      dut.io.writeAddress #= 1
      dut.io.writeData    #= 0x20

      dut.clockDomain.waitSampling()

      dut.io.writeEnable #= false

      // --------------------------------------------------
      // 3. R0 + R1
      // --------------------------------------------------

      dut.io.readAddressA #= 0
      dut.io.readAddressB #= 1
      dut.io.opcode       #= 0    // ADD

      // ALU는 combinational이므로 clock edge가 필요하지 않음
      sleep(1)

      println(
        f"ADD: R0 + R1 = 0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30,
        "ALU result should be 0x30"
      )

      // --------------------------------------------------
      // 4. ALU 결과 0x30을 R2에 write
      // --------------------------------------------------

      dut.io.writeEnable  #= true
      dut.io.writeAddress #= 2
      dut.io.writeData    #= dut.io.result.toBigInt

      dut.clockDomain.waitSampling()

      dut.io.writeEnable #= false

      // --------------------------------------------------
      // 5. R2 확인
      //
      // R2 = 0x30
      // R0 = 0x10
      //
      // R2 OR R0 = 0x30
      // --------------------------------------------------

      dut.io.readAddressA #= 2
      dut.io.readAddressB #= 0
      dut.io.opcode       #= 3    // OR

      sleep(1)

      println(
        f"R2 OR R0 = 0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30,
        "R2 should contain 0x30"
      )

      println("Mini CPU Datapath test passed.")
    }
  }
}
```

### 예상 결과

```text
ADD: R0 + R1 = 0x30
R2 OR R0 = 0x30
Mini CPU Datapath test passed.
```

여기서 중요한 것은 **`sleep(1)`과 `waitSampling()`의 역할을 섞지 않는 것**입니다.

| 동작 | 사용 |
|---|---|
| R0/R1/R2에 Register write | `waitSampling()` |
| Register read → ALU 계산 확인 | `sleep(1)` |
| ALU 결과 자체 | combinational |
| Register에 실제 저장 | clock edge 필요 |

그리고 현재 `MiniCpuDatapath`는 **ALU 결과를 내부에서 자동으로 write-back하는 CPU 구조는 아닙니다.** 테스트벤치에서 `writeData`에 ALU 결과를 넣어주는 형태입니다.

따라서 이 실습의 다음 단계는 자연스럽게 **12-5 Instruction/Control Unit**으로 가서,

```text
Instruction
    ↓
Decoder / Control
    ↓
Register File
    ↓
ALU
    ↓
Write Back
```

구조를 만들어 **진짜 Mini CPU 형태**로 연결하는 것입니다.

---

완벽합니다. **MiniCpuDatapath 시뮬레이션까지 통과**했습니다. ✅

```text
ADD: R0 + R1 = 0x30
R2 OR R0 = 0x30
Mini CPU Datapath test passed.
```

### Project 12 현재 진행 상황

| 단계 | 내용 | 상태 |
|---|---|---|
| 12-1 | Simple Bus + Address Decode | ✅ |
| 12-2 | Register File | ✅ |
| 12-3 | Dual-Read Register File | ✅ |
| 12-4 | Mini CPU Datapath | ✅ |

이제 **Project 12의 핵심 부품들이 모두 준비**됐습니다.

다음은 **12-5 Control Unit + Instruction Decoder**로 넘어가겠습니다.

목표는 직접 신호를 하나씩 넣는 방식에서 벗어나:

```text
Instruction
    ↓
Instruction Decoder
    ↓
Control Signals
    ↓
Register File + ALU
    ↓
Write Back
```

형태로 발전시키는 것입니다.

그리고 여기서부터는 **Mini CPU가 실제 명령어를 실행하는 구조**를 만들어 보겠습니다.

---

---

좋습니다. 이제 **Project 12-5 — Instruction Decoder + Control Unit**으로 넘어갑니다.

이번 단계에서는 CPU 전체를 한 번에 만들지 않고, **명령어 비트 → 제어 신호**라는 핵심 개념부터 잡겠습니다.

# 12-5 Instruction Decoder

간단한 16-bit 명령어 형식을 정의하겠습니다.

```text
15          12 11       10 9        8 7              0
+-------------+-----------+----------+----------------+
|   opcode    |   rd      |   rs1    |      rs2       |
+-------------+-----------+----------+----------------+
    4 bits       2 bits      2 bits       8 bits
```

이번에는 실제로 사용할 부분을 단순화해서:

```text
opcode = [15:12]
rd     = [11:10]
rs1    = [9:8]
```

로 하겠습니다.

명령어는 우선 4개만 사용합니다.

| opcode | 명령 |
|---|---|
| `0000` | ADD |
| `0001` | SUB |
| `0010` | AND |
| `0011` | OR |

## InstructionDecoder

```scala
import spinal.core._

class InstructionDecoder extends Component {
  val io = new Bundle {
    val instruction = in Bits(16 bits)

    val opcode = out UInt(2 bits)
    val rd     = out UInt(2 bits)
    val rs1    = out UInt(2 bits)

    val valid  = out Bool()
  }

  val opField = io.instruction(15 downto 12)

  io.opcode := opField(1 downto 0).asUInt
  io.rd     := io.instruction(11 downto 10).asUInt
  io.rs1    := io.instruction(9 downto 8).asUInt

  io.valid := False

  switch(opField) {
    is(B"4'b0000") { io.valid := True }
    is(B"4'b0001") { io.valid := True }
    is(B"4'b0010") { io.valid := True }
    is(B"4'b0011") { io.valid := True }
  }
}
```

여기서 중요한 점 하나가 있습니다.

`opcode` 자체는 2-bit ALU opcode로 사용하지만, 명령어 전체의 opcode field는 4-bit입니다.

즉:

```text
instruction opcode
0000 ──┐
0001   │
0010   ├──> valid 명령인지 판정
0011 ──┘
```

그리고 하위 2비트를 ALU opcode로 그대로 사용할 수 있습니다.

---

# Simulation

이번 테스트는 **clock이 필요 없는 조합 논리 테스트**입니다.

따라서 `waitSampling()`을 사용할 필요가 없습니다.

```scala
import spinal.core.sim._

object InstructionDecoderSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new InstructionDecoder
    ).doSim { dut =>

      def test(
        instruction: Int,
        expectedOpcode: Int,
        expectedRd: Int,
        expectedRs1: Int
      ): Unit = {

        dut.io.instruction #= instruction

        // 조합 논리 propagation
        sleep(1)

        println(
          f"instruction=0x$instruction%04X " +
          f"opcode=${dut.io.opcode.toInt} " +
          f"rd=${dut.io.rd.toInt} " +
          f"rs1=${dut.io.rs1.toInt} " +
          f"valid=${dut.io.valid.toBoolean}"
        )

        assert(dut.io.opcode.toInt == expectedOpcode)
        assert(dut.io.rd.toInt == expectedRd)
        assert(dut.io.rs1.toInt == expectedRs1)
        assert(dut.io.valid.toBoolean)
      }

      // ADD
      // opcode = 0000
      // rd     = 01
      // rs1    = 10
      test(
        instruction = 0x0600,
        expectedOpcode = 0,
        expectedRd = 1,
        expectedRs1 = 2
      )

      // SUB
      // opcode = 0001
      // rd     = 10
      // rs1    = 01
      test(
        instruction = 0x1900,
        expectedOpcode = 1,
        expectedRd = 2,
        expectedRs1 = 1
      )

      // AND
      // opcode = 0010
      // rd     = 00
      // rs1    = 11
      test(
        instruction = 0x2B00,
        expectedOpcode = 2,
        expectedRd = 0,
        expectedRs1 = 3
      )

      // OR
      // opcode = 0011
      // rd     = 11
      // rs1    = 00
      test(
        instruction = 0x3C00,
        expectedOpcode = 3,
        expectedRd = 3,
        expectedRs1 = 0
      )

      println("Instruction Decoder test passed.")
    }
  }
}
```

예상 결과:

```text
instruction=0x0600 opcode=0 rd=1 rs1=2 valid=true
instruction=0x1900 opcode=1 rd=2 rs1=1 valid=true
instruction=0x2B00 opcode=2 rd=0 rs1=3 valid=true
instruction=0x3C00 opcode=3 rd=3 rs1=0 valid=true
Instruction Decoder test passed.
```

### 이번 단계에서 기억할 핵심

```text
Bits(16 bits)
      │
      ├── [15:12] → Instruction type
      │
      ├── [11:10] → Destination register
      │
      └── [9:8]   → Source register
```

즉 **Instruction Decoder는 계산을 하는 블록이 아니라 명령어를 해석하는 블록**입니다.

다음 **12-6**에서는 이 Decoder를 앞에서 만든 `DualReadRegisterFile + ALU`와 연결해서,

```text
instruction
     ↓
decoder
     ↓
R0 + R1
     ↓
ALU
     ↓
rd
```

처럼 **명령어 하나가 실제로 Register → ALU → Register 흐름을 만드는 Mini CPU**로 연결하겠습니다.

---

맞습니다. **이번 실패는 DUT 문제가 아니라 제가 작성한 테스트 벡터의 `rd` 기대값을 잘못 계산한 것입니다.**  
이건 제가 미리 비트 필드를 직접 검산했어야 했습니다.

실제 출력:

```text
instruction=0x2B00 opcode=2 rd=2 rs1=3 valid=true
```

이 값이 **정상입니다.**

### 왜 `rd=2`인가?

명령어:

```text
0x2B00
```

을 16비트로 보면:

```text
0010 1011 0000 0000
^^^^ ^^^^
15:12 11:8
```

필드를 나누면:

```text
[15:12] = 0010 = opcode field
[11:10] = 10   = rd = 2
[9:8]   = 11   = rs1 = 3
```

따라서:

```text
opcode = 2
rd     = 2
rs1    = 3
```

입니다.

제가 테스트 코드에서:

```scala
expectedRd = 0
```

이라고 잘못 넣었습니다.

---

## 테스트 코드 수정

세 번째 테스트만:

```scala
test(
  instruction = 0x2B00,
  expectedOpcode = 2,
  expectedRd = 2,       // 수정
  expectedRs1 = 3
)
```

로 바꾸면 됩니다.

그리고 네 번째 `0x3C00`은:

```text
0011 1100 0000 0000
     ^^ ^^
     rd rs1
```

이므로:

```text
opcode = 3
rd     = 3
rs1    = 0
```

으로 현재 테스트가 맞습니다.

### 따라서 예상 결과

```text
instruction=0x0600 opcode=0 rd=1 rs1=2 valid=true
instruction=0x1900 opcode=1 rd=2 rs1=1 valid=true
instruction=0x2B00 opcode=2 rd=2 rs1=3 valid=true
instruction=0x3C00 opcode=3 rd=3 rs1=0 valid=true
Instruction Decoder test passed.
```

**Decoder 자체는 수정할 필요 없습니다.**

이번에는 특히 `0x2B00`의 비트 필드를 제가 잘못 계산한 것이므로, 다음 단계부터는 **명령어 인코딩과 expected 값도 함께 검산해서** 진행하겠습니다.

---

