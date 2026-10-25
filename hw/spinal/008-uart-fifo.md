좋습니다. 👍

그럼 지금까지 만든 **`UartTx + UartRx + SimpleFifo`를 최대한 그대로 재사용**해서 다음 단계로 진행하겠습니다.

# Project 8 — UART TX → UART RX → FIFO

이번 단계의 구조는 다음과 같습니다.

```text
                 UART serial
┌──────────┐                     ┌──────────┐
│  UartTx  │ ──────────────────► │  UartRx  │
└────┬─────┘                     └────┬─────┘
     │                                │
     │ txData / txValid               │ rxData / rxValid
     │                                │
     │                                ▼
     │                         ┌─────────────┐
     │                         │ SimpleFifo   │
     │                         └──────┬──────┘
     │                                │
     │                                ▼
     │                         received data
     │
     └── TX serial output
```

즉,

**TX가 `0x41`을 전송 → RX가 `0x41`을 복원 → FIFO에 `0x41` 저장**

하는 것을 검증합니다.

---

# 1. 최종 목표

예를 들어 TX에:

```text
0x41 = 'A'
0x42 = 'B'
0x43 = 'C'
```

를 순서대로 전송합니다.

그러면:

```text
UartTx
   │
   │  serial bits
   ▼
UartRx
   │
   │  0x41
   ▼
FIFO
```

FIFO에는:

```text
[A][B][C]
 ↑
head
```

가 되어야 합니다.

그리고 FIFO에서 읽으면:

```text
read → 0x41
read → 0x42
read → 0x43
```

순서가 유지되어야 합니다.

---

# 2. 새로운 Top-Level

먼저 세 블록을 연결하는 컴포넌트를 만들겠습니다.

```scala id="71426"
import spinal.core._

case class UartRxFifo(
    clockFrequencyHz: Int,
    baudRate: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    // -------------------------
    // UART TX input
    // -------------------------

    val txData  = in Bits(8 bits)
    val txValid = in Bool()

    val txBusy  = out Bool()

    // -------------------------
    // UART serial connection
    // -------------------------

    val serial = out Bool()

    // -------------------------
    // FIFO output
    // -------------------------

    val fifoRead     = in Bool()
    val fifoReadData = out Bits(8 bits)

    val fifoEmpty = out Bool()
    val fifoFull  = out Bool()
  }

  // ==================================================
  // UART TX
  // ==================================================

  val tx = UartTx(
    clockFrequencyHz = clockFrequencyHz,
    baudRate = baudRate
  )

  tx.io.data  := io.txData
  tx.io.valid := io.txValid

  io.txBusy := tx.io.busy

  io.serial := tx.io.tx

  // ==================================================
  // UART RX
  // ==================================================

  val rx = UartRx(
    clockFrequencyHz = clockFrequencyHz,
    baudRate = baudRate
  )

  rx.io.rx := tx.io.tx

  // ==================================================
  // FIFO
  // ==================================================

  val fifo = SimpleFifo(
    dataWidth = 8,
    depth = fifoDepth
  )

  fifo.io.write     := rx.io.valid
  fifo.io.writeData := rx.io.data

  fifo.io.read := io.fifoRead

  io.fifoReadData := fifo.io.readData
  io.fifoEmpty    := fifo.io.empty
  io.fifoFull     := fifo.io.full
}
```

---

# 3. 데이터 흐름

이 부분을 잘 이해하는 것이 중요합니다.

TX 쪽에서:

```scala
tx.io.data  := io.txData
tx.io.valid := io.txValid
```

그리고 TX의 serial 출력:

```scala
rx.io.rx := tx.io.tx
```

따라서 실제 하드웨어 연결은:

```text
tx.io.tx
   │
   └──────────────► rx.io.rx
```

입니다.

RX가 한 byte를 성공적으로 받은 순간:

```scala
rx.io.valid
```

이 1-clock pulse가 됩니다.

그 순간 FIFO에:

```scala
fifo.io.write     := rx.io.valid
fifo.io.writeData := rx.io.data
```

가 들어갑니다.

즉:

```text
RX valid = 1
     │
     ├── FIFO write = 1
     │
     └── FIFO writeData = RX data
```

가 됩니다.

---

# 4. 중요한 부분: FIFO READ

FIFO의 READ는 외부에서:

```scala
io.fifoRead
```

로 요청하도록 만들었습니다.

따라서 테스트벤치에서:

```scala
dut.io.fifoRead #= true
```

를 주면 FIFO의 head를 읽고 `readPtr`가 이동합니다.

현재 `SimpleFifo`의 구조에서는:

```scala
io.readData := mem(readPtr)
```

이므로 **READ를 하기 전에도 현재 FIFO head의 데이터가 `fifoReadData`에 나타납니다.**

예를 들어:

```text
FIFO = A B C

fifoReadData = A
```

READ를 실행하면 다음 sampling 이후:

```text
FIFO = B C

fifoReadData = B
```

가 됩니다.

---

# 5. 첫 번째 테스트 목표

처음부터 A/B/C 여러 개를 보내지 않겠습니다.

우리가 이전 UART 시뮬레이션에서 **simulation scheduling 문제를 실제로 경험했기 때문**입니다.

이번에도 단계적으로 검증합니다.

### Step 1

```text
TX → 0x41
```

### Step 2

```text
UART TX → serial → UART RX
```

### Step 3

```text
RX valid
```

확인.

### Step 4

```text
RX data = 0x41
```

확인.

### Step 5

```text
FIFO count = 1
FIFO head = 0x41
```

확인.

### Step 6

FIFO READ.

### Step 7

FIFO가 다시 EMPTY인지 확인.

---

