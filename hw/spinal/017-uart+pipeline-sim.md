좋습니다. 이번 단계는 **지금까지 배운 것들을 하나의 실제 시스템으로 묶는 단계**로 가겠습니다.

이번 작품의 목표는 단순히 "UART가 동작한다"가 아니라:

```text
CPU/Producer
    │
    │ Stream
    ▼
UART TX
    │
    │ serial TXD
    ▼
─────────────── loopback cable ───────────────
    ▲                                         │
    │ serial RXD                              ▼
UART RX                                      UART RX
    │                                         ▲
    │ Flow                                    │
    ▼                                         │
Flow → Stream                                Flow → Stream
    │                                         │
    ▼                                         │
SpinalHDL StreamFifo                         StreamFifo
    │                                         │
    ▼                                         ▼
Pipeline Stage 1                           Pipeline Stage 1
    │                                         │
    ▼                                         ▼
Pipeline Stage 2                           Pipeline Stage 2
    │                                         │
    ▼                                         ▼
Pipeline Stage 3                           Pipeline Stage 3
    │                                         │
    ▼                                         ▼
CPU/Consumer                               CPU/Consumer
```

그리고 **A→B와 B→A를 동시에 보내는 Full Duplex**로 만들겠습니다.

특히 RX 쪽은 `Flow → Stream → StreamFifo → 3-stage Stream pipeline`으로 만들고, 마지막 CPU consumer가 `ready`를 일부러 낮춰서 **Backpressure를 실제로 발생**시키겠습니다.

SpinalHDL의 `Stream`은 `valid/payload`가 master→slave, `ready`가 slave→master로 흐르고 `fire = valid && ready`인 handshake 구조입니다. `StreamFifo`는 `push`가 slave Stream, `pop`이 master Stream입니다. 또한 Stream에는 `.stage()`를 이용한 1-cycle register stage가 제공됩니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

---

# 1. 이번 프로젝트의 구조

우선 이름을 명확하게 하겠습니다.

```text
             UART LINK A ↔ B
             
        Endpoint A                 Endpoint B
   ┌──────────────────┐       ┌──────────────────┐
   │                  │       │                  │
CPU TX ─> UartTx ─────┼──────>│ UartRx ─> FIFO   │
   │                  │       │             │    │
   │                  │       │        Pipeline  │
   │                  │       │             │    │
CPU RX <──────────────┼───────┤<────────────┘    │
   │                  │       │                  │
   └──────────────────┘       └──────────────────┘
```

실제로는 duplex이므로 두 선이 있습니다.

```text
A.TX ─────────────────────> B.RX
A.RX <───────────────────── B.TX
```

즉:

```text
       A                              B

    UartTx ────────────────> UartRx
    UartRx <──────────────── UartTx
```

---

# 2. 이번 프로젝트에서 사용하는 것

### 우리가 직접 만든 것

- `UartTx`
- `UartRx`

### SpinalHDL 제공 기능

- `Stream`
- `StreamFifo`
- `Stream.stage()`
- `Flow.toStream()`
- SpinalSim `StreamDriver`
- SpinalSim `StreamMonitor`
- SpinalSim `StreamReadyRandomizer`
- `ScoreboardInOrder`

특히 `StreamFifo`와 Stream simulation utility들은 SpinalHDL 공식 문서에서도 FIFO 검증 예제로 사용됩니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

---

# 3. 먼저 우리가 사용할 UartTx

기존에 만들었던 인터페이스를 그대로 유지하겠습니다.

```scala
import spinal.core._

case class UartTx(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {
    val data  = in Bits(8 bits)
    val valid = in Bool()

    val tx   = out Bool()
    val busy = out Bool()
  }

  val cyclesPerBit =
    clockFrequencyHz / baudRate

  require(
    cyclesPerBit >= 2,
    "clockFrequencyHz / baudRate must be at least 2"
  )

  val timerWidth =
    log2Up(cyclesPerBit)

  val bitTimer =
    Reg(UInt(timerWidth bits)) init(0)

  val bitCounter =
    Reg(UInt(3 bits)) init(0)

  val shiftReg =
    Reg(Bits(8 bits)) init(0)

  object State extends SpinalEnum {
    val IDLE, START, DATA, STOP = newElement()
  }

  val state =
    RegInit(State.IDLE)

  io.busy := state =/= State.IDLE

  io.tx := True

  switch(state) {

    is(State.IDLE) {
      io.tx := True

      bitTimer := 0
      bitCounter := 0

      when(io.valid) {
        shiftReg := io.data
        bitTimer := 0
        bitCounter := 0
        state := State.START
      }
    }

    is(State.START) {
      io.tx := False

      when(bitTimer === cyclesPerBit - 1) {
        bitTimer := 0
        bitCounter := 0
        state := State.DATA
      } otherwise {
        bitTimer := bitTimer + 1
      }
    }

    is(State.DATA) {
      io.tx := shiftReg(bitCounter)

      when(bitTimer === cyclesPerBit - 1) {
        bitTimer := 0

        when(bitCounter === 7) {
          state := State.STOP
        } otherwise {
          bitCounter := bitCounter + 1
        }
      } otherwise {
        bitTimer := bitTimer + 1
      }
    }

    is(State.STOP) {
      io.tx := True

      when(bitTimer === cyclesPerBit - 1) {
        bitTimer := 0
        state := State.IDLE
      } otherwise {
        bitTimer := bitTimer + 1
      }
    }
  }
}
```

---

# 4. UartRx

그리고 우리가 앞에서 만든 **START 3-point majority voting 버전**을 그대로 사용합니다.

```scala
import spinal.core._

case class UartRx(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {
    val rx = in Bool()

    val data  = out Bits(8 bits)
    val valid = out Bool()
  }

  val cyclesPerBit =
    clockFrequencyHz / baudRate

  require(
    cyclesPerBit >= 4,
    "clockFrequencyHz / baudRate must be at least 4"
  )

  val timerWidth =
    log2Up(cyclesPerBit)

  val halfBitCount =
    cyclesPerBit / 2

  val bitTimer =
    Reg(UInt(timerWidth bits)) init(0)

  val bitCounter =
    Reg(UInt(3 bits)) init(0)

  val shiftReg =
    Reg(Bits(8 bits)) init(0)

  val startSample0 =
    Reg(Bool()) init(False)

  val startSample1 =
    Reg(Bool()) init(False)

  val startSample2 =
    Reg(Bool()) init(False)

  object State extends SpinalEnum {
    val IDLE, START, DATA, STOP = newElement()
  }

  val state =
    RegInit(State.IDLE)

  io.data  := shiftReg
  io.valid := False

  switch(state) {

    is(State.IDLE) {
      bitTimer := 0
      bitCounter := 0

      when(!io.rx) {
        bitTimer := 0

        startSample0 := False
        startSample1 := False
        startSample2 := False

        state := State.START
      }
    }

    is(State.START) {

      when(bitTimer === halfBitCount - 2) {

        startSample0 := io.rx
        bitTimer := bitTimer + 1

      } elsewhen(bitTimer === halfBitCount - 1) {

        startSample1 := io.rx
        bitTimer := bitTimer + 1

      } elsewhen(bitTimer === halfBitCount) {

        startSample2 := io.rx
        bitTimer := 0

        when(
          (!startSample0 && !startSample1) ||
          (!startSample1 && !io.rx) ||
          (!startSample0 && !io.rx)
        ) {
          bitCounter := 0
          state := State.DATA
        } otherwise {
          state := State.IDLE
        }

      } otherwise {
        bitTimer := bitTimer + 1
      }
    }

    is(State.DATA) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        shiftReg(bitCounter) := io.rx

        when(bitCounter === 7) {
          state := State.STOP
        } otherwise {
          bitCounter := bitCounter + 1
        }

      } otherwise {
        bitTimer := bitTimer + 1
      }
    }

    is(State.STOP) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        when(io.rx) {
          io.valid := True
        }

        state := State.IDLE

      } otherwise {
        bitTimer := bitTimer + 1
      }
    }
  }
}
```

