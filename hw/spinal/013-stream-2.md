# 좋습니다. 이제 이 구조를 실제로 만들어 봅시다.

우리가 목표로 하는 것은 **UART ↔ FIFO ↔ CPU 사이를 `Stream`으로 연결하는 구조**입니다.

```text
RX 방향

Serial RX
   │
   ▼
┌─────────┐
│ UartRx  │
└────┬────┘
     │ byte
     ▼
┌────────────┐
│ StreamFifo │
└─────┬──────┘
      │ Stream
      ▼
     CPU


TX 방향

     CPU
      │ Stream
      ▼
┌────────────┐
│ StreamFifo │
└─────┬──────┘
      │ byte
      ▼
┌─────────┐
│ UartTx  │
└────┬────┘
     │
     ▼
Serial TX
```

이번에는 **한 번에 전체를 구현하지 않고**, IP 블록 사이의 인터페이스부터 정확하게 만들겠습니다.

---

# 1. 먼저 중요한 설계 결정

RX와 TX는 약간 다르게 처리해야 합니다.

### RX

UART RX는 한 바이트를 받으면 `valid`를 발생시킵니다.

```text
UART RX
   │
   │ valid + data
   ▼
FIFO
```

UART는 이미 들어오는 serial frame을 받아야 하기 때문에 FIFO가 full이라고 해서 현재 수신 중인 UART frame을 멈출 수 없습니다.

따라서 RX 쪽은:

```text
UART RX → FIFO
```

에서 FIFO가 **버퍼 역할**을 합니다.

---

### TX

TX는 반대로 Stream을 아주 자연스럽게 사용할 수 있습니다.

```text
FIFO → UART TX
```

UART TX가 BUSY이면:

```text
ready = 0
```

UART TX가 IDLE이면:

```text
ready = 1
```

그리고:

```text
valid && ready
```

일 때 한 바이트를 받아서 전송을 시작합니다.

---

# 2. 먼저 `StreamFifo`를 인터페이스 블록으로 생각하자

우리가 만든 `StreamFifo`의 인터페이스는:

```scala
val io = new Bundle {
  val input  = slave Stream(Bits(dataWidth bits))
  val output = master Stream(Bits(dataWidth bits))
}
```

입니다.

따라서 외부에서 보면 FIFO는 단순히:

```text
             Stream
               │
               ▼
        ┌────────────┐
        │ StreamFifo │
        └────────────┘
               │
               ▼
             Stream
```

입니다.

FIFO 내부의:

```text
memory
writePtr
readPtr
count
```

는 외부 IP에서는 신경 쓸 필요가 없습니다.

이것이 **IP 블록화**의 중요한 개념입니다.

---

# 3. RX 쪽부터 만들자

먼저:

```text
UartRx → StreamFifo
```

를 연결합니다.

현재 `UartRx`는:

```scala
rx.io.data
rx.io.valid
```

를 제공합니다.

FIFO 입력은:

```scala
fifo.io.input.valid
fifo.io.input.payload
fifo.io.input.ready
```

입니다.

따라서 기본 연결은:

```scala
fifo.io.input.valid   := rx.io.valid
fifo.io.input.payload := rx.io.data
```

입니다.

그림으로 보면:

```text
UartRx                       StreamFifo

rx.valid ──────────────────► input.valid

rx.data  ──────────────────► input.payload

                             input.ready
                                 │
                                 X
                                 │
                         현재 UartRx에는
                         ready 입력이 없음
```

여기서 우리가 앞에서 이야기했던 UART RX의 특성이 나타납니다.

---

# 4. RX에서 FIFO `ready`는 당장 사용하지 않는다

현재 교육용 `UartRx`는:

```scala
val io = new Bundle {
    val rx = in Bool()
    val data = out Bits(8 bits)
    val valid = out Bool()
}
```

이므로 `ready` 입력이 없습니다.

따라서:

```scala
fifo.io.input.ready
```

를 UART RX에 연결할 수 없습니다.

대신 FIFO가 full이면 `input.fire`가 발생하지 않습니다.

