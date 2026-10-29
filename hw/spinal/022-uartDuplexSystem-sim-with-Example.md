좋습니다. 이제 방금 학습한 **`StreamDriver` + `StreamMonitor` + `StreamReadyRandomizer` + `ScoreboardInOrder`**를 정석적으로 사용해서 `UartDuplexSystemSim.scala`를 다시 작성하겠습니다.

이번 버전에서는 특히 이전에 발생했던 문제를 피하기 위해:

- TX driver가 `Scoreboard`를 직접 조작하지 않음
- TX의 실제 `fire`를 `StreamMonitor`로 관찰하여 `pushRef()`
- RX의 실제 `fire`를 `StreamMonitor`로 관찰하여 `pushDut()`
- RX backpressure는 `StreamReadyRandomizer` 사용
- 수동 `while (!ready)` 루프 최소화
- 테스트별 scoreboard를 분리
- 테스트 종료 조건을 `scoreboard.matches`로 명확하게 관리
- **SpinalHDL 1.15.0 기준**

으로 구성하겠습니다.

다만 한 가지 중요한 전제가 있습니다. 현재 사용 중인 `UartTxStreamAdapter`가 앞서 제가 제안한 
버퍼 방식이라면, **그 adapter 자체의 handshake 동작도 다시 검증해야 합니다.** 이번 simulation은 그 
부분까지 숨기지 않고 실제 Stream handshake를 기준으로 검사하도록 작성하겠습니다.

## `UartDuplexSystemSim.scala`