여기서 `io.valid`는 **register가 아니라 combinational default + STOP 상태에서의 1-cycle pulse**입니다.

---

# 5. UART TX와 Stream을 연결하는 Adapter

우리 UART TX는:

```text
data + valid + busy
```

이고 Stream은:

```text
payload + valid + ready
```

입니다.

따라서 작은 adapter 하나가 필요합니다.

```text
Stream
 ┌──────────────┐
 │ valid        │
 │ payload      │
 │ ready        │
 └──────┬───────┘
        │
        ▼
 UartTxStreamAdapter
        │
        ├── data
        └── valid
             │
             ▼
           UartTx
```

코드:

```scala
import spinal.core._
import spinal.lib._

case class UartTxStreamAdapter() extends Component {

  val io = new Bundle {
    val input = slave Stream(Bits(8 bits))

    val txData  = out Bits(8 bits)
    val txValid = out Bool()
    val txBusy  = in Bool()
  }

  io.input.ready := !io.txBusy

  io.txData := io.input.payload

  io.txValid :=
    io.input.valid &&
    io.input.ready
}
```

여기서 중요한 점은:

```scala
io.input.ready := !io.txBusy
```

입니다.

즉 UART TX가 바쁘면:

```text
ready = 0
```

이 되어 CPU의 Stream source가 기다립니다.

따라서:

```text
CPU
 ↓
Stream
 ↓
UART TX
```

에서도 **backpressure가 발생합니다.**

---

# 6. 한쪽 Endpoint 만들기

이제 실제 핵심 Component입니다.

```scala
import spinal.core._
import spinal.lib._

case class UartEndpoint(
    clockFrequencyHz: Int,
    baudRate: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    // CPU -> UART
    val txInput =
      slave Stream(Bits(8 bits))

    // UART -> CPU
    val rxOutput =
      master Stream(Bits(8 bits))

    // Physical UART pins
    val tx =
      out Bool()

    val rx =
      in Bool()

    // Debug
    val rxOverflow =
      out Bool()

    val fifoOccupancy =
      out UInt(log2Up(fifoDepth + 1) bits)
  }

  // ============================================================
  // UART TX
  // ============================================================

  val uartTx =
    UartTx(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  val txAdapter =
    UartTxStreamAdapter()

  txAdapter.io.input << io.txInput

  uartTx.io.data  := txAdapter.io.txData
  uartTx.io.valid := txAdapter.io.txValid

  txAdapter.io.txBusy := uartTx.io.busy

  io.tx := uartTx.io.tx


  // ============================================================
  // UART RX
  // ============================================================

  val uartRx =
    UartRx(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  uartRx.io.rx := io.rx


  // ============================================================
  // Flow -> Stream
  // ============================================================

  val rxFlow =
    Flow(Bits(8 bits))

  rxFlow.valid   := uartRx.io.valid
  rxFlow.payload := uartRx.io.data

  val rxOverflow =
    Bool()

  val rxStream =
    rxFlow.toStream(rxOverflow)


  // ============================================================
  // SpinalHDL StreamFifo
  // ============================================================

  val fifo =
    StreamFifo(
      dataType = Bits(8 bits),
      depth = fifoDepth
    )

  fifo.io.push << rxStream


  // ============================================================
  // 3-stage Stream Pipeline
  // ============================================================

  val stage1 =
    fifo.io.pop.stage()

  val stage2 =
    stage1.stage()

  val stage3 =
    stage2.stage()


  // ============================================================
  // CPU RX
  // ============================================================

  io.rxOutput << stage3


  // ============================================================
  // Debug
  // ============================================================

  io.rxOverflow := rxOverflow
  io.fifoOccupancy := fifo.io.occupancy
}
```

이 Component가 이번 프로젝트의 핵심입니다.

---

# 7. 데이터 경로를 정확히 보자

A endpoint 내부는:

```text
                 UartEndpoint A
┌──────────────────────────────────────────────────────┐
│                                                      │
│ CPU TX                                               │
│   │                                                  │
│   ▼                                                  │
│ Stream                                               │
│   │                                                  │
│   ▼                                                  │
│ UartTxStreamAdapter                                  │
│   │                                                  │
│   ▼                                                  │
│ UartTx                                              │
│   │                                                  │
│   │ TXD                                              │
│   ▼                                                  │
│                                                      │
│              UART physical link                     │
│                                                      │
│   ▲                                                  │
│   │ RXD                                              │
│   │                                                  │
│ UartRx                                               │
│   │                                                  │
│   ▼                                                  │
│ Flow                                                 │
│   │                                                  │
│   ▼                                                  │
│ Flow → Stream                                         │
│   │                                                  │
│   ▼                                                  │
│ StreamFifo                                            │
│   │                                                  │
│   ▼                                                  │
│ stage 1                                               │
│   │                                                  │
│   ▼                                                  │
│ stage 2                                               │
│   │                                                  │
│   ▼                                                  │
│ stage 3                                               │
│   │                                                  │
│   ▼                                                  │
│ CPU RX                                                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

# 8. Duplex Loopback Top

이제 A와 B를 서로 연결합니다.

```scala
import spinal.core._
import spinal.lib._

case class UartDuplexSystem(
    clockFrequencyHz: Int,
    baudRate: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    val aTx =
      slave Stream(Bits(8 bits))

    val aRx =
      master Stream(Bits(8 bits))


    val bTx =
      slave Stream(Bits(8 bits))

    val bRx =
      master Stream(Bits(8 bits))


    val aOverflow =
      out Bool()

    val bOverflow =
      out Bool()

    val aFifoOccupancy =
      out UInt(log2Up(fifoDepth + 1) bits)

    val bFifoOccupancy =
      out UInt(log2Up(fifoDepth + 1) bits)
  }


  val a =
    UartEndpoint(
      clockFrequencyHz,
      baudRate,
      fifoDepth
    )

  val b =
    UartEndpoint(
      clockFrequencyHz,
      baudRate,
      fifoDepth
    )


  // CPU interfaces

  a.io.txInput << io.aTx
  io.aRx << a.io.rxOutput

  b.io.txInput << io.bTx
  io.bRx << b.io.rxOutput


  // ============================================================
  // FULL DUPLEX UART LINK
  // ============================================================

  // A TX -> B RX
  b.io.rx := a.io.tx

  // B TX -> A RX
  a.io.rx := b.io.tx


  // Debug

  io.aOverflow := a.io.rxOverflow
  io.bOverflow := b.io.rxOverflow

  io.aFifoOccupancy := a.io.fifoOccupancy
  io.bFifoOccupancy := b.io.fifoOccupancy
}
```

이제 구조는 정확히:

```text
                         FULL DUPLEX

             A                                  B