```text
UART RX
valid = 1
data  = 0x41

FIFO
ready = 0

----------------
fire = 0
```

그러면 해당 byte는 FIFO에 저장되지 않습니다.

즉:

```text
FIFO overflow
      ↓
byte loss
```

가 발생할 수 있습니다.

**이것은 현재 설계의 의도적인 한계입니다.**

나중에 overflow flag 또는 더 정교한 RX buffering을 추가할 수 있습니다.

---

# 5. 이제 TX 쪽을 만들자

TX는 훨씬 깔끔합니다.

우리가 원하는 인터페이스:

```scala
val io = new Bundle {
    val input = slave Stream(Bits(8 bits))
    val tx    = out Bool()
}
```

입니다.

그러면 외부에서는:

```text
             Stream
CPU/FIFO ─────────────► UartTx
```

가 됩니다.

---

# 6. UART TX의 `ready`

UART TX가 IDLE이면 새 byte를 받을 수 있습니다.

따라서:

```scala
io.input.ready := state === State.IDLE
```

라는 구조를 사용할 수 있습니다.

즉:

```text
State.IDLE
    ↓
ready = 1

State.START
    ↓
ready = 0

State.DATA
    ↓
ready = 0

State.STOP
    ↓
ready = 0
```

이렇게 됩니다.

---

# 7. 그리고 `fire`에서 전송 시작

가장 중요한 부분:

```scala
when(io.input.fire) {
    shiftReg := io.input.payload
    state := State.START
}
```

입니다.

의미는:

> "CPU/FIFO가 보낸 byte를 UART TX가 실제로 받아들였다."

입니다.

즉 `fire`가 **IP 블록 사이의 실제 transaction**이 됩니다.

---

# 8. `UartTxStream` 만들기

SpinalHDL 1.15.0 기준으로 교육용 UART TX를 Stream 인터페이스로 바꿔보겠습니다.

```scala
import spinal.core._

case class UartTxStream(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {
    val input = slave Stream(Bits(8 bits))
    val tx    = out Bool()
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

  // --------------------------------
  // Stream ready
  // --------------------------------

  io.input.ready :=
    state === State.IDLE

  // --------------------------------
  // Default TX level
  // --------------------------------

  io.tx := True

  // --------------------------------
  // Bit timer
  // --------------------------------

  when(bitTimer === cyclesPerBit - 1) {
    bitTimer := 0
  } otherwise {
    bitTimer := bitTimer + 1
  }

  // --------------------------------
  // State machine
  // --------------------------------

  switch(state) {

    is(State.IDLE) {

      bitTimer   := 0
      bitCounter := 0

      when(io.input.fire) {

        shiftReg :=
          io.input.payload

        state :=
          State.START
      }
    }

    is(State.START) {

      io.tx := False

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer   := 0
        bitCounter := 0

        state :=
          State.DATA
      }
    }

    is(State.DATA) {

      io.tx :=
        shiftReg(0)

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        shiftReg :=
          shiftReg |>> 1

        when(bitCounter === 7) {

          state :=
            State.STOP

        } otherwise {

          bitCounter :=
            bitCounter + 1
        }
      }
    }

    is(State.STOP) {

      io.tx := True

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        state :=
          State.IDLE
      }
    }
  }
}
```

---

# 9. 이 코드에서 가장 중요한 세 부분

### 첫 번째

```scala
io.input.ready := state === State.IDLE
```

UART TX가 받을 수 있는 상태를 Stream의 `ready`로 표현합니다.

---

### 두 번째

```scala
when(io.input.fire) {
```

여기가 **새 byte를 실제로 받는 순간**입니다.

---

### 세 번째

```scala
shiftReg := io.input.payload
```

전달된 byte를 UART TX 내부에 저장합니다.

즉:

```text
input.fire
   ↓
payload 저장
   ↓
START
   ↓
DATA
   ↓
STOP
   ↓
IDLE
```

입니다.

---

# 10. 이제 TX FIFO와 연결

이제:

```text
CPU
 │
 │ Stream
 ▼
StreamFifo
 │
 │ Stream
 ▼
UartTxStream
```

를 만들 수 있습니다.

예를 들어 top-level:

```scala
import spinal.core._

case class UartTxPath(
    clockFrequencyHz: Int,
    baudRate: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    // CPU → FIFO
    val txInput =
      slave Stream(Bits(8 bits))

    // UART serial output
    val tx =
      out Bool()
  }

  val fifo =
    StreamFifo(
      dataWidth = 8,
      depth = fifoDepth
    )

  val tx =
    UartTxStream(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  // CPU → FIFO

  fifo.io.input <<
    io.txInput

  // FIFO → UART TX

  tx.io.input <<
    fifo.io.output

  // Serial TX

  io.tx :=
    tx.io.tx
}
```

이것이 상당히 중요한 코드입니다.

---

# 11. 연결을 자세히 보면

CPU가:

```text
txInput.valid
txInput.payload
```

을 발생시킵니다.

FIFO가:

```text
txInput.ready
```

를 제공합니다.

따라서:

```text
CPU
 │
 │ valid
 │ payload
 ▼
FIFO
 ▲
 │ ready
```

입니다.

FIFO에서 UART TX로는:

```text
FIFO output.valid
FIFO output.payload
```

가 UART TX로 갑니다.

UART TX가:

```text
tx.input.ready
```

를 제공합니다.

```text
FIFO
 │
 │ valid
 │ payload
 ▼
UART TX
 ▲
 │ ready
```

입니다.

---

# 12. 실제 데이터가 이동하는 과정

CPU가 `0x41`을 보내려고 합니다.

먼저:

```text
CPU → FIFO
```

에서:

```text
valid = 1
payload = 0x41
ready = 1
```

이면:

```text
fire = 1
```

입니다.

따라서 `0x41`이 FIFO에 저장됩니다.

---

그 다음:

```text
FIFO → UART TX
```

에서 FIFO가:

```text
valid = 1
payload = 0x41
```

를 출력합니다.

UART TX가 IDLE이므로:

```text
ready = 1
```

입니다.

따라서:

```text
fire = 1
```

입니다.

UART TX가:

```text
0x41
```

을 받아서 전송을 시작합니다.

---

# 13. UART TX가 전송 중이면?

UART TX가 START/DATA/STOP 상태가 되면:

```text
ready = 0
```

입니다.

그러면 FIFO는:

```text
valid = 1
```

을 계속 유지할 수 있습니다.

예를 들어 FIFO의 head가 `0x42`라면:

```text
cycle 1:
valid=1 ready=0 payload=42

cycle 2:
valid=1 ready=0 payload=42

cycle 3:
valid=1 ready=0 payload=42

...

UART TX가 IDLE

cycle N:
valid=1 ready=1 payload=42
fire=1
```

그때 `0x42`가 UART TX로 전달됩니다.

이것이 바로 **Stream backpressure가 IP 블록 사이에서 실제로 사용되는 모습**입니다.

---

# 14. 전체 TX 경로

이제 구조가 정확히 이렇게 됩니다.

```text
                  TX 방향

       ┌─────────┐
       │   CPU   │
       └────┬────┘
            │
            │ Stream
            │ valid
            │ payload
            ▼
      ┌─────────────┐
      │ StreamFifo  │
      └──────┬──────┘
             │
             │ Stream
             │ valid
             │ payload
             ▼
       ┌───────────┐
       │ UartTx    │
       │   Stream  │
       └─────┬─────┘
             │
             ▼
          Serial TX
```

그리고 `ready`는:

```text
Serial TX
    │
    ▼
 UartTx
    │ ready
    ▼
StreamFifo
    │ ready
    ▼
   CPU
```

방향으로 전달됩니다.

---

# 15. RX도 같은 개념으로 생각하기

RX는:

```text
Serial RX
    │
    ▼
 UartRx
    │
    │ valid + data
    ▼
StreamFifo
    │
    │ Stream
    ▼
   CPU
```

입니다.

CPU가 FIFO를 읽을 때:

```text
CPU ready = 1
```

이고 FIFO가 비어 있지 않으면:

```text
FIFO valid = 1
```

따라서:

```text
FIFO output.fire
```

가 발생합니다.

그러면 FIFO에서 byte 하나가 빠집니다.

---

# 16. 결국 우리가 배우고 있는 것은 이것입니다

`Stream`은 단순히 편리한 문법이 아닙니다.

**IP 블록 사이의 데이터 전달 규칙을 표준화하는 인터페이스**입니다.

예:

```text
CPU
 │
 │ Stream
 ▼
FIFO
 │
 │ Stream
 ▼
UART TX
```

각 IP는 상대방 내부 구조를 몰라도 됩니다.

CPU는:

> "나는 byte가 있다."

FIFO는:

> "공간이 있으면 받겠다."

UART TX는:

> "전송 가능할 때 받겠다."

그리고:

```text
valid && ready
```

가 되는 순간 실제 데이터가 이동합니다.

---

# 다음 단계

이제 **TX 경로는 구조적으로 완성**했습니다.

다음에는 이 구조를 조금 더 현실적으로 만들어서:

```text
CPU
 │
 ▼
TX StreamFifo
 │
 ▼
UartTxStream
 │
 ▼
Serial
```

에서 **FIFO에 여러 byte (`A → B → C`)를 넣었을 때 UART가 어떻게 순서대로 꺼내가는지** 살펴보겠습니다.

그 다음 RX까지 합쳐서 최종적으로:

```text
              ┌─────────────┐
Serial RX ───►│    UartRx   │
              └──────┬──────┘
                     ▼
              ┌─────────────┐
              │  RX FIFO    │──────► CPU
              └─────────────┘


CPU ─────────► TX FIFO
              │
              ▼
         ┌─────────────┐
         │ UartTxStream│
         └──────┬──────┘
                ▼
             Serial TX
```

형태의 **UART 통신 블록 전체 구조**로 완성하겠습니다.

---

---

# 좋습니다. 이번 단계에서는 **전체 구성품을 완성하고, 하나의 통합 시뮬레이션으로 검증**하겠습니다.

이번에는 중간 설명보다 **실제로 동작하는 최종 구조**에 집중하겠습니다.

목표:

```text
                    UART FULL DUPLEX

       RX 방향                         TX 방향

Serial RX                              Serial TX
    │                                       ▲
    ▼                                       │
┌─────────┐                           ┌─────────┐
│ UartRx  │                           │ UartTx  │
└────┬────┘                           └────▲────┘
     │                                    │
     ▼                                    │
┌────────────┐                       ┌────────────┐
│  RX FIFO   │                       │  TX FIFO   │
└─────┬──────┘                       └─────▲──────┘
      │                                    │
      ▼                                    │
     CPU ──────────────────────────────────┘
```

그리고 테스트에서는 **CPU가 TX FIFO에 A/B/C를 넣고 → UART로 전송 → 다시 UART RX로 수신 → RX FIFO에서 A/B/C를 읽는 loopback**을 하겠습니다.

---

# 1. 최종 구성품

이번 단계에서 사용할 것은 5개입니다.

```text
1. UartTxStream
2. UartRx
3. StreamFifo
4. UartFullDuplex
5. UartFullDuplexSim
```

모두 **SpinalHDL 1.15.0** 기준입니다.

---

# 2. `StreamFifo`

먼저 FIFO입니다.

