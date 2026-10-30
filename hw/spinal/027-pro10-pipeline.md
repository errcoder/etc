좋습니다. 이제 **Project 10 — Ready/Valid + Block Connection**으로 넘어가겠습니다.

이번 프로젝트는 앞으로 FIFO, ALU, CPU, Pipeline까지 계속 등장하는 개념이라 **너무 많은 예제를 만들지는 않되, handshake의 동작은 Simulation으로 확실히 확인**하겠습니다.

# Project 10 — Ready / Valid

## 1. 가장 먼저 개념부터

Ready/Valid 인터페이스는 두 신호를 사용합니다.

```text
valid = "보낼 데이터가 준비되어 있다"
ready = "받을 준비가 되어 있다"
```

그리고 실제 데이터 전송은:

```text
transfer = valid && ready
```

입니다.

즉:

```text
Producer                    Consumer
   │                           │
   │──── data ────────────────→│
   │──── valid ───────────────→│
   │←─── ready ────────────────│
   │                           │

          valid && ready
               ↓
            Transfer
```

---

# 2. 가장 작은 Ready/Valid 블록

먼저 Producer를 만들어 보겠습니다.

```scala
import spinal.core._

class SimpleProducer extends Component {
  val io = new Bundle {
    val data  = out Bits(8 bits)
    val valid = out Bool()
    val ready = in Bool()
  }

  val counter = Reg(UInt(8 bits)) init(0)

  io.data := counter.asBits
  io.valid := True

  when(io.valid && io.ready) {
    counter := counter + 1
  }
}
```

이 블록은 항상 데이터를 가지고 있습니다.

```text
valid = true
```

이고, Consumer가:

```text
ready = true
```

가 되는 순간 데이터를 하나 전송하고 counter가 증가합니다.

---

# 3. 핵심 코드

이 부분이 가장 중요합니다.

```scala
when(io.valid && io.ready) {
  counter := counter + 1
}
```

여기서:

```text
valid = 0, ready = 0 → 전송 없음
valid = 0, ready = 1 → 전송 없음
valid = 1, ready = 0 → 전송 없음
valid = 1, ready = 1 → 전송 발생
```

즉 **둘 다 1일 때만 transfer**입니다.

---

# 4. 왜 ready가 필요한가?

예를 들어 Producer가 데이터를 계속 만들어낸다고 해봅시다.

```text
Producer
  10
  11
  12
  13
  14
```

그런데 Consumer가 잠시 바빠서 데이터를 받을 수 없다면:

```text
Producer              Consumer

data=10 ─────────────→ ready=1
data=11 ─────────────→ ready=1

                       ↓ 바쁨

data=12 ─────────────→ ready=0
data=13 ─────────────→ ready=0
```

Producer는:

```text
valid = 1
ready = 0
```

상태에서 **데이터를 함부로 다음 값으로 넘기면 안 됩니다.**

이것이 Ready/Valid의 중요한 특징입니다.

---

# 5. 첫 번째 Simulation

이번에는 Consumer 역할을 Simulation에서 직접 만들어 보겠습니다.

```scala
import spinal.core._
import spinal.core.sim._

object SimpleProducerSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleProducer)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      dut.io.ready #= false

      for (cycle <- 0 until 10) {

        if (cycle >= 3) {
          dut.io.ready #= true
        }

        dut.clockDomain.waitSampling()

        val transfer =
          dut.io.valid.toBoolean && dut.io.ready.toBoolean

        println(
          f"cycle=$cycle%d " +
          f"valid=${dut.io.valid.toBoolean} " +
          f"ready=${dut.io.ready.toBoolean} " +
          f"data=0x${dut.io.data.toBigInt}%02X " +
          f"transfer=$transfer"
        )
      }
    }
}
```

---

# 6. 예상되는 동작

처음에는:

```text
ready = false
```

입니다.

따라서:

```text
valid = true
ready = false
transfer = false
```

이고 Producer의 counter는 증가하지 않습니다.

그러다가 cycle 3부터:

```text
ready = true
```

가 되면:

```text
valid = true
ready = true
transfer = true
```

가 됩니다.

따라서 데이터가:

```text
0x00
0x01
0x02
0x03
...
```

순서로 전송됩니다.

개념적으로:

