# 다음 단계 — `ValidReadyFifo` 직접 구현

이번에는 지금까지 만든 `SimpleFifo`를 **`valid / ready` 방식의 FIFO**로 발전시키겠습니다.

목표는 아주 명확합니다.

```text
Producer                         FIFO

valid ─────────────────────────► valid
data  ─────────────────────────► data

ready ◄───────────────────────── ready
```

그리고:

```text
writeFire = valid && ready
```

일 때만 FIFO에 데이터가 들어갑니다.

반대쪽에서는:

```text
readFire = valid && ready
```

일 때만 FIFO에서 데이터가 나갑니다.

---

# 1. FIFO 인터페이스

이번 FIFO는 다음처럼 만들겠습니다.

```text
Input side

  valid ─────►
  data  ─────►
  ready ◄─────


Output side

  valid ◄─────
  data  ◄─────
  ready ─────►
```

즉:

```scala
val inputValid = in Bool()
val inputData  = in Bits(8 bits)
val inputReady = out Bool()

val outputValid = out Bool()
val outputData  = out Bits(8 bits)
val outputReady = in Bool()
```

---

# 2. 핵심 동작

FIFO가 비어 있지 않으면:

```scala
outputValid := !empty
```

FIFO가 FULL이 아니면:

```scala
inputReady := !full
```

따라서:

```scala
val push = inputValid && inputReady
val pop  = outputValid && outputReady
```

입니다.

이것이 기존 FIFO의 `write/read`보다 조금 더 명확한 구조입니다.

---

# 3. `ValidReadyFifo`

SpinalHDL **1.15.0** 기준으로 작성하겠습니다.

```scala
import spinal.core._

case class ValidReadyFifo(
    dataWidth: Int,
    depth: Int
) extends Component {

  require(dataWidth >= 1)
  require(depth >= 2)

  val io = new Bundle {

    // ------------------------------------------------------------
    // Input side
    // ------------------------------------------------------------

    val inputValid = in Bool()
    val inputData  = in Bits(dataWidth bits)
    val inputReady = out Bool()

    // ------------------------------------------------------------
    // Output side
    // ------------------------------------------------------------

    val outputValid = out Bool()
    val outputData  = out Bits(dataWidth bits)
    val outputReady = in Bool()
  }

  // ==============================================================
  // Storage
  // ==============================================================

  val mem = Vec(
    Reg(Bits(dataWidth bits)) init(0),
    depth
  )

  // ==============================================================
  // Pointers
  // ==============================================================

  val ptrWidth =
    log2Up(depth)

  val writePtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val readPtr =
    Reg(UInt(ptrWidth bits)) init(0)

  // ==============================================================
  // Occupancy
  // ==============================================================

  val countWidth =
    log2Up(depth + 1)

  val count =
    Reg(UInt(countWidth bits)) init(0)

  // ==============================================================
  // Status
  // ==============================================================

  val empty =
    count === 0

  val full =
    count === depth

  // ==============================================================
  // Input handshake
  // ==============================================================

  io.inputReady :=
    !full

  // ==============================================================
  // Output handshake
  // ==============================================================

  io.outputValid :=
    !empty

  io.outputData :=
    mem(readPtr)

  // ==============================================================
  // Transfer events
  // ==============================================================

  val push =
    io.inputValid && io.inputReady

  val pop =
    io.outputValid && io.outputReady

  // ==============================================================
  // Write
  // ==============================================================

  when(push) {

    mem(writePtr) :=
      io.inputData

    when(writePtr === depth - 1) {
      writePtr := 0
    } otherwise {
      writePtr := writePtr + 1
    }
  }

  // ==============================================================
  // Read
  // ==============================================================

  when(pop) {

    when(readPtr === depth - 1) {
      readPtr := 0
    } otherwise {
      readPtr := readPtr + 1
    }
  }

  // ==============================================================
  // Occupancy counter
  // ==============================================================

  when(push && !pop) {

    count :=
      count + 1

  } elsewhen(!push && pop) {

    count :=
      count - 1
  }
}
```

---

# 4. 기존 `SimpleFifo`와 비교

기존에는:

```text
write
writeData
read
readData
full
empty
```

였습니다.

새로운 FIFO는:

```text
inputValid
inputData
inputReady

outputValid
outputData
outputReady
```

입니다.

개념적으로는:

| `SimpleFifo` | `ValidReadyFifo` |
|---|---|
| `write` | `inputValid` |
| `writeData` | `inputData` |
| `!full` | `inputReady` |
| `read` | `outputReady` |
| `readData` | `outputData` |
| `!empty` | `outputValid` |

특히 중요한 것은 `full/empty`가 외부 제어 신호에서 사라졌다는 점입니다.

외부에서는 단순히:

```text
"보낼 데이터가 있다"
        ↓
valid

"받을 수 있다"
        ↓
ready
```