```scala
import spinal.core._

case class StreamFifo(
    dataWidth: Int,
    depth: Int
) extends Component {

  require(dataWidth >= 1)
  require(depth >= 2)

  val io = new Bundle {
    val input  = slave Stream(Bits(dataWidth bits))
    val output = master Stream(Bits(dataWidth bits))
  }

  val mem = Vec(
    Reg(Bits(dataWidth bits)) init(0),
    depth
  )

  val ptrWidth =
    log2Up(depth)

  val writePtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val readPtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val countWidth =
    log2Up(depth + 1)

  val count =
    Reg(UInt(countWidth bits)) init(0)

  val empty =
    count === 0

  val full =
    count === depth

  // Input side
  io.input.ready := !full

  // Output side
  io.output.valid := !empty
  io.output.payload := mem(readPtr)

  // Actual transfers
  val push =
    io.input.fire

  val pop =
    io.output.fire

  // Write
  when(push) {

    mem(writePtr) :=
      io.input.payload

    when(writePtr === depth - 1) {
      writePtr := 0
    } otherwise {
      writePtr := writePtr + 1
    }
  }

  // Read
  when(pop) {

    when(readPtr === depth - 1) {
      readPtr := 0
    } otherwise {
      readPtr := readPtr + 1
    }
  }

  // Occupancy
  when(push && !pop) {
    count := count + 1
  } elsewhen(!push && pop) {
    count := count - 1
  }
}
```

핵심은 여전히 이것입니다.

```scala
val push = io.input.fire
val pop  = io.output.fire
```

그리고:

```text
input.fire  → FIFO에 넣음
output.fire → FIFO에서 꺼냄
```

---

# 3. `UartTxStream`

기존 UART TX에 Stream 입력을 붙입니다.

```scala
import spinal.core._

case class UartTxStream(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {
    val input = slave Stream(Bits(8 bits))
    val tx    = out Bool()
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

  // --------------------------------
  // Stream interface
  // --------------------------------

  io.input.ready :=
    state === State.IDLE

  // UART idle level
  io.tx := True

  // --------------------------------
  // Bit timer
  // --------------------------------

  when(bitTimer === cyclesPerBit - 1) {
    bitTimer := 0
  } otherwise {
    bitTimer := bitTimer + 1
  }

  // --------------------------------
  // UART state machine
  // --------------------------------

  switch(state) {

    is(State.IDLE) {

      bitTimer   := 0
      bitCounter := 0

      when(io.input.fire) {

        shiftReg :=
          io.input.payload

        state :=
          State.START
      }
    }

    is(State.START) {

      io.tx := False

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer   := 0
        bitCounter := 0

        state :=
          State.DATA
      }
    }

    is(State.DATA) {

      io.tx :=
        shiftReg(0)

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        shiftReg :=
          shiftReg |>> 1

        when(bitCounter === 7) {

          state :=
            State.STOP

        } otherwise {

          bitCounter :=
            bitCounter + 1
        }
      }
    }

    is(State.STOP) {

      io.tx := True

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        state :=
          State.IDLE
      }
    }
  }
}
```

---

# 4. `UartRx`

RX는 기존에 검증했던 구조를 그대로 사용합니다.

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

  object State extends SpinalEnum {
    val IDLE, START, DATA, STOP = newElement()
  }

  val state =
    RegInit(State.IDLE)

  io.data  := shiftReg
  io.valid := False

  switch(state) {

    is(State.IDLE) {

      bitTimer   := 0
      bitCounter := 0

      when(!io.rx) {
        bitTimer := 0
        state := State.START
      }
    }

    is(State.START) {

      when(bitTimer === halfBitCount - 1) {

        bitTimer := 0

        when(!io.rx) {
          bitCounter := 0
          state := State.DATA
        } otherwise {
          state := State.IDLE
        }

      } otherwise {

        bitTimer :=
          bitTimer + 1
      }
    }

    is(State.DATA) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        shiftReg(bitCounter) :=
          io.rx

        when(bitCounter === 7) {
          state := State.STOP
        } otherwise {
          bitCounter :=
            bitCounter + 1
        }

      } otherwise {

        bitTimer :=
          bitTimer + 1
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

        bitTimer :=
          bitTimer + 1
      }
    }
  }
}
```

여기서 다시 중요한 점:

```scala
io.valid := False
```

는 기본값이고,

```scala
when(io.rx) {
    io.valid := True
}
```

가 조건부 override입니다.

따라서 `valid`는 수신 완료 시점에 **1클럭 pulse**가 됩니다.

---

# 5. 이제 최종 `UartFullDuplex`

이것이 오늘의 핵심입니다.

CPU 쪽에는 RX/TX 각각 Stream 인터페이스를 제공합니다.

```scala
import spinal.core._

