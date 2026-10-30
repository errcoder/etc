좋습니다. **Project 11 — ALU + Data Processor**로 바로 진행하겠습니다.  
이번에는 반복 설명 없이 **ALU → 시뮬레이션 → Data Processor** 순서로 갑니다.

# Project 11-1 — 8-bit ALU

4가지 연산만 구현합니다.

| opcode | 연산 |
|---|---|
| `00` | ADD |
| `01` | SUB |
| `10` | AND |
| `11` | OR |

구조는 아주 단순합니다.

```text
        ┌─────────────┐
 a ────→│             │
 b ────→│    ALU      │───→ result
opcode →│             │
        └─────────────┘
                 │
                 └────→ zero
```

### 1. ALU 코드

SpinalHDL **1.15.0** 기준입니다.

```scala
import spinal.core._

class SimpleAlu extends Component {
  val io = new Bundle {
    val a      = in UInt(8 bits)
    val b      = in UInt(8 bits)
    val opcode = in UInt(2 bits)

    val result = out UInt(8 bits)
    val zero   = out Bool()
  }

  io.result := 0

  switch(io.opcode) {
    is(0) {
      // ADD
      io.result := io.a + io.b
    }

    is(1) {
      // SUB
      io.result := io.a - io.b
    }

    is(2) {
      // AND
      io.result := io.a & io.b
    }

    is(3) {
      // OR
      io.result := io.a | io.b
    }
  }

  io.zero := io.result === 0
}
```

여기서 중요한 점 하나:

```scala
val result = out UInt(8 bits)
```

이므로 ALU 결과는 **8비트**입니다.

예를 들어

```text
0xFF + 0x01 = 0x100
```

이 경우 ALU의 `result`에는 하위 8비트인

```text
0x00
```

만 남습니다.

즉, 현재 버전에서는 **carry는 버립니다.**

나중에 CPU로 확장할 때 `carry`, `overflow`, `negative` 등의 flag를 추가할 수 있습니다.

---

# 2. ALU 시뮬레이션

ALU는 전부 **combinational logic**이므로 clock이 필요 없습니다.

이 부분이 RAM 시뮬레이션과 중요한 차이입니다.

```text
RAM
입력 → clock → 결과

ALU
입력 → combinational propagation → 결과
```

따라서 이번에는 `waitSampling()`이 아니라 `sleep()`을 사용합니다.

```scala
import spinal.core._
import spinal.core.sim._

object SimpleAluSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleAlu)
    .doSim { dut =>

      def test(
        name: String,
        a: Int,
        b: Int,
        opcode: Int,
        expected: Int
      ): Unit = {

        dut.io.a #= a
        dut.io.b #= b
        dut.io.opcode #= opcode

        // Combinational logic가 평가될 시간을 줌
        sleep(1)

        val result = dut.io.result.toBigInt.intValue
        val zero   = dut.io.zero.toBoolean

        println(
          f"$name%-5s " +
          f"a=0x$a%02X " +
          f"b=0x$b%02X " +
          f"result=0x$result%02X " +
          f"zero=$zero"
        )

        assert(
          result == expected,
          s"$name failed: expected=0x$expected%02X, actual=0x$result%02X"
        )

        assert(
          zero == (expected == 0),
          s"$name zero flag failed"
        )
      }

      test("ADD", 0x12, 0x34, 0, 0x46)
      test("SUB", 0x34, 0x12, 1, 0x22)
      test("AND", 0xF0, 0x0F, 2, 0x00)
      test("OR",  0xA0, 0x0F, 3, 0xAF)

      println("Simple ALU test passed.")
    }
}
```

예상 결과:

```text
ADD   a=0x12 b=0x34 result=0x46 zero=false
SUB   a=0x34 b=0x12 result=0x22 zero=false
AND   a=0xF0 b=0x0F result=0x00 zero=true
OR    a=0xA0 b=0x0F result=0xAF zero=false

Simple ALU test passed.
```

### 여기서 핵심

이번 simulation에서는:

```scala
dut.io.a #= ...
dut.io.b #= ...
dut.io.opcode #= ...

sleep(1)
```

입니다.

**clock이 없는 조합회로이므로 `waitSampling()`을 사용하면 안 됩니다.**

이 차이는 앞으로 CPU를 만들 때도 계속 중요합니다.

---

# Project 11-2 — Data Processor

이제 ALU에 **register 하나**를 붙여봅니다.

```text
             ┌─────────────┐
 dataA ─────→│             │
 dataB ─────→│     ALU     │
 opcode ────→│             │
             └──────┬──────┘
                    │
                    ▼
                ┌───────┐
       enable ─→│  REG  │
                └───┬───┘
                    │
                    ▼
                 result
```