CPU-A TX ──> UartTx ────────────────> UartRx ──> FIFO
                                              │
                                              ▼
                                          Stage1
                                              │
                                              ▼
                                          Stage2
                                              │
                                              ▼
                                          Stage3
                                              │
CPU-B RX <────────────────────────────────────┘


CPU-A RX <────────────────────────────────────┐
                                              │
                                          Stage3
                                              ▲
                                          Stage2
                                              ▲
                                          Stage1
                                              ▲
                                             FIFO
                                              ▲
                                          UartRx
                                              ▲
CPU-B TX ──> UartTx ──────────────────────────┘
```

이게 이번 프로젝트의 **Full Duplex + FIFO + Pipeline**입니다.

---

# 9. 왜 `StreamFifo` 뒤에 Pipeline을 두었는가?

이것이 이번 프로젝트에서 중요한 설계 포인트입니다.

```text
UART RX
   │
   ▼
FIFO
   │
   ▼
Pipeline
   │
   ▼
CPU
```

FIFO가 먼저 데이터를 저장합니다.

그러면 CPU가:

```text
ready = 0
```

으로 멈춰도:

```text
Pipeline
   ↓
FIFO
```

가 어느 정도 데이터를 흡수할 수 있습니다.

SpinalHDL `StreamFifo`는 `push/pop` Stream을 제공하며 기본 설정에서는 `withAsyncRead=false`일 때 추가 transaction capacity와 latency 특성이 있으므로, 여기서는 **정확한 cycle 수보다 데이터 보존과 handshake 동작을 검증하는 것**을 우선하겠습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

---

# 10. 아주 중요한 UART 한계

여기서 반드시 기억해야 합니다.

우리 `UartRx`는:

```text
valid + data
```

를 내보내지만:

```text
ready
```

가 없습니다.

따라서:

```text
UART RX
   │
   ▼
Flow
   │
   ▼
Stream
```

사이에서 **UART 자체에 backpressure를 걸 수 없습니다.**

즉 FIFO가 완전히 가득 차서:

```text
FIFO ready = 0
```

이 되어도 UART가 계속 byte를 받아 버릴 수 있습니다.

그래서 이번 설계는:

```text
UART RX
   ↓
Flow → Stream
   ↓
FIFO
```

이고 FIFO가 감당할 수 있는 범위에서 backpressure를 흡수합니다.

overflow는 별도로:

```scala
io.rxOverflow
```

로 관찰합니다.

이것도 매우 중요한 실제 하드웨어 설계 경험입니다.

> **UART physical protocol에는 Stream backpressure가 없기 때문에, 충분한 RX buffering 또는 별도의 flow control이 필요합니다.**

---

# 11. 이제 시뮬레이션

이번에는 우리가 이전에 했던 것처럼 수동 `while` loop를 억지로 만들지 않겠습니다.

SpinalHDL이 제공하는:

```text
StreamDriver
StreamMonitor
StreamReadyRandomizer
ScoreboardInOrder
```

를 사용하겠습니다.

공식 문서에도 이 네 가지를 Stream simulation에 사용하는 예제가 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

---

# 12. Simulation code

파일:

```text
src/test/scala/UartDuplexSystemSim.scala
```

```scala
import spinal.core._
import spinal.core.sim._
import spinal.lib._
import spinal.lib.sim.{
  StreamDriver,
  StreamMonitor,
  StreamReadyRandomizer,
  ScoreboardInOrder
}

object UartDuplexSystemSim extends App {

  val clockFrequencyHz = 10_000_000
  val baudRate         = 1_000_000

  val fifoDepth = 8

  val testCount = 16


  val compiled =
    SimConfig
      .withWave
      .compile(
        UartDuplexSystem(
          clockFrequencyHz = clockFrequencyHz,
          baudRate = baudRate,
          fifoDepth = fifoDepth
        )
      )


  compiled.doSim("UART Duplex + FIFO + Pipeline") { dut =>

    println()
    println("==============================================")
    println(" UART DUPLEX + FIFO + PIPELINE TEST")
    println("==============================================")
    println(
      s"clock = ${clockFrequencyHz / 1000000} MHz"
    )
    println(
      s"baud  = ${baudRate / 1000000} MHz"
    )
    println(
      s"FIFO  = $fifoDepth"
    )
    println(
      s"bytes = $testCount / direction"
    )
    println("==============================================")
    println()


    dut.clockDomain.forkStimulus(
      period = 10
    )


    dut.io.aTx.valid #= false
    dut.io.aTx.payload #= 0

    dut.io.bTx.valid #= false
    dut.io.bTx.payload #= 0


    dut.clockDomain.waitSampling(5)


    // ============================================================
    // Scoreboards
    // ============================================================

    val aToB =
      ScoreboardInOrder[Int]()

    val bToA =
      ScoreboardInOrder[Int]()


    // ============================================================
    // A -> B
    // ============================================================

    StreamDriver(
      dut.io.aTx,
      dut.clockDomain
    ) { payload =>

      val index =
        aToB.matches

      if (index < testCount) {

        val value =
          0x40 + index

        payload #= value

        true

      } else {

        false
      }
    }


    // ============================================================
    // B -> A
    // ============================================================

    StreamDriver(
      dut.io.bTx,
      dut.clockDomain
    ) { payload =>

      val index =
        bToA.matches

      if (index < testCount) {

        val value =
          0xA0 + index

        payload #= value

        true

      } else {

        false
      }
    }


    // ============================================================
    // Reference monitors
    // ============================================================

    StreamMonitor(
      dut.io.aTx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[A -> B TX] 0x$value%02X"
      )

      aToB.pushRef(value)
    }


    StreamMonitor(
      dut.io.bTx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[B -> A TX] 0x$value%02X"
      )