```scala
package sim

import spinal.core._
import spinal.core.sim._
import spinal.lib._
import spinal.lib.sim._

import integration._

object UartDuplexSystemSim extends App {

  // ============================================================
  // Configuration
  // ============================================================

  val clockFrequencyHz = 10_000_000
  val baudRate         = 1_000_000
  val fifoDepth        = 8

  val clockPeriod = 10

  // ============================================================
  // Test 1
  //
  // Full duplex
  // No RX backpressure
  // ============================================================

  def testNoBackpressure(): Unit = {

    println()
    println("================================================")
    println("TEST 1 : Full Duplex / No Backpressure")
    println("================================================")

    val dut =
      SimConfig
        .withWave
        .compile(
          UartDuplexSystem(
            clockFrequencyHz = clockFrequencyHz,
            baudRate = baudRate,
            fifoDepth = fifoDepth
          )
        )

    dut.doSim("test1_no_backpressure") { dut =>

      SimTimeout(10000)

      // --------------------------------------------------------
      // Clock
      // --------------------------------------------------------

      dut.clockDomain.forkStimulus(clockPeriod)

      // --------------------------------------------------------
      // Initial values
      // --------------------------------------------------------

      dut.io.aTx.valid   #= false
      dut.io.aTx.payload #= 0

      dut.io.bTx.valid   #= false
      dut.io.bTx.payload #= 0

      dut.io.aRx.ready #= true
      dut.io.bRx.ready #= true

      // --------------------------------------------------------
      // Scoreboards
      //
      // A TX -> B RX
      // B TX -> A RX
      // --------------------------------------------------------

      val aToB =
        ScoreboardInOrder[Int]()

      val bToA =
        ScoreboardInOrder[Int]()

      // --------------------------------------------------------
      // A TX monitor
      //
      // 실제 A TX Stream fire를 관찰한다.
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.aTx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[A TX FIRE] 0x$value%02X"
        )

        aToB.pushRef(value)
      }

      // --------------------------------------------------------
      // B TX monitor
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.bTx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[B TX FIRE] 0x$value%02X"
        )

        bToA.pushRef(value)
      }

      // --------------------------------------------------------
      // A RX monitor
      //
      // B TX -> A RX
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.aRx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[A RX FIRE] 0x$value%02X"
        )

        bToA.pushDut(value)
      }

      // --------------------------------------------------------
      // B RX monitor
      //
      // A TX -> B RX
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.bRx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[B RX FIRE] 0x$value%02X"
        )

        aToB.pushDut(value)
      }

      // --------------------------------------------------------
      // A TX Driver
      //
      // StreamDriver가 ready/valid handshake를 관리한다.
      // Driver에서는 scoreboard를 직접 건드리지 않는다.
      // --------------------------------------------------------

      StreamDriver(
        dut.io.aTx,
        dut.clockDomain
      ) { payload =>

        payload #= 0x40

        true
      }

      // --------------------------------------------------------
      // B TX Driver
      // --------------------------------------------------------

      StreamDriver(
        dut.io.bTx,
        dut.clockDomain
      ) { payload =>

        payload #= 0x60

        true
      }

      // --------------------------------------------------------
      // Run
      //
      // 충분한 시간 동안 UART transaction이 진행되도록 한다.
      // --------------------------------------------------------

      dut.clockDomain.waitSampling(2000)

      // --------------------------------------------------------
      // Scoreboard check
      // --------------------------------------------------------

      println(
        s"[INFO] A -> B matches = ${aToB.matches}"
      )

      println(
        s"[INFO] B -> A matches = ${bToA.matches}"
      )

      aToB.checkEmptyness()
      bToA.checkEmptyness()

      println(
        "[PASS] test1_no_backpressure"
      )
    }
  }

  // ============================================================
  // Test 2
  //
  // Full duplex
  // Random RX backpressure
  // ============================================================

  def testRandomBackpressure(): Unit = {

    println()
    println("================================================")
    println("TEST 2 : Full Duplex / Random Backpressure")
    println("================================================")

    val dut =
      SimConfig
        .withWave
        .compile(
          UartDuplexSystem(
            clockFrequencyHz = clockFrequencyHz,
            baudRate = baudRate,
            fifoDepth = fifoDepth
          )
        )

    dut.doSim("test2_random_backpressure") { dut =>

      SimTimeout(20000)

      // --------------------------------------------------------
      // Clock
      // --------------------------------------------------------

      dut.clockDomain.forkStimulus(clockPeriod)

      // --------------------------------------------------------
      // Initial values
      // --------------------------------------------------------

      dut.io.aTx.valid   #= false
      dut.io.aTx.payload #= 0

      dut.io.bTx.valid   #= false
      dut.io.bTx.payload #= 0

      // --------------------------------------------------------
      // Scoreboards
      // --------------------------------------------------------

      val aToB =
        ScoreboardInOrder[Int]()

      val bToA =
        ScoreboardInOrder[Int]()

      // --------------------------------------------------------
      // TX monitors
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.aTx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[A TX FIRE] 0x$value%02X"
        )

        aToB.pushRef(value)
      }

      StreamMonitor(
        dut.io.bTx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[B TX FIRE] 0x$value%02X"
        )

        bToA.pushRef(value)
      }

      // --------------------------------------------------------
      // RX monitors
      // --------------------------------------------------------

      StreamMonitor(
        dut.io.aRx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[A RX FIRE] 0x$value%02X"
        )

        bToA.pushDut(value)
      }

      StreamMonitor(
        dut.io.bRx,
        dut.clockDomain
      ) { payload =>

        val value = payload.toInt

        println(
          f"[B RX FIRE] 0x$value%02X"
        )

        aToB.pushDut(value)
      }

      // --------------------------------------------------------
      // Random RX backpressure
      //
      // ready를 random하게 발생시킨다.
      // --------------------------------------------------------

      StreamReadyRandomizer(
        dut.io.aRx,
        dut.clockDomain
      )

      StreamReadyRandomizer(
        dut.io.bRx,
        dut.clockDomain
      )

      // --------------------------------------------------------
      // A TX Driver
      // --------------------------------------------------------

      StreamDriver(
        dut.io.aTx,
        dut.clockDomain
      ) { payload =>

        payload.randomize()

        true
      }

      // --------------------------------------------------------
      // B TX Driver
      // --------------------------------------------------------

      StreamDriver(
        dut.io.bTx,
        dut.clockDomain
      ) { payload =>

        payload.randomize()

        true
      }

      // --------------------------------------------------------
      // Run simulation
      // --------------------------------------------------------

      dut.clockDomain.waitSampling(5000)

      // --------------------------------------------------------
      // Scoreboard check
      // --------------------------------------------------------

      println(
        s"[INFO] A -> B matches = ${aToB.matches}"
      )

      println(
        s"[INFO] B -> A matches = ${bToA.matches}"
      )

      aToB.checkEmptyness()
      bToA.checkEmptyness()

      println(
        "[PASS] test2_random_backpressure"
      )
    }
  }

  // ============================================================
  // Main
  // ============================================================

  testNoBackpressure()

  testRandomBackpressure()

  println()
  println("================================================")
  println("All UartDuplexSystem simulations completed.")
  println("================================================")
}
```