만 보면 됩니다.

---

# 5. 가장 중요한 상황: `valid=1, ready=0`

이것을 반드시 테스트해야 합니다.

FIFO가 FULL이라고 해봅시다.

```text
FIFO

┌─────┐
│  A  │
├─────┤
│  B  │
├─────┤
│  C  │
├─────┤
│  D  │
└─────┘

full = true
```

그러면:

```text
inputReady = false
```

입니다.

Producer가:

```text
inputValid = true
inputData  = E
```

를 내보내도:

```text
inputValid = 1
inputReady = 0
```

이므로:

```scala
push = inputValid && inputReady
     = true && false
     = false
```

입니다.

따라서 **E는 FIFO에 들어가지 않습니다.**

---

# 6. 그런데 Producer는 E를 버리면 안 된다

여기가 `valid/ready`의 핵심입니다.

Producer가:

```text
valid = 1
data  = E
```

를 내보냈는데:

```text
ready = 0
```

이면 Producer는 다음 cycle에 갑자기:

```text
data = F
```

로 바꾸면 안 됩니다.

계속:

```text
valid = 1
data  = E
```

를 유지해야 합니다.

그러다가 FIFO가 공간을 만들면:

```text
valid = 1
ready = 1
data  = E
```

가 되고 그 순간:

```text
fire = 1
```

입니다.

즉:

```text
         FIFO FULL
             │
             ▼
Producer ── E ──────────►
           valid=1
           
FIFO
ready=0

       ... wait ...

FIFO에서 하나 제거
             │
             ▼
ready=1

Producer ── E ──────────►
           valid=1
           ready=1

             ↓
           FIRE
```

이것이 **backpressure**입니다.

---

# 7. 첫 번째 시뮬레이션

이번에는 FIFO에:

```text
A
B
C
D
```

를 넣어서 FULL로 만든 뒤,

```text
E
```

를 `valid=1` 상태로 유지하겠습니다.

그리고 FIFO가 E를 받을 수 없다는 것을 확인합니다.

그 다음 하나를 읽으면 FIFO가 공간을 만들고,

```text
E
```

가 정상적으로 들어오는 것을 확인합니다.

이 테스트가 굉장히 중요합니다.

```scala
import spinal.core._
import spinal.core.sim._

object ValidReadyFifoSim extends App {

  val depth = 4

  SimConfig
    .withWave
    .doSim(
      ValidReadyFifo(
        dataWidth = 8,
        depth = depth
      )
    ) { dut =>

      dut.clockDomain.forkStimulus(10)

      dut.io.inputValid  #= false
      dut.io.inputData   #= 0
      dut.io.outputReady #= false

      dut.clockDomain.waitSampling(2)

      // ----------------------------------------------------------
      // Write one item
      // ----------------------------------------------------------

      def push(data: Int): Unit = {

        dut.io.inputData  #= data
        dut.io.inputValid #= true

        // ready가 될 때까지 기다린다.
        while (!dut.io.inputReady.toBoolean) {
          dut.clockDomain.waitSampling()
        }

        // valid && ready인 sampling edge에서 전송
        dut.clockDomain.waitSampling()

        dut.io.inputValid #= false
      }

      // ----------------------------------------------------------
      // Read one item
      // ----------------------------------------------------------

      def pop(expected: Int): Unit = {

        assert(
          dut.io.outputValid.toBoolean,
          "FIFO should contain data"
        )

        val actual =
          dut.io.outputData.toBigInt.intValue

        println(
          f"[POP] expected=0x$expected%02X actual=0x$actual%02X"
        )

        assert(
          actual == expected,
          f"Expected 0x$expected%02X but got 0x$actual%02X"
        )

        dut.io.outputReady #= true

        dut.clockDomain.waitSampling()

        dut.io.outputReady #= false

        dut.clockDomain.waitFallingEdge()
      }

      // ==========================================================
      // TEST 1
      // A/B/C/D 넣기
      // ==========================================================

      println("[TEST] Fill FIFO")

      push(0x41)
      push(0x42)
      push(0x43)
      push(0x44)

      dut.clockDomain.waitFallingEdge()

      println(
        s"[FULL] inputReady=${dut.io.inputReady.toBoolean} " +
        s"outputValid=${dut.io.outputValid.toBoolean} " +
        f"outputData=0x${dut.io.outputData.toBigInt}%02X"
      )

      assert(
        !dut.io.inputReady.toBoolean,
        "FIFO should not be ready when full"
      )

      assert(
        dut.io.outputValid.toBoolean,
        "FIFO should have valid output when full"
      )

      assert(
        dut.io.outputData.toBigInt == 0x41,
        "FIFO head should be A"
      )

      println("[PASS] FIFO is FULL")

      // ==========================================================
      // TEST 2
      // E를 valid 상태로 유지
      // ==========================================================

      println("[TEST] Hold E while FIFO is FULL")

      dut.io.inputData  #= 0x45
      dut.io.inputValid #= true

      // FIFO가 FULL이므로 ready는 0이어야 한다.
      dut.clockDomain.waitSampling(3)

      assert(
        dut.io.inputValid.toBoolean,
        "Producer must keep valid asserted"
      )

      assert(
        !dut.io.inputReady.toBoolean,
        "FIFO must remain not-ready while full"
      )

      println(
        f"[HOLD] valid=${dut.io.inputValid.toBoolean} " +
        s"ready=${dut.io.inputReady.toBoolean} " +
        f"data=0x${dut.io.inputData.toBigInt}%02X"
      )

      println("[PASS] E is being held while FIFO is FULL")

      // ==========================================================
      // TEST 3
      // FIFO에서 A를 제거
      // ==========================================================

      dut.io.outputReady #= true

      dut.clockDomain.waitSampling()

      dut.io.outputReady #= false

      // E는 아직 valid 상태
      assert(
        dut.io.inputValid.toBoolean,
        "E must still be valid"
      )

      // FIFO에 공간이 생겼다.
      assert(
        dut.io.inputReady.toBoolean,
        "FIFO should become ready after one pop"
      )

      println(
        s"[SPACE] inputReady=${dut.io.inputReady.toBoolean}"
      )

      // ==========================================================
      // TEST 4
      // E 전송
      // ==========================================================

      dut.clockDomain.waitSampling()

      dut.io.inputValid #= false

      dut.clockDomain.waitFallingEdge()

      println("[PASS] E entered FIFO")

      // ==========================================================
      // FIFO 내용 확인
      // ==========================================================

      pop(0x42)
      pop(0x43)
      pop(0x44)
      pop(0x45)

      assert(
        !dut.io.outputValid.toBoolean,
        "FIFO should be empty"
      )

      println("[PASS] FIFO order = B -> C -> D -> E")
      println("[Done] Valid/Ready FIFO test passed.")

      simSuccess()
    }
}
```