      bToA.pushRef(value)
    }


    // ============================================================
    // Backpressure
    // ============================================================

    StreamReadyRandomizer(
      dut.io.aRx,
      dut.clockDomain
    )

    StreamReadyRandomizer(
      dut.io.bRx,
      dut.clockDomain
    )


    // ============================================================
    // RX monitors
    // ============================================================

    StreamMonitor(
      dut.io.aRx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[A RX    ] 0x$value%02X"
      )

      bToA.pushDut(value)
    }


    StreamMonitor(
      dut.io.bRx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[B RX    ] 0x$value%02X"
      )

      aToB.pushDut(value)
    }


    // ============================================================
    // Overflow monitor
    // ============================================================

    fork {

      while (true) {

        dut.clockDomain.waitSampling()

        assert(
          !dut.io.aOverflow.toBoolean,
          "A RX overflow detected"
        )

        assert(
          !dut.io.bOverflow.toBoolean,
          "B RX overflow detected"
        )
      }
    }


    // ============================================================
    // Wait until both directions are completely verified
    // ============================================================

    dut.clockDomain.waitActiveEdgeWhere {

      aToB.matches == testCount &&
      bToA.matches == testCount
    }


    println()
    println("==============================================")
    println(" A -> B : PASS")
    println(" B -> A : PASS")
    println(" FIFO + PIPELINE : PASS")
    println(" BACKPRESSURE    : ACTIVE")
    println(" OVERFLOW        : NONE")
    println("==============================================")
    println()


    simSuccess()
  }
}
```

---

# 13. 그런데 위 Simulation에는 한 가지 중요한 문제가 있습니다

여기서는 **의도적으로 잠깐 멈추겠습니다.**

`StreamDriver` callback에서:

```scala
val index = aToB.matches
```

를 이용해 다음 데이터를 선택하는 방식은 **reference scoreboard의 `matches`를 producer sequence counter로 사용하는 것은 좋은 설계가 아닙니다.**

왜냐하면:

```text
TX reference matches
```

는

```text
TX accepted count - RX checked count
```

라는 scoreboard 의미를 가질 수 있기 때문입니다.

즉 **producer의 sequence counter와 scoreboard counter를 분리해야 합니다.**

이것은 우리가 이전 FIFO/UART simulation에서 배운 것과 같은 종류의 subtle simulation issue를 피하기 위해 중요합니다.

따라서 최종본은 sequence counter를 별도로 둡니다.

---

# 14. 수정한 최종 Simulation

이 버전을 사용하세요.

```scala
import spinal.core._
import spinal.core.sim._
import spinal.lib._
import spinal.lib.sim.{
  StreamDriver,
  StreamMonitor,
  StreamReadyRandomizer,
  ScoreboardInOrder
}

object UartDuplexSystemSim extends App {

  val clockFrequencyHz = 10_000_000
  val baudRate         = 1_000_000

  val fifoDepth = 8

  val testCount = 16


  val compiled =
    SimConfig
      .withWave
      .compile(
        UartDuplexSystem(
          clockFrequencyHz = clockFrequencyHz,
          baudRate = baudRate,
          fifoDepth = fifoDepth
        )
      )


  compiled.doSim("UART Duplex + FIFO + Pipeline") { dut =>

    println()
    println("================================================")
    println(" UART TX/RX + FIFO + 3-STAGE PIPELINE")
    println(" FULL DUPLEX + BACKPRESSURE")
    println("================================================")


    dut.clockDomain.forkStimulus(
      period = 10
    )


    dut.io.aTx.valid #= false
    dut.io.aTx.payload #= 0

    dut.io.bTx.valid #= false
    dut.io.bTx.payload #= 0


    dut.clockDomain.waitSampling(5)


    // ============================================================
    // Scoreboards
    // ============================================================

    val aToB =
      ScoreboardInOrder[Int]()

    val bToA =
      ScoreboardInOrder[Int]()


    // ============================================================
    // Independent producer counters
    // ============================================================

    var aTxCount = 0
    var bTxCount = 0


    // ============================================================
    // A -> B producer
    // ============================================================

    StreamDriver(
      dut.io.aTx,
      dut.clockDomain
    ) { payload =>

      if (aTxCount < testCount) {

        val value =
          0x40 + aTxCount

        payload #= value

        aTxCount += 1

        true

      } else {

        false
      }
    }


    // ============================================================
    // B -> A producer
    // ============================================================

    StreamDriver(
      dut.io.bTx,
      dut.clockDomain
    ) { payload =>

      if (bTxCount < testCount) {

        val value =
          0xA0 + bTxCount

        payload #= value

        bTxCount += 1

        true

      } else {

        false
      }
    }


    // ============================================================
    // TX reference monitors
    // ============================================================

    StreamMonitor(
      dut.io.aTx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[A TX] -> B : 0x$value%02X"
      )

      aToB.pushRef(value)
    }


    StreamMonitor(
      dut.io.bTx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[B TX] -> A : 0x$value%02X"
      )

      bToA.pushRef(value)
    }


    // ============================================================
    // RX Backpressure
    // ============================================================

    StreamReadyRandomizer(
      dut.io.aRx,
      dut.clockDomain
    )

    StreamReadyRandomizer(
      dut.io.bRx,
      dut.clockDomain
    )


    // ============================================================
    // RX monitors
    // ============================================================

    StreamMonitor(
      dut.io.aRx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[A RX] <- B : 0x$value%02X"
      )

      bToA.pushDut(value)
    }


    StreamMonitor(
      dut.io.bRx,
      dut.clockDomain
    ) { payload =>

      val value =
        payload.toInt

      println(
        f"[B RX] <- A : 0x$value%02X"
      )

      aToB.pushDut(value)
    }


    // ============================================================
    // Overflow checker
    // ============================================================

    fork {

      while (true) {

        dut.clockDomain.waitSampling()

        assert(
          !dut.io.aOverflow.toBoolean,
          "A RX overflow detected"
        )

        assert(
          !dut.io.bOverflow.toBoolean,
          "B RX overflow detected"
        )
      }
    }


    // ============================================================
    // Timeout
    // ============================================================

    SimTimeout(100000)


    // ============================================================
    // Wait for both directions
    // ============================================================

    dut.clockDomain.waitActiveEdgeWhere {

      aToB.matches == testCount &&
      bToA.matches == testCount
    }


    // ============================================================
    // Final verification
    // ============================================================

    assert(
      aToB.matches == testCount,
      s"A -> B mismatch: matches=${aToB.matches}"
    )

    assert(
      bToA.matches == testCount,
      s"B -> A mismatch: matches=${bToA.matches}"
    )


    println()
    println("================================================")
    println("              FINAL RESULT")
    println("================================================")
    println(" A -> B : PASS")
    println(" B -> A : PASS")
    println(" FIFO   : PASS")
    println(" 3-STAGE PIPELINE : PASS")
    println(" BACKPRESSURE : ACTIVE")
    println(" RX OVERFLOW : NONE")
    println("================================================")
    println()


    simSuccess()
  }
}
```

---

# 15. `build.sbt`

기존 프로젝트 그대로 사용하면 됩니다.

```scala
ThisBuild / version := "0.1.0"
ThisBuild / scalaVersion := "2.13.16"

val spinalVersion = "1.15.0"

lazy val root = (project in file("."))
  .settings(
    name := "SpinalHDL-Learning",

    libraryDependencies ++= Seq(
      "com.github.spinalhdl" %% "spinalhdl-core" % spinalVersion,
      "com.github.spinalhdl" %% "spinalhdl-lib"  % spinalVersion
    ),

    Compile / compilerPlugin(
      "com.github.spinalhdl" %% "spinalhdl-idsl-plugin" % spinalVersion
    ),

    Compile / run / fork := true
  )