## 그런데 여기서 중요한 문제가 하나 있습니다

위 코드는 **학습한 4개의 simulation helper를 제대로 사용하는 구조**로 바뀌었습니다.

특히:

```scala
StreamDriver(...)
```

가 TX를 담당하고,

```scala
StreamMonitor(dut.io.aTx, ...)
```

가 실제 TX `fire`를 관찰해서:

```scala
aToB.pushRef(...)
```

합니다.

그리고 RX에서는:

```scala
StreamMonitor(dut.io.bRx, ...)
```

가 실제 RX `fire`를 관찰해서:

```scala
aToB.pushDut(...)
```

합니다.

즉:

```text
A TX
 │
 │ StreamDriver
 ▼
A UART
 │
 │ serial
 ▼
B UART
 │
 ▼
B RX
 │
 │ StreamMonitor
 ▼
pushDut()
```

이고 scoreboard는:

```text
A TX Monitor
     │
     ▼
 pushRef()
     │
     ▼
┌───────────────┐
│ Scoreboard    │
│               │
│ expected      │
│     vs        │
│ actual        │
└───────────────┘
     ▲
     │
 pushDut()
     │
B RX Monitor
```

가 됩니다.

---

## 다만 Test 1의 Driver는 학습용으로 조금 수정하는 것이 좋습니다

위 코드에서:

```scala
StreamDriver(dut.io.aTx, dut.clockDomain) { payload =>
  payload #= 0x40
  true
}
```

는 `0x40`을 **계속 생성**합니다.

즉:

```text
0x40
0x40
0x40
0x40
...
```

가 됩니다.

실제 우리가 원하는 것은 이전 테스트처럼:

```text
A → B : 40 41 42
B → A : 60 61
```

입니다.

따라서 **이번 학습의 목적에 맞게 `StreamDriver`를 사용하면서도 정해진 sequence를 전송하도록** 만드는 것이 더 좋습니다.

예를 들어 helper 함수를 하나 만들겠습니다.

```scala
def driveSequence(
    stream: Stream[Bits],
    clockDomain: ClockDomain,
    values: Seq[Int]
): Unit = {

  var index = 0

  StreamDriver(
    stream,
    clockDomain
  ) { payload =>

    if (index < values.length) {

      payload #= values(index)

      index += 1

      true

    } else {

      false
    }
  }
}
```

그러면:

```scala
driveSequence(
  dut.io.aTx,
  dut.clockDomain,
  Seq(0x40, 0x41, 0x42)
)

driveSequence(
  dut.io.bTx,
  dut.clockDomain,
  Seq(0x60, 0x61)
)
```

처럼 사용할 수 있습니다.

**이 방식이 지금 우리가 학습한 `StreamDriver`를 실제 UART simulation에 적용하는 가장 깔끔한 형태입니다.**