즉,

> **ALU = 조합논리**  
> **Register = 순차논리**

를 하나의 작은 데이터패스로 묶는 것입니다.

다음 단계에서 이 `DataProcessor`를 구현하고 시뮬레이션하겠습니다.

---

좋습니다. **Project 11-2 — Data Processor**로 바로 갑니다.

이번에는 방금 만든 ALU 뒤에 **8-bit Register**를 붙입니다.

```text
                 combinational
              ┌───────────────┐
 dataA ──────→│               │
 dataB ──────→│     ALU       │
 opcode ─────→│               │
              └───────┬───────┘
                      │
                      ▼
                 ┌─────────┐
          en ───→│ Register│
                 └────┬────┘
                      │
                      ▼
                    result
```

핵심은 이것입니다.

- ALU → **조합논리**
- Register → **순차논리**
- `enable`이 `true`인 clock에서 ALU 결과를 저장
- `enable=false`이면 기존 결과 유지

## 1. DataProcessor

```scala
import spinal.core._

class DataProcessor extends Component {
  val io = new Bundle {
    val a      = in UInt(8 bits)
    val b      = in UInt(8 bits)
    val opcode = in UInt(2 bits)
    val enable = in Bool()

    val result = out UInt(8 bits)
  }

  val alu = new SimpleAlu

  alu.io.a      := io.a
  alu.io.b      := io.b
  alu.io.opcode := io.opcode

  val resultReg = Reg(UInt(8 bits)) init(0)

  when(io.enable) {
    resultReg := alu.io.result
  }

  io.result := resultReg
}
```

여기서 중요한 부분은:

```scala
when(io.enable) {
  resultReg := alu.io.result
}
```

입니다.

예를 들어:

```text
clock 1: enable = true  → ALU 결과 저장
clock 2: enable = false → 기존 값 유지
clock 3: enable = true  → 새로운 ALU 결과 저장
```

즉, 아주 작은 **datapath의 기본 형태**입니다.

---

# 2. Simulation

이번에는 clock이 필요합니다. Register가 있기 때문입니다.

```scala
import spinal.core._
import spinal.core.sim._

object DataProcessorSim extends App {
  SimConfig
    .withWave
    .compile(new DataProcessor)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // 초기값
      dut.io.a #= 0
      dut.io.b #= 0
      dut.io.opcode #= 0
      dut.io.enable #= false

      dut.clockDomain.waitSampling()

      assert(
        dut.io.result.toBigInt == 0,
        "Initial result must be 0"
      )

      // ADD: 0x10 + 0x20 = 0x30
      dut.io.a #= 0x10
      dut.io.b #= 0x20
      dut.io.opcode #= 0
      dut.io.enable #= true

      dut.clockDomain.waitSampling()

      println(
        f"After ADD: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(dut.io.result.toBigInt == 0x30)

      // 새로운 ALU 입력을 넣지만 enable=false
      // 따라서 register 값은 유지되어야 함
      dut.io.a #= 0xFF
      dut.io.b #= 0xFF
      dut.io.opcode #= 2       // AND → 0xFF
      dut.io.enable #= false

      dut.clockDomain.waitSampling()

      println(
        f"Enable=false: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30,
        "Result must remain 0x30 when enable=false"
      )

      // OR: 0xA0 | 0x0F = 0xAF
      dut.io.a #= 0xA0
      dut.io.b #= 0x0F
      dut.io.opcode #= 3
      dut.io.enable #= true

      dut.clockDomain.waitSampling()

      println(
        f"After OR: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(dut.io.result.toBigInt == 0xAF)

      println("DataProcessor test passed.")
    }
}
```

예상 결과:

```text
After ADD: result=0x30
Enable=false: result=0x30
After OR: result=0xAF
DataProcessor test passed.
```

### 이번 프로젝트에서 딱 3가지만 기억하면 됩니다

```text
ALU
 ↓
combinational
 ↓
Register
 ↓
sequential
```

그리고

```scala
when(io.enable) {
  resultReg := alu.io.result
}
```

은 **clock edge에서 enable이 1이면 값을 저장하고, 0이면 유지**한다는 뜻입니다.

이 테스트까지 통과하면 **Project 11은 완료**하고 바로 **Project 12 — Bus + Mini CPU**로 넘어가겠습니다.

---

---