```

---

# 16. 파일 구조

이렇게 정리하면 좋겠습니다.

```text
SpinalHDL-Learning/
│
├── build.sbt
│
└── src/
    ├── main/
    │   └── scala/
    │       ├── UartTx.scala
    │       ├── UartRx.scala
    │       ├── UartTxStreamAdapter.scala
    │       ├── UartEndpoint.scala
    │       └── UartDuplexSystem.scala
    │
    └── test/
        └── scala/
            └── UartDuplexSystemSim.scala
```

---

# 17. 이 프로젝트에서 실제로 검증하는 것

단순히 `0x41` 하나 보내고 받는 테스트가 아닙니다.

### Test 1 — UART TX

A:

```text
40 41 42 43 ... 4F
```

B:

```text
A0 A1 A2 A3 ... AF
```

를 동시에 전송합니다.

---

### Test 2 — UART RX

A가 보낸 데이터가 B에서:

```text
40 41 42 ... 4F
```

순서 그대로 나와야 합니다.

B가 보낸 데이터는 A에서:

```text
A0 A1 A2 ... AF
```

순서 그대로 나와야 합니다.

---

### Test 3 — Full Duplex

동시에:

```text
A → B
B → A
```

가 진행됩니다.

즉:

```text
A TX ───────────────> B RX
A RX <─────────────── B TX
```

를 동시에 검증합니다.

---

### Test 4 — FIFO

RX 데이터는 바로 CPU로 가지 않습니다.

```text
UART RX
   ↓
Flow
   ↓
Stream
   ↓
StreamFifo
```

를 거칩니다.

---

### Test 5 — 3-stage Pipeline

FIFO 뒤에:

```text
Stage 1
   ↓
Stage 2
   ↓
Stage 3
```

가 있습니다.

각 `.stage()`는 `valid/payload`를 register stage로 잘라내는 1-cycle pipeline stage입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/Stream.html?utm_source=chatgpt.com)

---

### Test 6 — Backpressure

CPU RX에:

```text
ready = random
```

을 걸었습니다.

즉:

```text
CPU
 │
 │ ready = 0
 ▼
Pipeline
 │
 ▼
FIFO
```

가 stall될 수 있습니다.

SpinalHDL의 `StreamReadyRandomizer`는 바로 이 용도의 simulation utility입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

---

### Test 7 — Ordering

최종적으로 Scoreboard가:

```text
TX sequence
      ↓
reference queue
      ↓
RX sequence
```

를 비교합니다.

그래서 단순히 "개수가 16개다"가 아니라:

```text
A TX: 40 41 42 43 44 ...
B RX: 40 41 42 43 44 ...
```

의 **순서까지 검증**합니다.

---

# 18. 예상되는 콘솔 출력

정확한 interleaving은 simulation scheduling/randomizer에 따라 달라지므로 **아래는 형식 예시이지 제가 실제 실행했다고 주장하는 로그가 아닙니다.**

```text
================================================
 UART TX/RX + FIFO + 3-STAGE PIPELINE
 FULL DUPLEX + BACKPRESSURE
================================================

[A TX] -> B : 0x40
[B TX] -> A : 0xA0
[A TX] -> B : 0x41
[B TX] -> A : 0xA1
...

[A RX] <- B : 0xA0
[B RX] <- A : 0x40

[A TX] -> B : 0x42
[B TX] -> A : 0xA2

[A RX] <- B : 0xA1
[B RX] <- A : 0x41

...

================================================
              FINAL RESULT
================================================
 A -> B : PASS
 B -> A : PASS
 FIFO   : PASS
 3-STAGE PIPELINE : PASS
 BACKPRESSURE : ACTIVE
 RX OVERFLOW : NONE
================================================
```

**실행 전부터 "PASS"라고 단정하지 않는 이유도 중요합니다.** 특히 우리가 과거 UART simulation에서 겪었던 것처럼 SpinalSim의 scheduling/handshake 문제가 실제 실행에서 드러날 수 있기 때문입니다.

---

# 19. GTKWave에서 무엇을 봐야 하는가?

이번에는 VCD가 굉장히 교육적일 겁니다.

다음 신호들을 찾아보세요.

```text
a.io.tx
b.io.tx

a.uartRx.state
b.uartRx.state

a.fifo.io.push.valid
a.fifo.io.push.ready

a.fifo.io.pop.valid
a.fifo.io.pop.ready

a.fifo.io.occupancy

a.stage1.valid
a.stage1.ready

a.stage2.valid
a.stage2.ready

a.stage3.valid
a.stage3.ready

io.aRx.valid
io.aRx.ready

io.bRx.valid
io.bRx.ready
```

특히 이것을 보면 좋습니다.

```text
CPU RX ready = 0
       │
       ▼
stage3 stall
       │
       ▼
stage2 stall
       │
       ▼
stage1 stall
       │
       ▼
FIFO occupancy 증가
```

그리고 CPU가 다시:

```text
ready = 1
```

이 되면:

```text
FIFO
  ↓
stage1
  ↓
stage2
  ↓
stage3
  ↓
CPU
```

순서로 데이터가 빠져나갑니다.

이게 바로 우리가 앞에서 공부한 **Backpressure의 실제 waveform**입니다.

---

# 20. 이번 단계에서 아주 중요한 한 가지

이번 설계에서 **AXI-Stream이라는 말을 엄밀하게 구분**하겠습니다.

이번 프로젝트의 pipeline/FIFO 인터페이스는:

```text
SpinalHDL Stream
```

입니다.

즉:

```text
valid
ready
payload
```

의 handshake를 사용합니다.

SpinalHDL 공식 문서도 `Stream`을 UART controller request나 FIFO 등에 사용하는 기본 handshake interface로 설명합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

따라서 이번 단계는 정확히 말하면:

> **UART + SpinalHDL Stream + StreamFifo + 3-stage pipeline + Backpressure**

입니다.

아직 literal `Axi4Stream` library interface를 사용하는 프로젝트는 아닙니다.

이 구분을 해두는 것이 좋습니다.

---

# 21. 그리고 이 설계에서 우리가 배운 것이 전부 들어 있습니다

지금까지의 학습 흐름이 실제로 하나로 합쳐집니다.

```text
Project 5
Timer
 │
 ▼
Project 6
UART TX
 │
 ▼
Project 7
UART RX
 │
 ▼
UART Loopback
 │
 ▼
Simple FIFO
 │
 ▼
Stream
 │
 ▼
StreamFifo
 │
 ▼
Pipeline
 │
 ▼
Backpressure
 │
 ▼
       ★ 지금 ★
UART + FIFO + 3-stage Pipeline
+ Full Duplex
+ Simulation Scoreboard
```

그리고 다음 단계에서는 여기서 한 단계 더 올라갈 수 있습니다.

```text
                 ┌──────────────┐
                 │ CPU Emulator │
                 └──────┬───────┘
                        │
                     Stream
                        │
                        ▼
             ┌────────────────────┐
             │ UART TX            │
             └─────────┬──────────┘
                       │
                       ▼
                 SERIAL LINK
                       │
                       ▼
             ┌────────────────────┐
             │ UART RX            │
             └─────────┬──────────┘
                       │
                  StreamFifo
                       │
               ┌───────┴───────┐
               │               │
            Pipeline        Pipeline
               │               │
               └───────┬───────┘
                       │
                       ▼
                 CPU Emulator