그리고 한 가지 더 중요합니다. 현재 `UartTxStreamAdapter`의 내부 handshake가 정확히 한 번의 Stream transaction을 
한 번의 UART 전송으로 변환하는지 확인한 뒤에야 이 simulation을 최종 "정상 동작"으로 판정할 수 있습니다. 
이전 오류가 바로 그 경계에서 발생했기 때문에, 다음 단계에서는 **`StreamDriver → UartTxStreamAdapter → UartTx`의 handshake를 
waveform으로 확인**하는 것이 가장 안전합니다.

---

---

`StreamDriver → UartTxStreamAdapter → UartTx`의 handshake를 waveform으로 확인할 수 있도록 시뮬레이션 검증 코드를 작성하라.

---

좋습니다. 이번 검증은 **Full Duplex 전체를 테스트하기 전에**, 가장 문제가 되었던 경로 하나만 떼어내서 확인하는 것이 좋습니다.

검증 대상은 정확히 다음입니다.

```text
StreamDriver
    │
    │ input.valid / input.payload
    │
    ▼
UartTxStreamAdapter
    │
    │ txValid / txData
    │
    ▼
UartTx
    │
    │ busy
    ▼
serial TX
```

특히 waveform에서 다음을 확인합니다.

1. `StreamDriver`가 `0x40 → 0x41 → 0x42`를 공급하는가?
2. `input.valid && input.ready`가 **각 byte당 정확히 한 번** 발생하는가?
3. Adapter의 `pendingValid`가 한 byte를 잡고 있는가?
4. `txValid`가 UART가 IDLE일 때 정확히 한 번만 올라가는가?
5. UART가 같은 byte를 **두 번 받아들이지 않는가?**
6. `busy`가 한 byte 전송 동안 유지되는가?
7. `0x40 → 0x41 → 0x42`가 UART serial frame으로 순서대로 나가는가?

아래 코드는 **SpinalHDL 1.15.0 기준**입니다.

---

# 1. 먼저 Adapter에 simulation debug signal 추가

현재 `UartTxStreamAdapter`의 내부:

```scala
pendingValid
pendingData
```

를 waveform에서 직접 보고 싶습니다.

따라서 simulation visibility를 위해 `simPublic()`을 추가하겠습니다.

### `UartTxStreamAdapter.scala`

```scala
package uart

import spinal.core._
import spinal.core.sim._
import spinal.lib._

case class UartTxStreamAdapter() extends Component {

  val io = new Bundle {
    // CPU / application side
    val input = slave Stream(Bits(8 bits))

    // UART TX side
    val txData  = out Bits(8 bits)
    val txValid = out Bool()
    val txBusy  = in Bool()
  }

  // ============================================================
  // 1-byte holding register
  // ============================================================

  val pendingValid =
    Reg(Bool()) init(False)

  val pendingData =
    Reg(Bits(8 bits)) init(0)

  // ------------------------------------------------------------
  // Simulation visibility
  // ------------------------------------------------------------

  pendingValid.simPublic()
  pendingData.simPublic()

  // ============================================================
  // Stream input
  // ============================================================

  io.input.ready :=
    !pendingValid && !io.txBusy

  when(io.input.fire) {
    pendingData  := io.input.payload
    pendingValid := True
  }

  // ============================================================
  // UART TX
  // ============================================================

  io.txData  := pendingData
  io.txValid := pendingValid

  // ============================================================
  // UART acceptance
  //
  // UartTx가 IDLE이고 txValid가 올라온 cycle에서
  // 새로운 byte를 받아 START 상태로 들어간다.
  // ============================================================

  when(pendingValid && io.txBusy) {
    pendingValid := False
  }
}
```

여기서:

```scala
import spinal.core.sim._
```

이 필요합니다.

이것은 앞에서 확인했던 SpinalHDL 1.15.0의 `simPublic()` 사용 방식과 동일합니다.

---

# 2. 검증용 Top Component

이번에는 `UartDuplexSystem`까지 사용하지 않습니다.

오직:

```text
Stream
  ↓
Adapter
  ↓
UartTx
```

만 연결합니다.