# 6. 테스트벤치

여기서는 이전에 문제가 있었던 무제한 `while` 루프를 사용하지 않겠습니다.

**명확한 clock 수를 기준으로 기다리겠습니다.**

10 clocks/bit이므로:

```text
10 bits × 10 clocks = 100 clocks
```

입니다.

```scala id="39157"
import spinal.core._
import spinal.core.sim._

object UartRxFifoSim extends App {

  val clockFrequency = 10_000_000
  val baudRate       = 1_000_000
  val cyclesPerBit   = clockFrequency / baudRate

  SimConfig
    .withWave
    .doSim(
      UartRxFifo(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate,
        fifoDepth = 4
      )
    ) { dut =>

      dut.clockDomain.forkStimulus(10)

      // ------------------------------------------------
      // Initial values
      // ------------------------------------------------

      dut.io.txData   #= 0
      dut.io.txValid  #= false
      dut.io.fifoRead #= false

      dut.clockDomain.waitSampling(2)

      println(
        s"[Initial] " +
        s"txBusy=${dut.io.txBusy.toBoolean} " +
        s"fifoEmpty=${dut.io.fifoEmpty.toBoolean} " +
        s"fifoFull=${dut.io.fifoFull.toBoolean} " +
        f"fifoReadData=0x${dut.io.fifoReadData.toBigInt}%02X"
      )

      assert(dut.io.fifoEmpty.toBoolean)
      assert(!dut.io.fifoFull.toBoolean)

      // ------------------------------------------------
      // Send 0x41
      // ------------------------------------------------

      dut.io.txData  #= 0x41
      dut.io.txValid #= true

      // TX samples valid and starts transmission.
      dut.clockDomain.waitSampling()

      dut.io.txValid #= false

      assert(
        dut.io.txBusy.toBoolean,
        "UART TX should be busy after accepting data"
      )

      println("[TX] 0x41 accepted")

      // ------------------------------------------------
      // Wait for complete UART frame
      //
      // 1 START
      // 8 DATA
      // 1 STOP
      //
      // = 10 bits
      //
      // 10 clocks / bit
      //
      // = 100 clocks
      // ------------------------------------------------

      dut.clockDomain.waitSampling(
        cyclesPerBit * 10 + 5
      )

      dut.clockDomain.waitFallingEdge()

      println(
        s"[After UART frame] " +
        s"txBusy=${dut.io.txBusy.toBoolean} " +
        s"fifoEmpty=${dut.io.fifoEmpty.toBoolean} " +
        s"fifoFull=${dut.io.fifoFull.toBoolean} " +
        f"fifoReadData=0x${dut.io.fifoReadData.toBigInt}%02X"
      )

      // ------------------------------------------------
      // Verify RX -> FIFO
      // ------------------------------------------------

      assert(
        dut.io.fifoEmpty.toBoolean == false,
        "FIFO should contain received data"
      )

      assert(
        dut.io.fifoReadData.toBigInt == 0x41,
        "FIFO head should contain received 0x41"
      )

      println("[PASS] UART RX received 0x41")
      println("[PASS] FIFO contains 0x41")

      // ------------------------------------------------
      // Read FIFO
      // ------------------------------------------------

      dut.io.fifoRead #= true

      dut.clockDomain.waitSampling()

      dut.io.fifoRead #= false

      dut.clockDomain.waitFallingEdge()

      println(
        s"[After FIFO read] " +
        s"fifoEmpty=${dut.io.fifoEmpty.toBoolean} " +
        f"fifoReadData=0x${dut.io.fifoReadData.toBigInt}%02X"
      )

      assert(
        dut.io.fifoEmpty.toBoolean,
        "FIFO should be empty after reading its only byte"
      )

      println("[PASS] FIFO read completed")
      println("[Done] UART TX -> UART RX -> FIFO test passed.")
    }
}
```

---

# 7. 예상 결과

정상적으로 동작하면 대략:

```text
[Initial] txBusy=false fifoEmpty=true fifoFull=false fifoReadData=0x00

[TX] 0x41 accepted

[After UART frame]
txBusy=false
fifoEmpty=false
fifoFull=false
fifoReadData=0x41

[PASS] UART RX received 0x41
[PASS] FIFO contains 0x41

[After FIFO read]
fifoEmpty=true
fifoReadData=0x00

[PASS] FIFO read completed
[Done] UART TX -> UART RX -> FIFO test passed.
```

---

# ⚠️ 한 가지 주의

이 테스트는 **현재 우리가 만든 교육용 `UartTx` / `UartRx`의 타이밍 구조를 그대로 전제로 합니다.**

특히 이전에 확인했던 것처럼:

```text
10 clocks / bit
×
10 bits
=
100 clocks
```

이라는 관계를 정확히 사용하고 있습니다.

그리고 이번 테스트에서는 이전의 문제를 반복하지 않도록:

```scala
while (dut.io.txBusy.toBoolean) {
    ...
}
```

같은 무제한 조건 루프를 사용하지 않았습니다.

---

## 이번 단계의 목표

이번 테스트가 통과하면 다음 순서로 바로 확장하겠습니다.

```text
① 0x41
   ↓
TX → RX → FIFO
   ↓
FIFO read
```

↓

```text
② 0x41, 0x42, 0x43
   ↓
TX → RX → FIFO
   ↓
FIFO = A B C
```

↓

```text
③ FIFO A → B → C 순서 확인
```

↓

```text
④ UART RX FIFO의 의미와 실제 UART 시스템 구조
```

우선 **위 `UartRxFifoSim`을 실행해 주세요.** 결과를 보여주시면 그 결과를 기준으로 다음 단계로 진행하겠습니다.