---

# 8. 그런데 여기서 중요한 주의점

위 테스트의 `push()`에는:

```scala
while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

가 있습니다.

이전 UART 테스트에서 문제가 있었던 **무한 `while` 문제**를 기억해야 합니다.

따라서 학습용으로는 이 방식보다 **bounded wait**, 즉 최대 cycle 수를 정하는 방식이 더 안전합니다.

예를 들어:

```scala
for (_ <- 0 until 100) {
  if (dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
    return
  }

  dut.clockDomain.waitSampling()
}

assert(false, "Timeout waiting for FIFO ready")
```

처럼 만드는 것이 더 안전합니다.

이번 단계에서 특히 강조하고 싶은 부분입니다.

> **시뮬레이션의 `while`은 RTL의 `while`과 다릅니다.  
> Testbench에서는 반드시 종료 조건과 timeout을 고려해야 합니다.**

앞으로 제가 제공하는 SpinalSim 코드도 이 원칙으로 작성하는 것이 안전합니다.

---

# 9. 이번 단계에서 꼭 이해해야 하는 것

이제 FIFO를 다음 두 관점으로 볼 수 있어야 합니다.

### 기존 방식

```text
write
  +
!full
  ↓
실제 write
```

### Valid/Ready 방식

```text
valid
  +
ready
  ↓
fire
```

즉:

```text
write && !full
```

와

```text
valid && ready
```

는 FIFO의 입력 측면에서 사실상 같은 역할을 합니다.

하지만 `valid/ready` 방식은 **Producer와 Consumer 사이의 일반적인 인터페이스 규칙**으로 확장할 수 있습니다.

---

# 10. 다음 단계: SpinalHDL `Stream`

여기까지 직접 구현했으면 이제 SpinalHDL의 핵심 abstraction인:

```scala
Stream[T]
```

으로 넘어갈 준비가 됐습니다.

개념적으로:

```text
Stream[Bits(8 bits)]

       payload
          │
          ▼
   ┌─────────────┐
   │    Stream   │
   │             │
   │ valid       │
   │ ready       │
   │ payload     │
   └─────────────┘
```

그리고 우리가 방금 만든:

```text
inputValid
inputReady
inputData
```

가 `Stream`에서는 하나의 인터페이스로 묶입니다.

그 다음에는 드디어:

```text
Stream
   ↓
StreamFifo
   ↓
UART RX
   ↓
StreamFifo
   ↓
CPU
```

라는 **실제 SpinalHDL 설계 스타일**로 넘어갈 수 있습니다.

다음 단계에서는 **`Stream[Bits(8 bits)]`의 `valid/ready/fire`를 직접 만들어 보고, `StreamFifo`까지 연결**하겠습니다.