### `UartTxHandshakeTestTop.scala`

```scala
package sim

import spinal.core._
import spinal.lib._

import uart._

case class UartTxHandshakeTestTop(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {

    // ----------------------------------------------------------
    // Testbench -> UART TX
    // ----------------------------------------------------------

    val input =
      slave Stream(Bits(8 bits))

    // ----------------------------------------------------------
    // UART serial output
    // ----------------------------------------------------------

    val tx =
      out Bool()

    // ----------------------------------------------------------
    // Debug signals
    // ----------------------------------------------------------

    val inputReady =
      out Bool()

    val txValid =
      out Bool()

    val txData =
      out Bits(8 bits)

    val txBusy =
      out Bool()
  }

  // ============================================================
  // Adapter
  // ============================================================

  val adapter =
    UartTxStreamAdapter()

  adapter.io.input << io.input

  // ============================================================
  // UART TX
  // ============================================================

  val uartTx =
    UartTx(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  adapter.io.txBusy :=
    uartTx.io.busy

  uartTx.io.data :=
    adapter.io.txData

  uartTx.io.valid :=
    adapter.io.txValid

  // ============================================================
  // Outputs
  // ============================================================

  io.tx :=
    uartTx.io.tx

  io.inputReady :=
    io.input.ready

  io.txValid :=
    adapter.io.txValid

  io.txData :=
    adapter.io.txData

  io.txBusy :=
    uartTx.io.busy
}
```

이렇게 하면 waveform에서 최소한 다음을 한눈에 볼 수 있습니다.

```text
input.valid
input.ready
input.payload

pendingValid
pendingData

txValid
txData
txBusy

tx
```

---

# 3. 실제 검증 코드

이제 핵심입니다.

### `UartTxHandshakeSim.scala`

```scala
package sim

import spinal.core._
import spinal.core.sim._
import spinal.lib._
import spinal.lib.sim._

object UartTxHandshakeSim extends App {

  // ============================================================
  // Configuration
  // ============================================================

  val clockFrequencyHz = 10_000_000
  val baudRate         = 1_000_000

  val clockPeriod = 10

  // ============================================================
  // Test data
  // ============================================================

  val testData =
    Seq(
      0x40,
      0x41,
      0x42
    )

  // ============================================================
  // DUT
  // ============================================================

  val compiled =
    SimConfig
      .withWave
      .compile(
        UartTxHandshakeTestTop(
          clockFrequencyHz = clockFrequencyHz,
          baudRate = baudRate
        )
      )

  compiled.doSim("uart_tx_handshake") { dut =>

    // ==========================================================
    // Timeout
    // ==========================================================

    SimTimeout(10000)

    // ==========================================================
    // Clock
    // ==========================================================

    dut.clockDomain.forkStimulus(clockPeriod)

    // ==========================================================
    // Initial values
    // ==========================================================

    dut.io.input.valid   #= false
    dut.io.input.payload #= 0

    // ==========================================================
    // Counters
    // ==========================================================

    var inputFireCount = 0

    var uartAcceptCount = 0

    // ==========================================================
    // Stream monitor
    //
    // 실제 input Stream fire를 관찰한다.
    // ==========================================================

    StreamMonitor(
      dut.io.input,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      inputFireCount += 1

      println(
        f"[STREAM FIRE] data=0x$value%02X"
      )
    }

    // ==========================================================
    // UART acceptance monitor
    //
    // UartTx는 ready 신호가 없기 때문에
    //
    //     txValid && !txBusy
    //
    // 인 순간을 UART가 byte를 받아들이는 순간으로 본다.
    // ==========================================================

    fork {

      while (true) {

        dut.clockDomain.waitSampling()

        val txValid =
          dut.io.txValid.toBoolean

        val txBusy =
          dut.io.txBusy.toBoolean

        if (txValid && !txBusy) {

          val value =
            dut.io.txData.toInt

          uartAcceptCount += 1

          println(
            f"[UART ACCEPT] data=0x$value%02X"
          )
        }
      }
    }

    // ==========================================================
    // StreamDriver
    //
    // 0x40 -> 0x41 -> 0x42
    // ==========================================================

    var index = 0

    StreamDriver(
      dut.io.input,
      dut.clockDomain
    ) { payload =>

      if (index < testData.length) {

        val value =
          testData(index)

        payload #= value

        println(
          f"[DRIVER] data=0x$value%02X"
        )

        index += 1

        true

      } else {

        false
      }
    }

    // ==========================================================
    // Wait until all input transactions have fired
    // ==========================================================

    dut.clockDomain.waitActiveEdgeWhere(
      inputFireCount == testData.length
    )

    println()
    println(
      s"[CHECK] inputFireCount = $inputFireCount"
    )

    assert(
      inputFireCount == testData.length,
      s"Expected ${testData.length} input fires, got $inputFireCount"
    )

    // ==========================================================
    // Wait until all UART bytes have been accepted
    // ==========================================================

    dut.clockDomain.waitActiveEdgeWhere(
      uartAcceptCount == testData.length
    )

    println()
    println(
      s"[CHECK] uartAcceptCount = $uartAcceptCount"
    )

    assert(
      uartAcceptCount == testData.length,
      s"Expected ${testData.length} UART accepts, got $uartAcceptCount"
    )

    // ==========================================================
    // Give UART enough time to transmit the final frame.
    //
    // 1 byte = 10 bits
    // 10 clocks / bit
    // = 100 clocks / byte
    //
    // final byte까지 waveform에서 충분히 관찰하기 위해
    // 추가 clock을 진행한다.
    // ==========================================================

    dut.clockDomain.waitSampling(150)

    // ==========================================================
    // Final result
    // ==========================================================

    println()
    println("==============================================")
    println("UART TX handshake verification completed.")
    println("==============================================")
  }
}
```