```

**이번에는 먼저 이 버전을 실제로 컴파일/실행해서 
로그와 VCD를 확인하는 것이 좋습니다.** 특히 `StreamDriver`/`ScoreboardInOrder`가 현재 사용 중인 SpinalHDL 1.15.0에서 정확히 어떻게 동작하는지와, 
우리의 `UartTx/UartRx` 타이밍이 실제 simulation에서 맞물리는지를 직접 확인하면, 
다음 단계인 **CPU emulator + UART register interface + interrupt/FIFO 상태**로 넘어갈 때 훨씬 강해집니다.

---

---

네. 다시 면밀히 점검해 보니 **이전 답변의 코드는 그대로 실행하면 안 됩니다.** 특히 시뮬레이션 코드뿐 아니라 RTL에도 수정해야 할 부분이 있습니다.

제가 확인한 공식 SpinalHDL 문서의 `Stream`/`StreamFifo`/simulation API와, 우리가 앞서 만든 UART의 handshake 의미를 기준으로 다시 검토했습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

## 결론

이전 코드에서 특히 문제가 되는 부분은 **4개**입니다.

1. ❌ `UartTxStreamAdapter`에서 `txValid := input.valid && input.ready`
2. ⚠️ `Flow.toStream()`을 사용하면서 다시 `StreamFifo`를 붙인 구조의 의미가 불명확함
3. ❌ `StreamDriver`의 producer counter를 scoreboard와 연결한 방식
4. ⚠️ `StreamReadyRandomizer` 때문에 실제 UART 속도와 RX FIFO capacity 관계를 충분히 고려하지 않음

따라서 **코드를 새로 정리해서 검증 가능한 버전으로 만드는 것이 맞습니다.**

---

# 1. 가장 먼저 수정해야 하는 부분

이전 코드:

```scala
io.input.ready := !io.txBusy

io.txData := io.input.payload

io.txValid :=
  io.input.valid &&
  io.input.ready
```

이 부분은 좋지 않습니다.

왜냐하면 `Stream`의 `valid`는 `ready`에 의존하면 안 됩니다. 공식 Stream semantics에서도 `valid`가 `ready`에 combinational하게 의존하지 않도록 명시합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

특히:

```text
valid ──> ready
  ▲       │
  └───────┘
```

같은 combinational dependency가 생길 수 있습니다.

### 수정

```scala
io.input.ready := !io.txBusy

io.txData := io.input.payload
io.txValid := io.input.valid
```

즉:

```text
Stream
 valid ───────────────> UartTx.valid
 payload ─────────────> UartTx.data

 UartTx.busy ─────────> !Stream.ready
```

입니다.

그리고 실제 Stream transfer는:

```text
input.valid && input.ready
```

일 때 발생합니다.

그 순간 UART TX도:

```text
txValid = 1
txBusy  = 0
```

이므로 데이터를 받아들입니다.

---

# 2. 그런데 여기서 UART TX 자체의 인터페이스가 중요한 문제

우리 `UartTx`는:

```scala
val data  = in Bits(8 bits)
val valid = in Bool()
val tx    = out Bool()
val busy  = out Bool()
```

입니다.

즉 UART TX 자체는 `Stream`이 아닙니다.

하지만 현재 구조에서는:

```text
Stream
   │
   ▼
adapter
   │
   ▼
UART TX
```

이고 adapter가:

```scala
ready = !busy
```

를 만들어 줍니다.

따라서 이것은 논리적으로:

```text
CPU Stream
      │
      │ fire
      ▼
UART TX accepts byte
```

라는 정확한 구조가 됩니다.

---

# 3. `Flow.toStream()` 문제

이 부분도 다시 봐야 합니다.

제가 이전에:

```scala
val rxStream =
  rxFlow.toStream(rxOverflow)

val fifo =
  StreamFifo(
    dataType = Bits(8 bits),
    depth = fifoDepth
  )

fifo.io.push << rxStream
```

라고 했습니다.

그런데 `Flow.toStream(overflow)` 자체가 **Flow를 buffering 가능한 Stream으로 변환**합니다. 공식 API에도 `toStream`, `toStream(overflow)`, `toStream(overflow, fifoSize, overflowOccupancyAt)`가 존재합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/Flow.html?utm_source=chatgpt.com)

그러므로:

```text
UART RX
   ↓
Flow
   ↓
toStream()
   ↓
StreamFifo
```

가 되면 **FIFO가 두 개**가 됩니다.

교육적으로 불가능한 것은 아니지만 이번 프로젝트의 목표는:

> UART RX → **SpinalHDL StreamFifo** → Pipeline

을 명확하게 보여주는 것이므로 불필요하게 복잡합니다.

---

# 4. 그렇다면 Flow → Stream을 어떻게 할 것인가?

여기서 중요한 현실적인 문제입니다.

UART RX는:

```text
valid + data
```

만 있고:

```text
ready
```

가 없습니다.

따라서 그냥:

```scala
rxStream.valid := uartRx.io.valid
rxStream.payload := uartRx.io.data
```

하고

```scala
fifo.io.push << rxStream
```

하면 **FIFO가 full일 때 UART RX가 데이터를 잃을 수 있습니다.**

이것은 실제 UART 시스템에서 아주 중요한 문제입니다.

따라서 저는 이번 프로젝트에서는 `Flow.toStream()`을 **의도적으로 사용하겠습니다.**

즉:

```text
UART RX
   │
   ▼
Flow
   │
   ▼
Flow.toStream(...)
   │
   ▼
StreamFifo
   │
   ▼
3-stage pipeline
```

이렇게 해서:

- UART RX와 Stream 세계의 변환
- buffering
- overflow
- StreamFifo
- backpressure

를 모두 관찰할 수 있습니다.

단, 이것은 **FIFO가 2단계라는 것을 명확하게 주석으로 표시**해야 합니다.

---

# 5. 그런데 더 중요한 문제가 있습니다

우리가 원하는 것은:

```text
UART RX
   ↓
FIFO
   ↓
Pipeline
   ↓
CPU
```

입니다.

그런데 `Flow.toStream(overflow)`가 내부 FIFO를 만들기 때문에 실제로는:

```text
UART RX
   ↓
toStream internal FIFO
   ↓
StreamFifo
   ↓
Stage1
   ↓
Stage2
   ↓
Stage3
   ↓
CPU
```

입니다.

이것도 나쁘지 않습니다.

오히려 이번 학습에서는:

> **UART RX가 backpressure를 직접 받을 수 없는 문제를 어떻게 buffering으로 해결하는가**

를 보여주는 좋은 사례가 됩니다.

---

# 6. 3-stage pipeline 자체는 괜찮다

이 부분:

```scala
val stage1 = fifo.io.pop.stage()
val stage2 = stage1.stage()
val stage3 = stage2.stage()
```

은 올바른 방향입니다.

SpinalHDL의 `Stream.stage()`는 valid/payload path를 register stage로 끊고 latency 1 cycle을 갖는다고 공식 문서에 나옵니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

그리고 중요한 점:

```text
FIFO.pop
   ↓