```text
Cycle    valid   ready   transfer
----------------------------------
  0        1       0        0
  1        1       0        0
  2        1       0        0
  3        1       1        1
  4        1       1        1
  5        1       1        1
...
```

여기서 중요한 점은 **ready가 0인 동안 data가 마음대로 소비되지 않는다는 것**입니다.

---

# 7. 한 가지 중요한 SpinalHDL 관점

여기서 우리가 이전 FIFO에서 배웠던 것과 연결됩니다.

```scala
dut.io.ready #= true
```

는 Simulation에서 입력을 설정하는 것이고,

```scala
dut.clockDomain.waitSampling()
```

은 실제 clock edge를 기다리는 것입니다.

따라서 이번에도 FIFO 때처럼:

```text
Input 설정
   ↓
waitSampling()
   ↓
DUT의 sequential logic 동작
```

이라는 구조를 의식해야 합니다.

이것이 Ready/Valid Simulation에서 아주 중요합니다.

---

## 이번 단계의 목표

우선 이 **가장 작은 Producer** 하나만 실행해 보겠습니다.

성공하면 다음 단계에서:

```text
Producer
    │
    │ Ready/Valid
    ▼
Consumer
```

두 개의 실제 Component를 연결합니다.

그때 **Block Connection**과 `Stream`까지 연결해서 Project 10의 핵심을 완성하겠습니다.

먼저 위 `SimpleProducerSim`을 실행하고 **simulation 출력 결과를 보내주세요.**

---


좋습니다. 이제 **Project 10의 핵심인 `Stream`과 Block Connection**으로 진행하겠습니다.

앞에서 직접 만든

```text
valid
ready
data
```

를 SpinalHDL에서는 `Stream`으로 묶어서 다룰 수 있습니다.

---

# Project 10-2 — Stream

## 1. `Stream`이란?

우리가 앞에서 만든 Ready/Valid 인터페이스:

```text
data
valid
ready
```

를 하나의 객체로 묶은 것이:

```scala
Stream(Bits(8 bits))
```

입니다.

개념적으로:

```text
Stream(Bits(8 bits))

┌────────────────┐
│ payload : 8bit │
│ valid          │
│ ready          │
└────────────────┘
```

그리고 전송 조건은 여전히 동일합니다.

```text
valid && ready
```

---

# 2. Producer

이번에는 직접 `data`, `valid`, `ready`를 만들지 않고 `Stream`을 사용하겠습니다.

```scala id="q7w4c2"
import spinal.core._
import spinal.lib._

class StreamProducer extends Component {
  val io = new Bundle {
    val output = master(Stream(Bits(8 bits)))
  }

  val counter = Reg(UInt(8 bits)) init(0)

  io.output.valid := True
  io.output.payload := counter.asBits

  when(io.output.fire) {
    counter := counter + 1
  }
}
```

여기서 새로운 것이 하나 있습니다.

```scala id="7ujj3p"
io.output.fire
```

`Stream`에서:

```text
fire = valid && ready
```

입니다.

따라서:

```scala id="3qfdcm"
when(io.output.fire) {
  counter := counter + 1
}
```

은 앞에서 사용했던:

```scala id="0x9q5xj"
when(io.valid && io.ready) {
  counter := counter + 1
}
```

와 같은 의미입니다.

---

# 3. Consumer

Consumer는 반대로 `ready`를 생성합니다.

```scala id="7qj7o0"
import spinal.core._
import spinal.lib._

class StreamConsumer extends Component {
  val io = new Bundle {
    val input = slave(Stream(Bits(8 bits)))
  }

  io.input.ready := True

  when(io.input.fire) {
    println("This is hardware description, not simulation print")
  }
}
```

그런데 여기서 주의해야 합니다.

`println()`을 Hardware Component 내부에 넣어서 Simulation 로그를 출력하는 방식은 지금 단계에서는 사용하지 않겠습니다.

Consumer는 단순하게:

```scala id="y9x8u3"
io.input.ready := True
```

만 만들겠습니다.

그러면 Consumer는 항상 받을 준비가 되어 있습니다.

---

# 4. 실제 Block Connection

이제 두 블록을 하나로 연결합니다.

```scala id="9kq5z1"
import spinal.core._
import spinal.lib._

class StreamSystem extends Component {
  val producer = new StreamProducer
  val consumer = new StreamConsumer

  consumer.io.input <> producer.io.output
}
```