---

# 4. 이 테스트에서 가장 중요한 부분

## `StreamDriver`

```scala
StreamDriver(
  dut.io.input,
  dut.clockDomain
) { payload =>

  if (index < testData.length) {
    payload #= testData(index)
    index += 1
    true
  } else {
    false
  }
}
```

테스트 데이터:

```text
0x40
0x41
0x42
```

를 순서대로 Stream에 넣습니다.

Driver가 직접 `ready`를 검사하거나 `valid`를 조작하지 않습니다.

이것이 이번에 학습한 `StreamDriver`의 사용법입니다.

---

# 5. `StreamMonitor`

```scala
StreamMonitor(
  dut.io.input,
  dut.clockDomain
) { payload =>
    ...
}
```

이 Monitor는:

```text
input.valid && input.ready
```

가 실제로 발생한 transaction을 관찰합니다.

따라서:

```text
[DRIVER]
0x40

      ↓

[STREAM FIRE]
0x40
```

가 되어야 합니다.

---

# 6. UART 쪽은 `Stream`이 아니다

여기서 매우 중요한 차이가 있습니다.

우리 `UartTx`의 interface는:

```scala
val data  = in Bits(8 bits)
val valid = in Bool()
val tx    = out Bool()
val busy  = out Bool()
```

입니다.

즉:

```text
UART TX
```

에는:

```text
ready
```

가 없습니다.

따라서:

```scala
StreamMonitor(dut.io.tx...)
```

를 사용할 수 없습니다.

우리는 UART가 byte를 받아들이는 조건을:

```text
txValid && !txBusy
```

로 정의해서 관찰합니다.

---

# 7. 가장 중요한 waveform 패턴

정상적인 경우 첫 번째 byte `0x40`에 대해 다음과 같은 흐름을 기대합니다.

### Cycle A

```text
input.valid = 1
input.ready = 1
input.payload = 0x40
```

따라서:

```text
input.fire = 1
```

입니다.

동시에 Adapter가:

```text
pendingValid ← 1
pendingData  ← 0x40
```