좋습니다. 이건 **DataProcessor 하드웨어 코드의 문제가 아니라 simulation scheduling 문제**입니다.  
이번 경우는 이전에 UART/FIFO에서 경험했던 것과 같은 종류의 **stimulus → combinational logic → clock edge 순서**를 정확히 봐야 합니다.

현재 결과:

```text
After ADD: result=0x00
```

`0x30`이 아니라 이전 register 값 `0x00`이 저장됐습니다.

## 원인

현재 testbench가:

```scala
dut.io.a #= 0x10
dut.io.b #= 0x20
dut.io.opcode #= 0
dut.io.enable #= true

dut.clockDomain.waitSampling()
```

으로 되어 있습니다.

문제는 `a`, `b`, `opcode`를 변경한 직후 바로 clock edge를 기다리고 있다는 것입니다.

구조가:

```text
a/b/opcode
    │
    ▼
  ALU           ← combinational
    │
    ▼
 resultReg      ← clocked
```

이므로 원하는 순서는:

```text
입력 변경
   ↓
ALU combinational 평가
   ↓
ALU result = 0x30
   ↓
clock edge
   ↓
resultReg = 0x30
```

이어야 합니다.

따라서 **clock edge 전에 ALU가 평가될 시간을 명시적으로 줘야 합니다.**

---

# 수정된 Simulation

`waitSampling()` 전에 `sleep(1)`을 넣겠습니다.

```scala
import spinal.core._
import spinal.core.sim._

object DataProcessorSim extends App {
  SimConfig
    .withWave
    .compile(new DataProcessor)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // Initial
      dut.io.a #= 0
      dut.io.b #= 0
      dut.io.opcode #= 0
      dut.io.enable #= false

      dut.clockDomain.waitSampling()

      assert(
        dut.io.result.toBigInt == 0,
        "Initial result must be 0"
      )

      // --------------------------------------------------
      // ADD
      // 0x10 + 0x20 = 0x30
      // --------------------------------------------------

      dut.io.a #= 0x10
      dut.io.b #= 0x20
      dut.io.opcode #= 0
      dut.io.enable #= true

      // ALU가 새 입력을 계산할 시간을 줌
      sleep(1)

      println(
        f"Before ADD clock: ALU result=0x${dut.alu.io.result.toBigInt}%02X"
      )

      dut.clockDomain.waitSampling()

      println(
        f"After ADD: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30,
        "ADD result must be 0x30"
      )

      // --------------------------------------------------
      // enable = false
      // --------------------------------------------------

      dut.io.a #= 0xFF
      dut.io.b #= 0xFF
      dut.io.opcode #= 2       // AND
      dut.io.enable #= false

      sleep(1)

      println(
        f"Before disabled clock: ALU result=0x${dut.alu.io.result.toBigInt}%02X"
      )

      dut.clockDomain.waitSampling()

      println(
        f"Enable=false: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0x30,
        "Result must remain 0x30 when enable=false"
      )

      // --------------------------------------------------
      // OR
      // 0xA0 | 0x0F = 0xAF
      // --------------------------------------------------

      dut.io.a #= 0xA0
      dut.io.b #= 0x0F
      dut.io.opcode #= 3
      dut.io.enable #= true

      sleep(1)

      println(
        f"Before OR clock: ALU result=0x${dut.alu.io.result.toBigInt}%02X"
      )

      dut.clockDomain.waitSampling()

      println(
        f"After OR: result=0x${dut.io.result.toBigInt}%02X"
      )

      assert(
        dut.io.result.toBigInt == 0xAF,
        "OR result must be 0xAF"
      )

      println("DataProcessor test passed.")
    }
}
```

### 이번에 확인할 포인트

특히 이 로그를 추가했습니다.

```scala
println(
  f"Before ADD clock: ALU result=0x${dut.alu.io.result.toBigInt}%02X"
)
```

정상이라면:

```text
Before ADD clock: ALU result=0x30
After ADD: result=0x30
```

가 나와야 합니다.

즉,

```text
ALU result
   ↓
0x30
   ↓ clock edge
Register
   ↓
0x30
```

입니다.

그리고 이 문제는 **아주 좋은 학습 포인트**입니다.

> `waitSampling()`은 단순히 "값이 계산될 때까지 기다리는 것"이 아니라 **clock edge를 기준으로 DUT의 순차논리가 동작하도록 하는 것**입니다.

이번에는 `sleep(1)`로 combinational ALU의 평가와 clock edge를 분리해 주는 것이 핵심입니다.

수정 후 결과를 보여주시면, 통과하면 **Project 11 완료 → Project 12 Bus + Mini CPU**로 바로 넘어가겠습니다.

---

---