stage1
   ↓
stage2
   ↓
stage3
```

각 stage가 **elastic**하게 동작하기 때문에 downstream backpressure가 upstream으로 전파됩니다.

---

# 7. `StreamFifo`도 확인했다

이전 코드:

```scala
StreamFifo(
  dataType = Bits(8 bits),
  depth = fifoDepth
)
```

형식은 현재 공식 API와 일치합니다. `StreamFifo`는 `push: slave Stream`, `pop: master Stream`, `occupancy`를 제공합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

그리고 중요한 특징:

```text
withAsyncRead = false
```

기본값에서는 read latency가 추가되고, 문서상 depth에 따라 추가 transaction capacity 특성도 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

따라서 이번 검증에서는:

> "FIFO depth가 정확히 N일 때 N개에서 full이 된다"

같은 cycle-정밀 capacity 검증을 하는 것이 아니라,

> **데이터 순서가 보존되고 backpressure가 정상 전파되는가**

를 검증하는 것이 맞습니다.

---

# 8. Simulation 코드에서 가장 위험했던 부분

이전:

```scala
StreamDriver(
  dut.io.aTx,
  dut.clockDomain
) { payload =>

  if (aTxCount < testCount) {

    val value = 0x40 + aTxCount

    payload #= value

    aTxCount += 1

    true
  } else {
    false
  }
}
```

여기서 `aTxCount += 1`의 위치가 마음에 걸렸던 것이 맞습니다.

`StreamDriver`는 testbench master 역할을 하며 callback을 통해 payload를 공급하고, StreamMonitor는 실제 `fire`된 transaction을 관찰합니다. 공식 예제도 `StreamDriver`와 `StreamMonitor`를 이렇게 사용합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

**검증의 기준은 callback 호출 횟수가 아니라 실제 Stream transfer입니다.**

따라서 producer sequence는 별도로 관리하고, scoreboard에는 **TX stream monitor에서 실제 fire된 데이터**를 넣는 것이 안전합니다.

이 부분은 우리가 이미 수정 방향을 잡았지만, 더 깔끔하게 만들겠습니다.

---

# 9. 더 중요한 simulation 문제: UART 전송 속도

현재:

```text
clock = 10 MHz
baud  = 1 MHz
```

이면:

```text
1 bit = 10 clock
1 byte = 10 bit
       = 100 clock
```

입니다.

16 bytes:

```text
16 × 100 = 1600 clocks
```

한 방향입니다.

Full duplex이므로 양쪽에서 동시에:

```text
A → B : 1600 clocks
B → A : 1600 clocks
```

입니다.

따라서:

```scala
SimTimeout(100000)
```

은 충분히 넉넉합니다.

---

# 10. Backpressure randomizer의 진짜 문제

여기서는 매우 중요한 것을 발견했습니다.

우리가:

```scala
StreamReadyRandomizer(dut.io.aRx, dut.clockDomain)
StreamReadyRandomizer(dut.io.bRx, dut.clockDomain)
```

를 사용하면 CPU RX가 랜덤하게 stall됩니다.

그런데 UART는 계속 데이터를 받습니다.

따라서:

```text
UART RX
   ↓
buffer
   ↓
FIFO
   ↓
pipeline
   ↓
random ready
```

구조에서 buffer capacity가 부족하면:

```text
RX overflow
```

가 발생할 수 있습니다.

그러면:

```text
Scoreboard mismatch
```

가 발생하는 것이 **RTL 오류가 아니라 의도된 overflow**일 수도 있습니다.

따라서 첫 번째 검증에서는 랜덤 ready를 그대로 쓰면 안 됩니다.

---

# 11. 검증을 두 단계로 나누자

이게 훨씬 좋은 학습 방법입니다.

## Test 1 — 기능 검증

```text
A → B
B → A

ready = True
```

먼저:

```text
UART
+
FIFO
+
Pipeline
+
Full Duplex
```

의 **기본 데이터 무결성**을 검증합니다.

---

## Test 2 — Backpressure 검증

그 다음:

```text
ready = random
```

을 적용합니다.

그리고 충분한 FIFO depth를 사용해서:

```text
overflow = false
```

를 검증합니다.

이렇게 해야 실패했을 때:

```text
UART timing 문제인가?
FIFO 문제인가?
pipeline 문제인가?
backpressure 문제인가?
simulation driver 문제인가?
```

를 분리할 수 있습니다.

이 방식이 우리가 이전 FIFO simulation에서 겪었던 문제를 피하는 데 훨씬 좋습니다.

---

# 12. 따라서 첫 실행용 FIFO는 8이 아니라 더 크게

우선:

```scala
val fifoDepth = 32
```

로 하겠습니다.

16 bytes만 전송하므로 충분한 buffer margin을 확보합니다.

그리고 나중에:

```text
depth = 8
depth = 4
depth = 2
```

로 줄여서 overflow 경계를 실험합니다.

---

# 13. 그리고 `StreamReadyRandomizer`는 두 번째 테스트에서 사용

### Test 1

```scala
dut.io.aRx.ready #= true
dut.io.bRx.ready #= true
```

### Test 2

```scala
StreamReadyRandomizer(
    dut.io.aRx,
    dut.clockDomain
)

StreamReadyRandomizer(
    dut.io.bRx,
    dut.clockDomain
)
```

이렇게 분리하는 것이 좋습니다.

---

# 14. 이번에 사용할 최종 검증 전략

제가 권장하는 것은 다음입니다.

### Phase A

```text
16 bytes A → B
16 bytes B → A

RX ready = 1

Expected:
A→B = 40 41 ... 4F
B→A = A0 A1 ... AF
```

### Phase B

```text
16 bytes A → B
16 bytes B → A

RX ready = random

FIFO depth = 32
```

Expected:

```text
A→B PASS
B→A PASS
overflow = false
ordering = PASS
```

### Phase C

```text
FIFO depth = 4 or 8
RX ready = aggressive stall
```

Expected:

```text
overflow may occur
```

그리고 이건 실패가 아니라 **의도적으로 overflow를 관찰하는 실험**입니다.

---

# 15. 또 하나: `StreamReadyRandomizer`와 `Scoreboard`

공식 예제에서는 정확히 다음 패턴을 사용합니다.

```scala
StreamReadyRandomizer(
    dut.io.pop,
    dut.clockDomain
)