를 저장합니다.

---

### Cycle B

그 다음에는:

```text
pendingValid = 1
pendingData  = 0x40

txValid = 1
txData  = 0x40

txBusy = 0
```

가 됩니다.

따라서:

```text
txValid && !txBusy
```

이고 UART가 `0x40`을 받아들입니다.

그 결과:

```text
UART ACCEPT = 0x40
```

가 발생합니다.

그리고 UART는:

```text
IDLE → START
```

로 전환합니다.

---

### Cycle C 이후

```text
txBusy = 1
```

이 됩니다.

따라서 같은 `0x40`을 다시 받아들이면 안 됩니다.

Adapter의:

```text
pendingValid
```

도 내려가야 합니다.

즉:

```text
0x40
```

에 대한 UART acceptance는 **정확히 한 번**이어야 합니다.

---

# 8. 우리가 특히 찾는 버그

이전 문제가 바로 이것이었습니다.

잘못된 구조라면:

```text
       0x40
        │
        ▼
    txValid = 1
        │
        ▼
   UART IDLE
        │
        ▼
    0x40 accept
        │
        ▼
   UART START
        │
        ▼
   UART DATA...
        │
        ▼
   UART STOP
        │
        ▼
   UART IDLE
        │
        ▼
   txValid가 아직 1 ?
        │
       YES
        │
        ▼
    0x40 AGAIN
```

이라는 문제가 생길 수 있습니다.

이번 테스트의 핵심 목적은 바로 이것입니다.

---

# 9. 정상 waveform에서 기대하는 관계

최종적으로 GTKWave에서 다음 관계를 확인하면 됩니다.

```text
                0x40             0x41             0x42

input.valid    ┌─────┐          ┌─────┐          ┌─────┐
               │     │          │     │          │     │
───────────────┘     └──────────┘     └──────────┘     └──

input.ready    ────┐  ┌────────────────┐  ┌──────────────
                   └──┘                └──┘

input.fire         ▲                    ▲                 ▲
                   │                    │                 │
                   40                   41                42


txValid         ┌───┐                         ┌───┐
                │   │                         │   │
────────────────┘   └─────────────────────────┘   └────────

txData          40                              41
                ▲                               ▲
                │                               │
             UART accept                     UART accept


txBusy             ┌──────────────────┐
                   │                  │
───────────────────┘                  └──────────────────
```

실제 UART `tx`는 각각:

```text
0x40
```

에 대해서:

```text
START
D0
D1
...
D7
STOP
```

10 bit frame을 만들고,

그 다음:

```text
0x41
```

그리고:

```text
0x42
```

순으로 전송해야 합니다.

---

# 10. 이 테스트의 성공 조건

단순히 simulation이 종료되는 것으로 성공이라고 판단하지 않습니다.

최소한 다음이 성립해야 합니다.

```text
input Stream fire

0x40
0x41
0x42
```

각각 정확히 한 번:

```text
inputFireCount = 3
```

그리고 UART acceptance도:

```text
0x40
0x41
0x42
```

각각 정확히 한 번:

```text
uartAcceptCount = 3
```

이어야 합니다.

즉:

```text
StreamDriver
     │
     │ 0x40
     ▼
input.fire
     │
     ▼
Adapter
     │
     │ 0x40
     ▼
UART accept
```

이 관계가:

```text
0x40 → 1회
0x41 → 1회
0x42 → 1회
```

로 정확히 유지되는지를 검증하는 것입니다.

**이 테스트가 먼저 정상적으로 확인된 후에 `UartEndpoint → UartDuplexSystem`으로 올라가는 것이 가장 안전합니다.** 특히 이번에는 waveform에서 `input.valid/ready`, `pendingValid/pendingData`, `txValid/txData`, `txBusy`, `tx`를 직접 비교하면 이전에 발생했던 `0x41 → 0x42` scoreboard mismatch의 원인을 상당히 명확하게 확인할 수 있습니다.