여기서:

```scala id="v3zv7w"
consumer.io.input <> producer.io.output
```

가 중요합니다.

`<>`는 **Stream의 연결 방향에 맞춰 내부 신호들을 연결**합니다.

개념적으로:

```text id="cmt9yv"
Producer                    Consumer

payload ─────────────────→ payload
valid   ─────────────────→ valid
ready   ←───────────────── ready
```

가 됩니다.

---

# 5. `master`와 `slave`

여기서 처음 보는 것이:

```scala id="j7t5jz"
master(Stream(...))
```

와

```scala id="cbt5s7"
slave(Stream(...))
```

입니다.

간단하게 생각하면 됩니다.

### Producer

```scala id="8v9qnj"
val output = master(Stream(...))
```

Producer가 데이터를 **내보내는 쪽**입니다.

### Consumer

```scala id="2nqv0b"
val input = slave(Stream(...))
```

Consumer가 데이터를 **받는 쪽**입니다.

방향은:

```text id="j6v4b4"
Producer
 master
   │
   │ Stream
   ▼
Consumer
 slave
```

입니다.

---

# 6. Simulation

이번에는 실제로 Producer와 Consumer가 연결되었는지 확인하겠습니다.

Consumer는 항상 `ready=true`이므로 모든 데이터가 전달됩니다.

```scala id="2p1i8m"
import spinal.core._
import spinal.core.sim._

object StreamSystemSim extends App {
  SimConfig
    .withWave
    .compile(new StreamSystem)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      for (cycle <- 0 until 10) {
        dut.clockDomain.waitSampling()

        val valid =
          dut.producer.io.output.valid.toBoolean

        val ready =
          dut.producer.io.output.ready.toBoolean

        val data =
          dut.producer.io.output.payload.toBigInt

        val fire =
          dut.producer.io.output.fire.toBoolean

        println(
          f"cycle=$cycle%d " +
          f"valid=$valid " +
          f"ready=$ready " +
          f"data=0x$data%02X " +
          f"fire=$fire"
        )
      }
    }
}
```

---

# 7. 예상 결과

이번에는 Consumer가 항상:

```scala id="4j1o8v"
io.input.ready := True
```

이므로:

```text id="7glq48"
cycle   valid   ready   data   fire
------------------------------------
0       true    true     00     true
1       true    true     01     true
2       true    true     02     true
3       true    true     03     true
4       true    true     04     true
...
```

와 같은 형태가 나와야 합니다.

앞에서 했던 테스트와 비교하면:

### 이전

```text id="rb7gkp"
Producer
   │
valid
ready
data
   │
Simulation에서 직접 ready 조작
```

### 지금

```text id="xg5s2j"
Producer
   │
   │ Stream
   ▼
Consumer
   │
ready
```

즉 **Ready/Valid가 실제 Component 간 연결로 발전했습니다.**

---

# 8. 이번 단계에서 기억할 것

이번에는 이것만 확실히 기억하면 충분합니다.

```text id="4p9qmg"
Stream = payload + valid + ready
```

그리고:

```text id="5frz2m"
stream.fire
    =
stream.valid && stream.ready
```

그리고 연결은:

```scala id="m5v3fa"
consumer.io.input <> producer.io.output
```

입니다.

---

## 다음 단계

이 Simulation을 실행해 주세요.

정상적으로 `00, 01, 02, 03...`이 전달되면 다음에는 **Consumer가 `ready=false`를 중간에 발생시키는 상황**을 만들어 보겠습니다.

그렇게 해야 `Stream`의 진짜 장점인 **back-pressure**를 확인할 수 있습니다.

```text
Producer ──→ Stream ──→ Consumer
              ↑
           ready=0
              │
        Producer 정지
```

이것까지 확인하면 Project 10의 Ready/Valid 핵심을 거의 완성하게 됩니다.

---

---


맞습니다. 이번 오류는 **DUT 내부의 `Stream` 신호를 Simulation에서 직접 읽으려고 했기 때문**입니다.

오류 메시지가 정확히 원인을 알려주고 있습니다.

```text
UNACCESSIBLE SIGNAL
toplevel/producer/io_output_valid : out Bool
isn't accessible during the simulation.
```

그리고 이 부분은 이전 FIFO에서 우리가 이미 경험했던 `simPublic()` 문제와 같은 종류입니다.