case class UartFullDuplex(
    clockFrequencyHz: Int,
    baudRate: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    // --------------------------------
    // CPU → UART TX
    // --------------------------------

    val txInput =
      slave Stream(Bits(8 bits))

    // --------------------------------
    // UART RX → CPU
    // --------------------------------

    val rxOutput =
      master Stream(Bits(8 bits))

    // --------------------------------
    // Physical UART pins
    // --------------------------------

    val uartTx =
      out Bool()

    val uartRx =
      in Bool()
  }

  // ============================================================
  // TX PATH
  // CPU → TX FIFO → UartTxStream → uartTx
  // ============================================================

  val txFifo =
    StreamFifo(
      dataWidth = 8,
      depth = fifoDepth
    )

  val tx =
    UartTxStream(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  txFifo.io.input <<
    io.txInput

  tx.io.input <<
    txFifo.io.output

  io.uartTx :=
    tx.io.tx

  // ============================================================
  // RX PATH
  // uartRx → UartRx → RX FIFO → CPU
  // ============================================================

  val rx =
    UartRx(
      clockFrequencyHz = clockFrequencyHz,
      baudRate = baudRate
    )

  val rxFifo =
    StreamFifo(
      dataWidth = 8,
      depth = fifoDepth
    )

  rx.io.rx :=
    io.uartRx

  rxFifo.io.input.valid :=
    rx.io.valid

  rxFifo.io.input.payload :=
    rx.io.data

  io.rxOutput <<
    rxFifo.io.output
}
```

---

# 6. 최종 구조를 다시 보면

```text
                         UartFullDuplex
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│ TX PATH                                                     │
│                                                             │
│ txInput                                                     │
│   │                                                         │
│   ▼                                                         │
│ ┌─────────────┐      ┌─────────────┐                        │
│ │   TX FIFO   │─────►│ UartTxStream│──────► uartTx         │
│ └─────────────┘      └─────────────┘                        │
│                                                             │
│                                                             │
│ RX PATH                                                     │
│                                                             │
│ uartRx ───► ┌─────────┐      ┌─────────────┐                │
│             │ UartRx  │─────►│   RX FIFO   │──────► rxOutput│
│             └─────────┘      └─────────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

여기서 **CPU와 FIFO 사이에는 Stream**이 사용됩니다.

```text
CPU
 │
 │ Stream
 ▼
TX FIFO
 │
 │ Stream
 ▼
UART TX
```

그리고:

```text
UART RX
 │
 │ byte event
 ▼
RX FIFO
 │
 │ Stream
 ▼
CPU
```

입니다.

---

# 7. 통합 Loopback 시뮬레이션

이제 가장 중요한 부분입니다.

실제 외부 UART 장치 대신:

```scala
dut.io.uartRx #= dut.io.uartTx
```

로 내부 loopback을 만듭니다.

즉:

```text
TX FIFO
   ↓
UartTx
   ↓
uartTx
   │
   └──────────┐
              ▼
           uartRx
              │
              ▼
           UartRx
              ↓
           RX FIFO
```

가 됩니다.

---

## `UartFullDuplexSim`