StreamMonitor(
    dut.io.pop,
    dut.clockDomain
) { payload =>
    scoreboard.pushDut(payload.toInt)
}
```

즉 우리가 사용하려는 방법 자체는 올바른 방향입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

---

# 16. 최종적으로 수정해야 할 RTL

우선 TX adapter는 반드시:

```scala
case class UartTxStreamAdapter() extends Component {

  val io = new Bundle {
    val input = slave Stream(Bits(8 bits))

    val txData  = out Bits(8 bits)
    val txValid = out Bool()
    val txBusy  = in Bool()
  }

  io.input.ready := !io.txBusy

  io.txData  := io.input.payload
  io.txValid := io.input.valid
}
```

로 수정하겠습니다.

이것이 첫 번째 필수 수정입니다.

---

# 17. 그리고 RX 부분은 명시적으로 작성

```scala
val rxFlow = Flow(Bits(8 bits))

rxFlow.valid   := uartRx.io.valid
rxFlow.payload := uartRx.io.data

val rxOverflow = Bool()

val rxStream =
  rxFlow.toStream(rxOverflow)
```

공식 API에 `Flow.toStream(overflow: Bool)`이 존재하는 것은 확인했습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/Flow.html?utm_source=chatgpt.com)

그 뒤:

```scala
val fifo =
  StreamFifo(
    dataType = Bits(8 bits),
    depth = fifoDepth
  )

fifo.io.push << rxStream
```

그리고:

```scala
val stage1 = fifo.io.pop.stage()
val stage2 = stage1.stage()
val stage3 = stage2.stage()

io.rxOutput << stage3
```

입니다.

---

# 18. 중요한 학습 포인트 하나

이 프로젝트에서 우리는 이제 다음 세 가지를 구분할 수 있습니다.

### UART RX

```text
valid + payload
```

**Flow-like**

↓

### FIFO

```text
valid + payload + ready
```

**Stream**

↓

### CPU

```text
valid + payload + ready
```

**Stream**

즉:

```text
UART
  │
  │ Flow
  ▼
buffer
  │
  │ Stream
  ▼
FIFO
  │
  │ Stream
  ▼
Pipeline
  │
  │ Stream
  ▼
CPU
```

이것이 상당히 현실적인 데이터 경로입니다.

---

# 19. 이번 점검 결과

| 항목 | 이전 코드 | 판정 | 조치 |
|---|---|---|---|
| `UartTx` | 기존 구현 | ⚠️ | 기존 검증본 유지 |
| `UartRx` | majority voting | ⚠️ | 기존 검증본 유지 |
| TX Adapter `ready` | `!busy` | ✅ | 유지 |
| TX Adapter `valid` | `valid && ready` | ❌ | **`valid`로 수정** |
| `Flow.toStream()` | 사용 | ✅ | API 확인 |
| `StreamFifo` | 사용 | ✅ | API 확인 |
| `.stage()` ×3 | 사용 | ✅ | API 확인 |
| `StreamDriver` | 사용 | ✅ | 공식 패턴 확인 |
| `StreamMonitor` | 사용 | ✅ | 공식 패턴 확인 |
| `ScoreboardInOrder` | 사용 | ✅ | 공식 패턴 확인 |
| Random ready | 첫 테스트부터 사용 | ⚠️ | **2단계 테스트로 분리** |
| FIFO depth=8 | 첫 검증 | ⚠️ | **첫 테스트는 32 권장** |
| Timeout | 100000 | ✅ | 충분 |
| Full duplex | A↔B 동시 | ✅ | 구조적으로 적절 |
| TX/RX ordering | scoreboard | ✅ | 실제 fire 기준으로 검증 |
| overflow 검사 | 항상 false 기대 | ⚠️ | backpressure 테스트와 분리 |

---

## 결론

**지금 상태로는 로컬에서 실행하지 마세요.**

제가 이전 답변에서 "완성된 코드"라고 제시한 것은 너무 성급했습니다. 특히 사용자가 예전에 정확히 지적했던 **SpinalSim scheduling / handshake 문제를 다시 피하기 위해서라도**, 이번에는 먼저 RTL과 testbench를 한 번 더 정제해야 합니다.

그리고 공식 문서 기준으로는 `StreamDriver`, `StreamMonitor`, `StreamReadyRandomizer`, `ScoreboardInOrder`, `StreamFifo`, `.stage()`, `Flow.toStream()`의 존재와 기본 사용법은 확인되었습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/stream.html?utm_source=chatgpt.com)

**다음 버전에서는 제가 아예 `UartEndpoint.scala` + `UartDuplexSystem.scala` + `UartDuplexSystemSim.scala`를 처음부터 다시 작성해서,**

1. **Test-1: Full-duplex, no-backpressure**
2. **Test-2: Full-duplex + random backpressure**
3. **Test-3: 의도적인 FIFO overflow**

세 개를 독립 테스트로 만들고, 각 테스트의 **expected behavior까지 명확하게 정의한 뒤** 드리는 것이 맞습니다. 
이렇게 해야 사용자가 로컬에서 실행했을 때 로그 하나하나를 가지고 바로 RTL 학습을 진행할 수 있습니다.

---

---

좋습니다. 👍

그럼 다음 단계는 **이전 코드를 그대로 이어 붙이지 않고**, SpinalHDL **1.15.0 기준으로 전체 구조와 시뮬레이션을 다시 정리해서** 진행하겠습니다.

### 진행 순서

1. **UART + FIFO + Pipeline 통합 구조 확정**
   ```text
   A TX
      │
      ▼
   ┌────────┐
   │ UART TX│
   └────┬───┘
        │ serial
        ▼
   ┌────────┐
   │ UART RX│
   └────┬───┘
        │ Flow
        ▼
   Flow.toStream()
        │
        ▼
   StreamFifo
        │
        ▼
   Pipeline
        │
        ▼
      A RX
   ```

   동시에 반대 방향도 똑같이 동작시켜서 **Full Duplex**를 구성합니다.

2. **SpinalHDL 제공 컴포넌트 적극 사용**
   - `Stream`
   - `Flow`
   - `StreamFifo`
   - `.stage()`
   - `StreamDriver`
   - `StreamMonitor`
   - `StreamReadyRandomizer`
   - `ScoreboardInOrder`

3. **테스트를 세 단계로 분리**
   - **Test 1:** Full Duplex + Backpressure 없음
   - **Test 2:** Full Duplex + Random Backpressure
   - **Test 3:** 의도적인 FIFO Overflow 실험

4. 특히 시뮬레이션은
   - `waitSampling()` / `waitFallingEdge()` 문제
   - Stream `valid/ready/fire`
   - UART TX의 `busy`
   - RX의 `valid` 1-cycle pulse
   - `Flow → Stream` 변환
   - FIFO가 가득 찼을 때 데이터 손실
   - scoreboard가 실제 handshake를 기준으로 데이터를 추적하는지

   를 하나씩 검증하겠습니다.

5. **PASS라고 미리 가정하지 않겠습니다.**
   
   먼저 코드를 완성한 뒤 실제로 컴파일/시뮬레이션을 돌려보고, 문제가 나오면 그 로그를 기준으로 수정하는 방식으로 진행하겠습니다.

다음 답변에서는 바로 **프로젝트 디렉터리 구조 → 각 Scala 파일 전체 코드 → 시뮬레이션 코드 → 실행 명령 → 각 테스트에서 기대하는 로그** 순서로 정리해서 시작하겠습니다.