---

# 1. 왜 오류가 발생했는가?

Simulation 코드에서:

```scala
dut.producer.io.output.valid.toBoolean
```

처럼 **내부 Component의 output signal을 직접 읽었습니다.**

하지만 SpinalSim은 기본적으로 모든 내부 신호를 Simulation에서 읽을 수 있게 노출하지 않습니다.

그래서:

```scala
simPublic()
```

가 필요합니다.

다만 여기서는 **Producer 내부의 Stream 신호를 전부 `simPublic()`하는 것보다**, 테스트를 위한 신호를 `StreamSystem`의 top-level IO로 노출하는 것이 더 깔끔합니다.

학습용 코드에서도 이 방법이 더 좋습니다.

---

# 2. `StreamSystem`에 Simulation용 관찰 신호 추가

다음처럼 수정하겠습니다.

```scala
import spinal.core._
import spinal.lib._

class StreamSystem extends Component {
  val io = new Bundle {
    val valid = out Bool()
    val ready = out Bool()
    val data  = out Bits(8 bits)
    val fire  = out Bool()
  }

  val producer = new StreamProducer
  val consumer = new StreamConsumer

  consumer.io.input <> producer.io.output

  // Simulation observation
  io.valid := producer.io.output.valid
  io.ready := producer.io.output.ready
  io.data  := producer.io.output.payload
  io.fire  := producer.io.output.fire
}
```

이렇게 하면 Simulation에서:

```text id="f5o1qk"
StreamSystem
│
├── io.valid
├── io.ready
├── io.data
└── io.fire
```

가 top-level output으로 존재하게 됩니다.

따라서 별도의 `simPublic()`이 필요 없습니다.

---

# 3. Simulation도 수정

기존:

```scala
val valid =
  dut.producer.io.output.valid.toBoolean
```

대신:

```scala
val valid =
  dut.io.valid.toBoolean
```

를 사용합니다.

전체 코드는:

```scala
import spinal.core._
import spinal.core.sim._

object StreamSystemSim extends App {
  SimConfig
    .withWave
    .compile(new StreamSystem)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      for (cycle <- 0 until 10) {
        dut.clockDomain.waitSampling()

        val valid = dut.io.valid.toBoolean
        val ready = dut.io.ready.toBoolean
        val data  = dut.io.data.toBigInt
        val fire  = dut.io.fire.toBoolean

        println(
          f"cycle=$cycle%d " +
          f"valid=$valid " +
          f"ready=$ready " +
          f"data=0x$data%02X " +
          f"fire=$fire"
        )
      }
    }
}
```

---

# 4. 예상 결과

Consumer가 항상:

```scala
io.input.ready := True
```

이므로:

```text
valid = true
ready = true
fire  = true
```

가 계속 유지되어야 합니다.

그리고:

```text
data
00
01
02
03
04
05
...
```

순서로 증가합니다.

예상:

```text
cycle=0 valid=true ready=true data=0x00 fire=true
cycle=1 valid=true ready=true data=0x01 fire=true
cycle=2 valid=true ready=true data=0x02 fire=true
cycle=3 valid=true ready=true data=0x03 fire=true
...
```

---

## 5. 이번 오류에서 중요한 교훈

이번 오류는 단순히 코드를 고치는 것보다 **Simulation에서 신호를 어떻게 관찰하는가**를 이해하는 좋은 기회입니다.

### 방법 1 — top-level IO로 노출

```scala
val io = new Bundle {
  val data = out Bits(8 bits)
}
```

그리고:

```scala
io.data := internalSignal
```

**→ 이번처럼 외부에서 관찰할 신호에 가장 적합**

### 방법 2 — `simPublic()`

```scala
internalSignal.simPublic()
```

**→ 내부 구현 신호를 디버깅할 때 적합**

우리가 FIFO에서:

```scala
writePtr.simPublic()
readPtr.simPublic()
count.simPublic()
```

했던 것이 바로 이 방법입니다.

---

이번에는 **top-level IO 방식으로 수정해서 실행**해 주세요.

정상적으로 통과하면 다음에는 바로 **back-pressure (`ready=false`)**를 넣겠습니다.  
그 단계에서야 `Stream`이 왜 단순한 `data + valid`가 아니라 **Ready/Valid protocol**인지 확실하게 체감할 수 있습니다.