```scala
import spinal.core._
import spinal.core.sim._

object UartFullDuplexSim extends App {

  val clockFrequencyHz = 10_000_000
  val baudRate         = 1_000_000
  val fifoDepth        = 4

  val cyclesPerBit =
    clockFrequencyHz / baudRate

  val frameCycles =
    cyclesPerBit * 10

  SimConfig
    .withWave
    .doSim(
      UartFullDuplex(
        clockFrequencyHz = clockFrequencyHz,
        baudRate = baudRate,
        fifoDepth = fifoDepth
      )
    ) { dut =>

      dut.clockDomain.forkStimulus(10)

      // --------------------------------------------------
      // Initial values
      // --------------------------------------------------

      dut.io.txInput.valid #= false
      dut.io.txInput.payload #= 0

      dut.io.rxOutput.ready #= false

      // UART idle
      dut.io.uartRx #= true

      dut.clockDomain.waitSampling(2)

      println(
        "[Initial] " +
        s"txReady=${dut.io.txInput.ready.toBoolean} " +
        s"rxValid=${dut.io.rxOutput.valid.toBoolean}"
      )

      // --------------------------------------------------
      // Connect TX serial output to RX serial input
      // --------------------------------------------------

      fork {
        while (true) {
          dut.io.uartRx #=
            dut.io.uartTx.toBoolean

          dut.clockDomain.waitSampling()
        }
      }

      // --------------------------------------------------
      // Send one byte into TX Stream
      // --------------------------------------------------

      def sendByte(data: Int): Unit = {

        dut.io.txInput.payload #=
          data

        dut.io.txInput.valid #=
          true

        // Wait until the Stream transaction occurs.
        while (!dut.io.txInput.ready.toBoolean) {
          dut.clockDomain.waitSampling()
        }

        dut.clockDomain.waitSampling()

        dut.io.txInput.valid #=
          false

        println(
          f"[CPU TX] accepted 0x$data%02X"
        )
      }

      // --------------------------------------------------
      // Send A, B, C
      // --------------------------------------------------

      sendByte(0x41)
      sendByte(0x42)
      sendByte(0x43)

      println(
        "[CPU TX] A -> B -> C inserted into TX FIFO"
      )

      // --------------------------------------------------
      // Wait for UART transmission and RX
      //
      // 3 frames + margin
      // --------------------------------------------------

      dut.clockDomain.waitSampling(
        frameCycles * 3 + 20
      )

      println(
        "[UART] transmission / reception wait complete"
      )

      // --------------------------------------------------
      // RX FIFO should now contain A, B, C
      // --------------------------------------------------

      assert(
        dut.io.rxOutput.valid.toBoolean,
        "RX FIFO should contain received data"
      )

      println(
        "[RX] data available"
      )

      // --------------------------------------------------
      // Read A
      // --------------------------------------------------

      assert(
        dut.io.rxOutput.payload.toInt == 0x41,
        f"Expected A (0x41), got 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      println(
        f"[CPU RX] received 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      dut.io.rxOutput.ready #=
        true

      dut.clockDomain.waitSampling()

      // --------------------------------------------------
      // Read B
      // --------------------------------------------------

      assert(
        dut.io.rxOutput.valid.toBoolean,
        "RX FIFO should contain B"
      )

      assert(
        dut.io.rxOutput.payload.toInt == 0x42,
        f"Expected B (0x42), got 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      println(
        f"[CPU RX] received 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      dut.clockDomain.waitSampling()

      // --------------------------------------------------
      // Read C
      // --------------------------------------------------

      assert(
        dut.io.rxOutput.valid.toBoolean,
        "RX FIFO should contain C"
      )

      assert(
        dut.io.rxOutput.payload.toInt == 0x43,
        f"Expected C (0x43), got 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      println(
        f"[CPU RX] received 0x${dut.io.rxOutput.payload.toInt}%02X"
      )

      dut.clockDomain.waitSampling()

      // --------------------------------------------------
      // FIFO should now be empty
      // --------------------------------------------------

      dut.io.rxOutput.ready #=
        false

      dut.clockDomain.waitSampling()

      assert(
        !dut.io.rxOutput.valid.toBoolean,
        "RX FIFO should be empty after reading A/B/C"
      )

      println(
        "[PASS] RX FIFO is empty"
      )

      println(
        "[PASS] UART TX -> UART RX -> RX FIFO loopback"
      )

      println(
        "[DONE] Full UART Stream/FIFO simulation passed."
      )
    }
}
```

---

# 8. 여기서 한 가지 주의할 점

위 simulation의:

```scala
while (!dut.io.txInput.ready.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

은 우리가 이전에 문제가 되었던 무제한 `while`과는 성격이 조금 다릅니다.

여기서는 `ready`가:

```scala
state === State.IDLE
```

이고 TX가 정상적으로 동작한다면 반드시 IDLE에 도달합니다.

하지만 실제 프로젝트에서는 **timeout을 추가하는 것이 더 안전**합니다.

이번 학습에서는 코드 흐름을 간단하게 유지하기 위해 먼저 이 형태로 이해하겠습니다.

그리고 중요한 것은:

```scala
dut.io.txInput.valid #= true
```

를 먼저 유지하고,

```scala
while (!ready) {
    waitSampling()
}
```

으로 **ready가 올라올 때까지 valid/payload를 유지한다**는 것입니다.

이것이 Stream Producer의 올바른 기본 동작입니다.

---

# 9. 이 시뮬레이션에서 확인하는 것

이번 테스트는 단순히 "UART가 동작한다"만 검사하지 않습니다.

총 4개의 계층을 동시에 검사합니다.

### TX Stream

```text
CPU
 ↓
txInput.fire
```

A/B/C가 TX FIFO에 들어갑니다.

### TX FIFO

```text
TX FIFO
 ↓
output.fire
```

A/B/C를 UART TX에 순서대로 전달합니다.

### UART

```text
UartTx
 ↓
serial
 ↓
UartRx
```

UART frame을 실제 serial bit로 변환했다가 다시 byte로 복원합니다.

### RX FIFO

```text
UartRx.valid
 ↓
RX FIFO
 ↓
rxOutput.fire
 ↓
CPU
```

최종적으로:

```text
A → B → C
```

순서가 보존되는지 검사합니다.

---

# 10. 예상되는 핵심 로그

정상적으로 동작하면 핵심적으로 이런 형태가 나와야 합니다.

```text
[Initial] txReady=true rxValid=false

[CPU TX] accepted 0x41
[CPU TX] accepted 0x42
[CPU TX] accepted 0x43

[CPU TX] A -> B -> C inserted into TX FIFO

[UART] transmission / reception wait complete

[RX] data available

[CPU RX] received 0x41
[CPU RX] received 0x42
[CPU RX] received 0x43

[PASS] RX FIFO is empty

[PASS] UART TX -> UART RX -> RX FIFO loopback

[DONE] Full UART Stream/FIFO simulation passed.
```

---

# 11. 이번 단계에서 가장 중요한 학습 포인트

이번에는 단순히 UART를 만든 것이 아닙니다.

우리가 만든 데이터 경로는:

```text
CPU
 │
 │ Stream
 │
 ▼
TX FIFO
 │
 │ Stream
 │
 ▼
UART TX
 │
 │ serial
 ▼
UART RX
 │
 │ valid + data
 ▼
RX FIFO
 │
 │ Stream
 │
 ▼
CPU
```

입니다.

여기서 **Stream이 실제 IP 블록 사이의 계약(interface contract)** 역할을 합니다.

특히 TX 방향:

```text
CPU
 ↓
Stream
 ↓
TX FIFO
 ↓
Stream
 ↓
UART TX
```

에서는:

```text
valid
payload
ready
fire
```

가 모두 사용됩니다.

---

# 12. 다음 학습 단계

이제 UART + FIFO + Stream의 **기본 통합 구조가 완성**되었습니다.

다음부터는 새로운 기능을 계속 추가하기보다, 이 구조를 기반으로 **하드웨어 설계에서 중요한 두 가지**를 배우는 것이 좋습니다.

### 다음 ① — `Stream`과 `Flow`의 차이

```text
Stream
  valid
  ready
  payload

Flow
  valid
  payload
```

왜 `Flow`에는 `ready`가 없는지 배우겠습니다.

### 다음 ② — `StreamFifo`를 SpinalHDL 라이브러리 FIFO와 비교

우리가 직접 만든:

```text
StreamFifo
```

와 SpinalHDL이 제공하는 FIFO/Stream 관련 컴포넌트를 비교하면서,

```text
직접 RTL 작성
        vs
라이브러리 IP 사용
```

의 차이를 배우겠습니다.

그리고 그 다음에는 **AXI-Stream / `Stream` 기반 pipeline**으로 넘어가면 됩니다.