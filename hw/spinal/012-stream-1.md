맞습니다. 여기서는 **더 이상 `ValidReadyFifoSim`의 timing 문제를 파고들지 않겠습니다.**

이번 과정에서 오히려 중요한 학습은 충분히 얻었습니다.

- `while` 자체는 좋은 도구다.
- `while`의 조건 평가도 DUT 관찰이다.
- `waitSampling()`은 단순히 "안전한 대기"가 아니다.
- 한 sampling edge에서 어떤 `push/pop`이 실제 발생하는지를 먼저 분석해야 한다.
- `waitFallingEdge()`를 습관적으로 추가하는 것도 해결책이 아니다.
- 특히 SpinalSim의 event ordering은 **실제 실행 결과와 waveform으로 검증하면서 다뤄야 한다.**
- 이번 FIFO simulation은 계속 수정하는 것보다 **학습 흐름을 앞으로 진행하는 것이 효율적**이라는 판단에 동의합니다.

그리고 제가 이번 과정에서 여러 차례 simulation timing을 충분히 검증하지 않은 상태에서 코드를 제시한 것은 분명히 제 실수입니다.

---

# 다음 학습 단계로 진행합시다

지금까지 학습한 흐름은:

```text
Project 5
Timer
  ↓
Project 6
UART TX
  ↓
Project 7
UART RX
  ↓
UART Loopback
  ↓
Simple FIFO
  ↓
UART RX → FIFO
  ↓
Valid / Ready
```

여기까지 왔습니다.

다음 단계에서는 **ValidReadyFifo 자체를 더 디버깅하기보다는**, 지금까지 배운 개념을 실제 디지털 시스템 구조로 확장하는 것이 좋습니다.

제가 추천하는 다음 순서는:

```text
        현재
         │
         ▼
┌──────────────────────┐
│  Valid / Ready 개념  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Stream / Handshake  │
│  구조 이해           │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  UART TX/RX + FIFO   │
│  시스템 구조         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Register Interface  │
│  (간단한 CPU 접근)   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  APB / 간단한 버스   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Timer + UART + FIFO │
│  작은 SoC 구조       │
└──────────────────────┘
```

## 다음 주제로는 **Stream**을 추천합니다.

왜냐하면 지금 배운:

```text
valid
ready
fire = valid && ready
```

가 SpinalHDL에서 매우 중요한 기본 패턴이고, 이를 `Stream`으로 연결하면 훨씬 자연스럽게 사용할 수 있기 때문입니다.

예를 들어 개념적으로:

```scala
val input  = slave(Stream(Bits(8 bits)))
val output = master(Stream(Bits(8 bits)))
```

같은 구조를 배우게 됩니다.

그리고 이것을 통해:

```text
Producer
   │
   │ valid + payload
   ▼
 Stream
   │
   │ ready
   ▼
Consumer
```

라는 **하드웨어 데이터 전송 모델**을 제대로 이해할 수 있습니다.

그 다음에는 `Stream`을 이용해서 FIFO를 바라보면:

```text
UART RX
   │
   │ Stream[8-bit]
   ▼
 FIFO
   │
   │ Stream[8-bit]
   ▼
CPU / UART TX / 다른 로직
```

처럼 지금까지 만든 블록들이 훨씬 깔끔하게 연결됩니다.

---

# 다음 학습: `Stream`

다음부터는 simulation timing 문제를 억지로 더 파지 않고,

**① Stream의 구조 → ② `valid/ready/fire` → ③ `Stream` 연결 → ④ 간단한 Stream Producer/Consumer → ⑤ FIFO와 Stream의 관계**

순서로 진행하는 것이 좋겠습니다.

그리고 SpinalHDL 코드는 계속 **Release 34 / v1.15.0 기준**으로 작성하고, simulation 예제는 이번 경험을 반영해서 **실제 edge와 관찰 시점을 먼저 분석한 후** 제시하겠습니다.

---

```scala
import spinal.core._

case class StreamPassThrough(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input  = slave Stream(Bits(dataWidth bits))
    val output = master Stream(Bits(dataWidth bits))
  }

  io.output.valid   := io.input.valid
  io.output.payload := io.input.payload

  io.input.ready := io.output.ready
}
```

---

---

# 다음 단계 — `Stream.fire`

좋습니다. 이제 **`Stream.fire`**를 배우겠습니다.

이 개념은 앞으로 `Stream`, FIFO, UART, AXI-Stream 같은 인터페이스를 이해할 때 계속 등장합니다.

---

# 1. `fire`란?

Stream에는:

```scala
stream.valid
stream.ready
```

가 있습니다.

데이터가 실제로 전달되는 조건은:

```scala
stream.valid && stream.ready
```

입니다.

SpinalHDL에서는 이것을 간단하게:

```scala
stream.fire
```

라고 표현할 수 있습니다.

즉,

```scala
stream.fire
```

는 개념적으로:

```scala
stream.valid && stream.ready
```

입니다.

---

# 2. 예를 들어보겠습니다

다음 Stream이 있다고 합시다.

```scala
val input = slave Stream(Bits(8 bits))
```

그러면:

```scala
input.valid
input.payload
input.ready
input.fire
```

를 사용할 수 있습니다.

다음 코드는:

```scala
when(input.fire) {
    // 데이터가 실제로 전달됨
}
```

이라는 뜻입니다.

---

# 3. 네 가지 경우

| valid | ready | fire | 의미 |
|---:|---:|---:|---|
| 0 | 0 | 0 | 데이터 없음 |
| 0 | 1 | 0 | 받을 수 있지만 데이터 없음 |
| 1 | 0 | 0 | 데이터는 있지만 아직 못 받음 |
| 1 | 1 | **1** | **데이터 전달** |

가장 중요한 것은 이것입니다.

> `valid = 1`이라고 해서 데이터가 전달된 것이 아닙니다.

반드시:

```text
valid = 1
ready = 1
```

이어야 합니다.

---

# 4. Producer 입장에서 보면

Producer가:

```text
valid = 1
payload = 0x41
```

을 출력합니다.

그런데 Consumer가:

```text
ready = 0
```

이면:

```text
fire = 0
```

입니다.

따라서 Producer는 `0x41`을 유지해야 합니다.

```text
Cycle       valid    ready    payload    fire

  1           1        0       0x41       0
  2           1        0       0x41       0
  3           1        1       0x41       1
```

3번째 클럭에서 실제 전송됩니다.

---

# 5. `fire`는 "이벤트"라고 생각하면 좋습니다

다음처럼 생각하면 이해하기 쉽습니다.

```text
valid = "보낼 물건이 있다"
ready = "받을 수 있다"

fire = "그래서 지금 거래가 성립했다"
```

즉:

```text
valid
  +
ready
  ↓
 fire
  ↓
transfer
```

---

# 6. `fire`를 실제 RTL에서 사용하는 방법

예를 들어 Counter로 데이터를 보내는 Producer를 생각해 봅시다.

```scala
import spinal.core._

case class StreamCounter(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val output = master Stream(UInt(dataWidth bits))
  }

  val counter = Reg(UInt(dataWidth bits)) init(0)

  io.output.valid   := True
  io.output.payload := counter

  when(io.output.fire) {
    counter := counter + 1
  }
}
```

여기서 아주 중요한 부분은:

```scala
when(io.output.fire) {
    counter := counter + 1
}
```

입니다.

---

# 7. 이 회로가 어떻게 동작하는가?

초기 상태:

```text
counter = 0
```

따라서:

```text
output.valid   = 1
output.payload = 0
```

입니다.

Consumer가:

```text
output.ready = 0
```

이면:

```text
fire = 0
```

따라서 counter는 증가하지 않습니다.

```text
0 → 0 → 0 → 0
```

Consumer가 `ready = 1`이 되는 순간:

```text
valid = 1
ready = 1
fire  = 1
```

이므로:

```text
counter := counter + 1
```

이 실행됩니다.

다음 데이터는:

```text
1
```

이 됩니다.

---

# 8. 이것이 FIFO와도 정확히 연결됩니다

우리가 앞에서 배운 FIFO를 생각해 봅시다.

FIFO 입력:

```text
input.valid
input.payload
input.ready
```

실제 FIFO write는:

```scala
when(input.fire) {
    // FIFO에 데이터 저장
}
```

가 됩니다.

즉:

```scala
input.fire
```

는:

```scala
input.valid && input.ready
```

이고,

```scala
input.ready
```

는 FIFO가 가득 차지 않았을 때:

```scala
input.ready := !full
```

가 됩니다.

결과적으로:

```text
Producer
   │
   │ valid + payload
   ▼
┌───────────┐
│    FIFO   │
└───────────┘
   ▲
   │ ready = !full
```

FIFO가 가득 차면:

```text
full  = 1
ready = 0
fire  = 0
```

따라서 데이터가 들어가지 않습니다.

FIFO에 공간이 생기면:

```text
full  = 0
ready = 1
```

그리고 Producer가 `valid=1`을 유지하고 있다면:

```text
fire = 1
```

이 되어 데이터가 들어갑니다.

---

# 9. `fire`에서 매우 중요한 규칙

특히 Producer를 만들 때 이것을 기억해야 합니다.

### `valid`는 `ready`를 기다렸다가 올리는 것이 아닙니다.

잘못된 사고:

```text
ready == 1
    ↓
valid = 1
```

일반적인 Stream 규칙에서는 Producer가 먼저:

```text
valid = 1
payload = valid data
```

를 제시하고,

Consumer가:

```text
ready = 1
```

을 제시하면:

```text
fire = 1
```

이 됩니다.

즉:

```text
Producer                  Consumer

valid=1 ────────────────►
data=41  ───────────────►
         ◄────────────── ready=1

             ↓
          fire = 1
```

---

# 10. 오늘 여기서 잡아야 할 핵심

지금 단계에서는 이것만 확실하게 기억하면 됩니다.

```scala
stream.fire
```

=

```scala
stream.valid && stream.ready
```

그리고 의미는:

> **"현재 클럭에서 Stream 데이터 전송이 성립했다."**

입니다.

---

# 다음 단계

다음에는 이제 이 개념을 실제로 사용해서:

```text
StreamProducer
       │
       ▼
StreamPassThrough
       │
       ▼
StreamConsumer
```

구조를 만들고,

```text
valid
ready
fire
payload
```

가 **클럭마다 어떻게 움직이는지** 직접 추적해 보겠습니다.

그 다음 바로 **`StreamFifo`**로 넘어가면 지금까지 배운 `FIFO + valid/ready + fire`가 하나로 연결됩니다.

---

---

# 다음 단계 — `StreamProducer → PassThrough → Consumer`

좋습니다. 이제 `fire`를 실제 구조에 넣어보겠습니다.

이번 단계의 목적은 **코드를 복잡하게 만드는 것보다 `valid / ready / fire`의 관계를 완전히 이해하는 것**입니다.

---

## 1. 전체 구조

```text
Producer
   │
   │ Stream
   ▼
PassThrough
   │
   │ Stream
   ▼
Consumer
```

Producer가 데이터를 만들고,

PassThrough가 그대로 전달하고,

Consumer가 받을 수 있을 때 `ready`를 올립니다.

---

# 2. Producer

먼저 아주 간단한 Producer를 만들어 보겠습니다.

```scala
import spinal.core._

case class StreamProducer(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val output = master Stream(Bits(dataWidth bits))
  }

  val counter = Reg(UInt(dataWidth bits)) init(0)

  io.output.valid   := True
  io.output.payload := counter

  when(io.output.fire) {
    counter := counter + 1
  }
}
```

여기서 핵심은 딱 이것입니다.

```scala
when(io.output.fire) {
    counter := counter + 1
}
```

---

# 3. 왜 `fire`일 때만 증가할까?

Consumer가 데이터를 받을 준비가 안 되어 있을 수 있기 때문입니다.

예를 들어:

```text
cycle   valid   ready   payload   fire
---------------------------------------
  1       1       0       0        0
  2       1       0       0        0
  3       1       1       0        1
  4       1       1       1        1
  5       1       1       2        1
```

Producer는 처음부터:

```text
valid = 1
payload = 0
```

을 유지합니다.

하지만 Consumer가 `ready=0`인 동안에는:

```text
fire = 0
```

입니다.

따라서 counter도 계속:

```text
0
```

입니다.

Consumer가 `ready=1`이 되는 순간:

```text
fire = 1
```

이 되고 그때 counter가 증가합니다.

---

# 4. PassThrough

앞에서 만들었던 회로입니다.

```scala
import spinal.core._

case class StreamPassThrough(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input  = slave Stream(Bits(dataWidth bits))
    val output = master Stream(Bits(dataWidth bits))
  }

  io.output.valid   := io.input.valid
  io.output.payload := io.input.payload

  io.input.ready := io.output.ready
}
```

여기서는 데이터가 저장되지 않습니다.

그냥:

```text
input.valid   → output.valid
input.payload → output.payload
output.ready  → input.ready
```

로 연결합니다.

---

# 5. Consumer

이번에는 Consumer가 `ready`를 결정하도록 하겠습니다.

```scala
import spinal.core._

case class StreamConsumer(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input = slave Stream(Bits(dataWidth bits))
  }

  io.input.ready := True

  when(io.input.fire) {
    // 데이터가 실제로 들어온 순간
    // 여기에서 데이터 처리
  }
}
```

여기서는 항상:

```scala
io.input.ready := True
```

이므로 Consumer는 항상 받을 수 있습니다.

따라서 Producer가 `valid=1`이면:

```text
valid = 1
ready = 1
fire  = 1
```

이 됩니다.

---

# 6. 세 개를 연결하면

Top-level에서:

```scala
import spinal.core._

case class StreamDemo(
    dataWidth: Int
) extends Component {

  val producer = StreamProducer(dataWidth)
  val pass     = StreamPassThrough(dataWidth)
  val consumer = StreamConsumer(dataWidth)

  pass.io.input  << producer.io.output
  consumer.io.input << pass.io.output
}
```

여기서:

```scala
pass.io.input << producer.io.output
```

는 Stream 전체를 연결합니다.

그리고:

```scala
consumer.io.input << pass.io.output
```

도 마찬가지입니다.

---

# 7. 실제 데이터 흐름

회로는 이렇게 됩니다.

```text
┌──────────┐
│ Producer │
└────┬─────┘
     │
     │ valid
     │ payload
     ▼
┌─────────────┐
│ PassThrough │
└──────┬──────┘
       │
       │ valid
       │ payload
       ▼
┌──────────┐
│ Consumer │
└──────────┘
```

그리고 `ready`는 반대 방향입니다.

```text
┌──────────┐
│ Producer │
└────▲─────┘
     │ ready
     │
┌────┴────────┐
│ PassThrough │
└────▲────────┘
     │ ready
     │
┌────┴─────┐
│ Consumer │
└──────────┘
```

---

# 8. 여기서 아주 중요한 부분

Consumer가:

```scala
io.input.ready := True
```

이므로 항상 받을 수 있습니다.

그러면 처음부터:

```text
Producer:

valid   = 1
payload = 0

Consumer:

ready   = 1
```

따라서:

```text
fire = 1
```

입니다.

다음 클럭에는 Producer의 counter가 증가합니다.

```text
cycle 0 → payload 0
cycle 1 → payload 1
cycle 2 → payload 2
cycle 3 → payload 3
...
```

즉 **매 클럭마다 데이터가 하나씩 전달됩니다.**

---

# 9. 그런데 Consumer가 멈춘다면?

여기가 정말 중요합니다.

Consumer를 다음처럼 바꿀 수 있습니다.

```scala
io.input.ready := False
```

그러면:

```text
Producer
valid = 1
payload = 0

Consumer
ready = 0
```

이므로:

```text
fire = 0
```

입니다.

Producer의 counter는 증가하지 않습니다.

```text
cycle 1 → payload 0
cycle 2 → payload 0
cycle 3 → payload 0
cycle 4 → payload 0
```

Consumer가 다시:

```text
ready = 1
```

이 되는 순간:

```text
valid  = 1
ready  = 1
fire   = 1
payload = 0
```

그리고 그때 데이터가 전달됩니다.

---

# 10. 이것이 `Stream`의 가장 중요한 장점

Producer는 Consumer의 상태를 직접 알 필요가 없습니다.

Producer는 단순히:

```text
"데이터가 있습니다."
```

라고:

```text
valid = 1
```

을 유지합니다.

Consumer는:

```text
"지금 받을 수 있습니다."
```

라고:

```text
ready = 1
```

을 올립니다.

둘이 만나는 순간:

```text
valid && ready
```

즉:

```text
fire
```

가 발생합니다.

---

# 11. 현재까지 배운 것을 하나로 정리

```text
                 Stream
                   │
        ┌──────────┴──────────┐
        │                     │
      valid                 payload
        │                     │
        │                     │
        ▼                     ▼
     Producer ──────────────► Consumer
        ▲                     │
        │                     │
        └────── ready ────────┘
```

그리고:

```text
             valid
               +
             ready
               ↓
             fire
               ↓
          data transfer
```

---

# 12. 다음 — `StreamFifo`

이제 드디어 우리가 앞에서 공부했던 FIFO와 합칠 차례입니다.

지금까지 배운:

```text
SimpleFifo
    +
valid / ready
    +
fire
```

를 하나의 회로로 만들면:

```text
Producer
   │
   │ Stream
   ▼
┌────────────┐
│ StreamFifo │
└────────────┘
   │
   │ Stream
   ▼
Consumer
```

가 됩니다.

그리고 여기서 아주 중요한 설계 포인트가 하나 생깁니다.

> **FIFO가 FULL일 때 `ready`를 어떻게 만들 것인가?**

바로:

```scala
input.ready := !full
```

입니다.

그리고 FIFO 출력에서는:

```scala
output.valid := !empty
```

가 됩니다.

따라서 우리가 이미 배운 `SimpleFifo`의:

```text
full
empty
write
read
```

가

```text
ready
valid
fire
```

로 자연스럽게 변환됩니다.

**다음 단계에서는 이 `StreamFifo`를 SpinalHDL 1.15.0 기준으로 직접 구현하면서 `SimpleFifo`와 비교해 보겠습니다.**

---

---

# 다음 단계 — `StreamFifo`

좋습니다. 이제 **`SimpleFifo`를 `Stream` 인터페이스로 바꾸는 단계**입니다.

이 단계는 지금까지 배운 내용을 연결하는 중요한 단계입니다.

```text
SimpleFifo
    ↓
valid / ready
    ↓
fire
    ↓
StreamFifo
```

---

# 1. 먼저 기존 `SimpleFifo`와 비교

기존 FIFO는:

```text
입력
write
writeData

출력
read
readData

상태
full
empty
```

였습니다.

Stream 인터페이스로 바꾸면:

```text
입력
input.valid
input.payload
input.ready

출력
output.valid
output.payload
output.ready
```

가 됩니다.

둘을 대응시키면:

| SimpleFifo | StreamFifo |
|---|---|
| `write` | `input.fire` |
| `writeData` | `input.payload` |
| `!full` | `input.ready` |
| `read` | `output.fire` |
| `readData` | `output.payload` |
| `!empty` | `output.valid` |

즉 핵심은:

```text
input.fire  → FIFO write
output.fire → FIFO read
```

입니다.

---

# 2. StreamFifo 구현

SpinalHDL 1.15.0 기준으로 직접 구현해 보겠습니다.

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

  // -------------------------
  // Memory
  // -------------------------

  val mem = Vec(
    Reg(Bits(dataWidth bits)) init(0),
    depth
  )

  // -------------------------
  // Pointers
  // -------------------------

  val ptrWidth =
    log2Up(depth)

  val writePtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val readPtr =
    Reg(UInt(ptrWidth bits)) init(0)

  // -------------------------
  // Count
  // -------------------------

  val countWidth =
    log2Up(depth + 1)

  val count =
    Reg(UInt(countWidth bits)) init(0)

  // -------------------------
  // FIFO status
  // -------------------------

  val empty =
    count === 0

  val full =
    count === depth

  // -------------------------
  // Stream interface
  // -------------------------

  io.input.ready :=
    !full

  io.output.valid :=
    !empty

  io.output.payload :=
    mem(readPtr)

  // -------------------------
  // Transfers
  // -------------------------

  val push =
    io.input.fire

  val pop =
    io.output.fire

  // -------------------------
  // Write
  // -------------------------

  when(push) {

    mem(writePtr) :=
      io.input.payload

    when(writePtr === depth - 1) {
      writePtr := 0
    } otherwise {
      writePtr := writePtr + 1
    }
  }

  // -------------------------
  // Read
  // -------------------------

  when(pop) {

    when(readPtr === depth - 1) {
      readPtr := 0
    } otherwise {
      readPtr := readPtr + 1
    }
  }

  // -------------------------
  // Count
  // -------------------------

  when(push && !pop) {
    count := count + 1
  } elsewhen(!push && pop) {
    count := count - 1
  }
}
```

---

# 3. 가장 중요한 부분

코드에서 사실 핵심은 이것입니다.

```scala
io.input.ready := !full
```

그리고:

```scala
io.output.valid := !empty
```

그리고:

```scala
val push = io.input.fire
val pop  = io.output.fire
```

이 네 줄의 의미를 이해하면 됩니다.

---

# 4. 입력 쪽

FIFO가 비어 있든 차 있든, **FULL이 아니면 데이터를 받을 수 있습니다.**

```scala
io.input.ready := !full
```

예를 들어:

```text
count = 0
full  = false
ready = true
```

Producer가:

```text
valid = true
```

라면:

```text
valid = 1
ready = 1
fire  = 1
```

따라서:

```text
FIFO에 데이터 저장
```

입니다.

---

# 5. FIFO가 FULL이면?

예를 들어 depth가 4이고:

```text
count = 4
```

이면:

```text
full = true
```

따라서:

```scala
io.input.ready := !full
```

결과:

```text
ready = false
```

입니다.

Producer가 데이터를 가지고 있어도:

```text
valid = 1
ready = 0
fire  = 0
```

입니다.

따라서 FIFO는 데이터를 받지 않습니다.

---

# 6. 출력 쪽

반대로 FIFO가 비어 있지 않으면:

```scala
io.output.valid := !empty
```

입니다.

예를 들어:

```text
count = 3
empty = false
```

이면:

```text
output.valid = 1
```

입니다.

Consumer가:

```text
output.ready = 1
```

이면:

```text
output.valid = 1
output.ready = 1
```

따라서:

```text
output.fire = 1
```

입니다.

그리고:

```scala
when(pop) {
    ...
}
```

에서 `readPtr`가 이동합니다.

---

# 7. 중요한 점 — `payload`는 현재 FIFO의 Head

이 부분:

```scala
io.output.payload := mem(readPtr)
```

때문에 `output.payload`는 항상 현재 FIFO의 첫 번째 데이터입니다.

예를 들어:

```text
FIFO:

A
B
C
D
```

이면:

```text
readPtr → A

output.payload = A
```

Consumer가:

```text
ready = 0
```

이라면:

```text
valid = 1
ready = 0
fire  = 0
```

이고 `readPtr`는 움직이지 않습니다.

따라서:

```text
output.payload = A
```

가 그대로 유지됩니다.

Consumer가 `ready=1`이 되는 순간:

```text
valid = 1
ready = 1
fire  = 1
```

A가 전달되고 `readPtr`가 B를 가리키게 됩니다.

---

# 8. 이것이 FIFO + Stream의 핵심

전체적으로 보면:

```text
             input Stream
                 │
                 ▼
        ┌─────────────────┐
        │                 │
        │   StreamFifo    │
        │                 │
        │   memory        │
        │   writePtr      │
        │   readPtr       │
        │   count         │
        │                 │
        └─────────────────┘
                 │
                 ▼
            output Stream
```

입력에서는:

```text
input.fire
```

가 발생하면 저장합니다.

출력에서는:

```text
output.fire
```

가 발생하면 꺼냅니다.

---

# 9. `SimpleFifo`와 비교하면 훨씬 명확합니다

기존:

```scala
val doWrite =
    io.write && !io.full

val doRead =
    io.read && !io.empty
```

StreamFifo:

```scala
val push =
    io.input.fire

val pop =
    io.output.fire
```

그런데 `fire`의 정의를 생각하면:

```text
input.fire
=
input.valid && input.ready
```

이고:

```text
input.ready
=
!full
```

이므로 사실상:

```text
input.fire
=
input.valid && !full
```

입니다.

즉 기존 `doWrite`와 **완전히 같은 역할**입니다.

출력도 마찬가지입니다.

```text
output.fire
=
output.valid && output.ready
```

그리고:

```text
output.valid
=
!empty
```

이므로:

```text
output.fire
=
!empty && output.ready
```

입니다.

기존 `doRead`와 동일한 개념입니다.

---

# 10. 아주 중요한 차이

기존 FIFO에서는 사용자가:

```text
write
read
```

를 직접 제어했습니다.

하지만 StreamFifo에서는:

```text
valid
ready
```

를 통해 **양쪽이 협력해서 전송 여부를 결정**합니다.

그래서 Stream은 단순 FIFO보다 더 일반적인 인터페이스입니다.

```text
Producer
   │
   │ valid + payload
   ▼
Stream
   ▲
   │ ready
Consumer
```

이 구조를 FIFO뿐 아니라:

- UART
- DMA
- CPU bus
- packet processing
- pipeline
- FPGA data path

등에서도 사용할 수 있습니다.

---

# 11. 이번 단계에서 꼭 기억할 것

딱 세 줄입니다.

### 입력

```scala
io.input.ready := !full
```

### 출력

```scala
io.output.valid := !empty
```

### 실제 전송

```scala
io.input.fire
io.output.fire
```

그리고:

```text
input.fire  → FIFO에 넣는다
output.fire → FIFO에서 꺼낸다
```

입니다.

---

# 다음 단계

이제 **StreamFifo 자체의 구조는 완성**했습니다.

다음은 바로 중요한 실전 개념인:

## `StreamFifo`의 `push + pop` 동시 발생

을 보겠습니다.

예를 들어 FIFO가:

```text
A B C
```

를 가지고 있고 동시에:

```text
input = D
output.ready = 1
```

이라면 한 클럭에서:

```text
push = 1
pop  = 1
```

이 발생할 수 있습니다.

그러면 `count`는:

```text
3 → 3
```

으로 **변하지 않아야 합니다.**

이것을 이해하면 지금까지 공부한 `count`, `full`, `empty`, `fire`가 하나로 연결됩니다.

---

---

# 다음 단계 — `push + pop` 동시 발생

좋습니다. 이제 **StreamFifo에서 가장 중요한 동작 중 하나**를 보겠습니다.

FIFO가 데이터를 넣는 동시에 데이터를 꺼낼 수 있습니다.

```text
             ┌─────────────┐
input ──────►│ StreamFifo  │──────► output
             └─────────────┘
                  ↑     ↑
                 push   pop
```

핵심은:

```text
push = input.fire
pop  = output.fire
```

입니다.

---

# 1. FIFO에 A, B, C가 있다고 합시다

초기:

```text
FIFO

┌───┬───┬───┐
│ A │ B │ C │
└───┴───┴───┘
  ↑
readPtr
```

현재:

```text
count = 3
```

입니다.

Consumer가 A를 받을 준비가 되어 있고:

```text
output.ready = 1
```

Producer도 D를 보내려고 합니다.

```text
input.valid   = 1
input.payload = D
```

FIFO가 full이 아니라면:

```text
input.ready = 1
output.valid = 1
```

따라서:

```text
input.fire  = 1
output.fire = 1
```

입니다.

---

# 2. 한 클럭에서 동시에 발생

결국 한 클럭에서:

```text
push = 1
pop  = 1
```

입니다.

그러면:

```text
FIFO
A B C
```

에서

```text
A 제거
D 추가
```

가 동시에 일어납니다.

결과:

```text
FIFO

B C D
```

입니다.

---

# 3. 그런데 `count`는?

이것이 핵심입니다.

기존 `SimpleFifo`에서 이미 이런 코드를 사용했습니다.

```scala
when(doWrite && !doRead) {
  count := count + 1
} elsewhen(!doWrite && doRead) {
  count := count - 1
}
```

StreamFifo에서는:

```scala
when(push && !pop) {
  count := count + 1
} elsewhen(!push && pop) {
  count := count - 1
}
```

입니다.

`push=1`, `pop=1`이면:

```text
push && !pop = false

!push && pop = false
```

따라서 어느 쪽도 실행되지 않습니다.

즉:

```text
count = 3
```

그대로 유지됩니다.

---

# 4. 이것이 정확히 맞는 이유

한 개 들어오고:

```text
+1
```

한 개 나갑니다:

```text
-1
```

따라서:

```text
+1 - 1 = 0
```

입니다.

즉:

```text
count
  ↓
변화 없음
```

입니다.

---

# 5. 네 가지 경우를 정리

StreamFifo에서는 `push`와 `pop`을 다음 네 가지 경우로 볼 수 있습니다.

| push | pop | count 변화 |
|---:|---:|---:|
| 0 | 0 | 0 |
| 1 | 0 | +1 |
| 0 | 1 | -1 |
| 1 | 1 | 0 |

이 표는 앞으로 FIFO를 설계할 때 매우 중요합니다.

---

# 6. FULL 상태에서는 조금 더 생각해야 합니다

여기서 재미있는 문제가 생깁니다.

FIFO가 FULL이라고 합시다.

depth = 4:

```text
A B C D
```

그러면:

```text
count = 4
full = true
```

현재 코드에서는:

```scala
io.input.ready := !full
```

이므로:

```text
input.ready = 0
```

입니다.

따라서 Producer가 `valid=1`이어도:

```text
input.fire = 0
```

입니다.

반면 Consumer가 받을 준비가 되어 있으면:

```text
output.fire = 1
```

입니다.

결과:

```text
push = 0
pop  = 1
```

이므로:

```text
count: 4 → 3
```

입니다.

---

# 7. FULL에서 READ + WRITE를 동시에 하면?

현재 우리가 만든 StreamFifo에서는:

```text
full = true
```

이므로:

```text
input.ready = false
```

입니다.

따라서:

```text
push = 0
```

입니다.

Consumer가 읽으면:

```text
pop = 1
```

그래서:

```text
count: 4 → 3
```

입니다.

즉 **현재 설계에서는 FULL 상태에서 같은 클럭의 push+pop을 허용하지 않습니다.**

이것은 오류가 아닙니다.

**설계 정책**입니다.

---

# 8. 왜 이 정책을 사용하는가?

현재 구조는 매우 단순합니다.

```scala
io.input.ready := !full
```

즉 FIFO가 FULL이면 무조건:

```text
ready = 0
```

입니다.

따라서 FIFO가 FULL인 순간에는 새로운 입력을 받을 수 없습니다.

Consumer가 데이터를 하나 꺼낸 뒤 다음 사이클부터:

```text
count = 3
full = false
input.ready = true
```

가 됩니다.

그때 다시 입력을 받을 수 있습니다.

---

# 9. 다른 FIFO는 다르게 만들 수도 있습니다

좀 더 고급 FIFO에서는 FULL 상태에서도:

```text
pop = 1
```

이라면 동시에:

```text
push = 1
```

을 허용할 수 있습니다.

예:

```text
FULL

A B C D
```

동시에:

```text
pop A
push E
```

결과:

```text
B C D E
```

count는:

```text
4 → 4
```

그대로입니다.

이런 설계에서는 `ready`를 단순히:

```scala
!full
```

로 만들 수 없습니다.

예를 들어 개념적으로:

```text
ready = !full || pop
```

처럼 만들 수 있습니다.

하지만 여기에는 **조합 논리 경로와 같은 사이클의 `pop` 의존성** 등을 신중하게 고려해야 합니다.

지금 단계에서는 이것을 구현하지 않고 넘어가는 것이 좋습니다.

---

# 10. 지금까지 배운 FIFO와 연결

우리가 처음 만든 `SimpleFifo`에서:

```text
write
read
```

를 사용했습니다.

이제 StreamFifo에서는:

```text
input.fire
output.fire
```

를 사용합니다.

그리고 `fire`는:

```text
valid && ready
```

입니다.

따라서 전체 구조는:

```text
                 input
        valid ───────────►
        payload ─────────►
        ready ◄──────────

              StreamFifo

        valid ───────────►
        payload ─────────►
        ready ◄──────────
                 output
```

그리고 내부적으로:

```text
input.fire
    ↓
 push
    ↓
writePtr / memory / count


output.fire
    ↓
 pop
    ↓
readPtr / count
```

입니다.

---

# 11. 이제 `StreamFifo`의 핵심이 완성됐습니다

지금까지:

```text
① Stream
       ↓
② valid / ready
       ↓
③ fire
       ↓
④ Stream Producer / Consumer
       ↓
⑤ StreamFifo
       ↓
⑥ push / pop
       ↓
⑦ simultaneous push + pop
```

까지 왔습니다.

이제부터는 **실제 설계에 Stream을 적용**하는 단계로 넘어가면 됩니다.

---

# 다음 단계 — UART와 Stream 연결

지금까지 만든 UART 구조는:

```text
UartTx → serial → UartRx → FIFO
```

였습니다.

여기에 Stream 개념을 적용하면:

```text
UART RX
   │
   │ valid + data
   ▼
Stream
   │
   ▼
StreamFifo
   │
   │ valid + data
   ▼
CPU / Consumer
```

가 됩니다.

그리고 반대쪽 TX는:

```text
CPU / Producer
   │
   │ Stream
   ▼
StreamFifo
   │
   ▼
UART TX
```

가 됩니다.

즉 최종적으로 우리가 만들게 될 구조는:

```text
        RX 방향
UART RX ──► StreamFifo ──► CPU


        TX 방향
CPU ──► StreamFifo ──► UART TX
```

입니다.

**다음부터는 이 구조를 만들면서 `Stream`이 실제 IP 블록 사이에서 어떻게 사용되는지 배우겠습니다.**

---

---

# 다음 단계 — UART RX를 `Stream`으로 만들기

좋습니다. 이제 **지금까지 만든 UART + FIFO를 `Stream` 기반으로 한 단계 발전**시키겠습니다.

지금까지의 구조는:

```text
UartRx
   │
   │ valid
   │ data
   ▼
SimpleFifo
```

였습니다.

이제 이것을:

```text
UartRx
   │
   │ Stream
   ▼
StreamFifo
   │
   │ Stream
   ▼
Consumer
```

로 바꿔보겠습니다.

---

# 1. 먼저 UartRx의 현재 인터페이스

현재 `UartRx`는:

```scala
val io = new Bundle {
  val rx = in Bool()

  val data  = out Bits(8 bits)
  val valid = out Bool()
}
```

입니다.

즉 UART RX가 한 바이트를 받으면:

```text
data  = 받은 데이터
valid = 1
```

이 됩니다.

그런데 문제가 하나 있습니다.

UART RX에는 현재:

```text
ready
```

가 없습니다.

따라서 수신한 데이터가 준비되면 **무조건 valid를 발생시킵니다.**

---

# 2. Stream으로 바꾸면

UART RX의 출력을 바로:

```scala
master Stream(Bits(8 bits))
```

로 만들 수 있습니다.

개념적으로:

```scala id="6x5u9v"
val io = new Bundle {
  val rx = in Bool()

  val output = master Stream(Bits(8 bits))
}
```

그리고 기존:

```text
data
valid
```

가:

```text
output.payload
output.valid
```

로 바뀝니다.

추가로:

```text
output.ready
```

가 생깁니다.

---

# 3. 그런데 UART RX에 `ready`가 필요한 이유

여기서 중요한 문제가 있습니다.

UART는 이미 **직렬 데이터 수신을 시작한 뒤에는** 중간에 멈출 수 없습니다.

예를 들어:

```text
UART serial
    ↓
START
D0
D1
D2
D3
D4
D5
D6
D7
STOP
```

이 프레임을 받는 동안:

```text
ready = 0
```

이라고 해서 UART RX가:

> "Consumer가 준비될 때까지 UART 수신을 멈추겠습니다."

라고 할 수는 없습니다.

UART 송신기가 이미 데이터를 보내고 있기 때문입니다.

---

# 4. 그래서 UART RX + Stream에서는 중요한 설계 문제가 생깁니다

일반적인 Stream은:

```text
valid = 1
ready = 0
```

이면 Producer가 payload를 유지하면서 기다립니다.

하지만 UART RX는:

```text
한 바이트 수신 완료
        ↓
valid = 1
```

이 된 뒤 Consumer가 준비되지 않았다고 해서 무한히 새로운 UART 수신을 막을 수는 없습니다.

따라서 일반적으로:

```text
UART RX
   ↓
FIFO
   ↓
Stream
```

처럼 **FIFO가 완충 역할**을 하는 구조가 중요합니다.

---

# 5. 가장 현실적인 구조

따라서 우리는 UART RX 자체를 무리하게 Stream Producer로 만드는 것보다:

```text
             byte event
UART RX ─────────────────► FIFO
                             │
                             │ Stream
                             ▼
                         Consumer
```

구조를 사용합니다.

조금 더 정확하게 표현하면:

```text
┌──────────┐
│  UartRx  │
└────┬─────┘
     │
     │ valid
     │ data
     ▼
┌────────────┐
│ StreamFifo │
└─────┬──────┘
      │
      │ valid
      │ payload
      ▼
   Consumer
```

---

# 6. FIFO 입력은 사실상 Stream 입력입니다

앞에서 만든 `StreamFifo`는:

```scala
val io = new Bundle {
  val input  = slave Stream(Bits(dataWidth bits))
  val output = master Stream(Bits(dataWidth bits))
}
```

입니다.

그런데 UART RX는:

```text
valid
data
```

만 있습니다.

그래서 연결할 때:

```scala
fifo.io.input.valid   := rx.io.valid
fifo.io.input.payload := rx.io.data
```

를 할 수 있습니다.

문제는:

```text
fifo.io.input.ready
```

입니다.

UART RX는 현재 `ready`를 받지 않기 때문입니다.

---

# 7. 현재 교육용 구조에서는 이렇게 할 수 있습니다

```scala
fifo.io.input.valid   := rx.io.valid
fifo.io.input.payload := rx.io.data
```

그리고 FIFO가 full이면:

```scala
fifo.io.input.ready = 0
```

인데 UART RX는 그것을 사용하지 않습니다.

따라서:

```text
UART RX valid
      │
      ▼
FIFO full?
   │
   ├── NO  → 저장
   │
   └── YES → 데이터 손실
```

입니다.

이것은 우리가 앞에서 이미 경험했던 **UART RX FIFO overflow 문제**입니다.

---

# 8. 이것이 중요한 설계 포인트

따라서:

```text
UART RX → FIFO
```

에서는 단순히 Stream 인터페이스만 붙인다고 문제가 해결되는 것이 아닙니다.

**UART의 물리적인 특성 때문에 backpressure를 UART RX까지 전달할 수 없는 경우가 있습니다.**

그래서 FIFO가 충분히 깊어야 하고, overflow 처리도 고려해야 합니다.

예를 들어:

```text
UART
 ↓
RX
 ↓
FIFO depth 16
 ↓
CPU
```

처럼 버퍼를 둡니다.

---

# 9. 이제 반대 방향은 훨씬 자연스럽습니다

UART TX 쪽은 오히려 Stream과 아주 잘 맞습니다.

UART TX는 현재:

```scala
val io = new Bundle {
  val data  = in Bits(8 bits)
  val valid = in Bool()

  val tx   = out Bool()
  val busy = out Bool()
}
```

입니다.

이것을 Stream 입력으로 생각하면:

```text
UART TX

input.valid
input.payload
input.ready
```

가 됩니다.

그리고 `ready`는:

```text
UART TX가 현재 새로운 데이터를 받을 수 있는가?
```

를 의미하게 만들 수 있습니다.

즉:

```text
Producer
   │
   │ Stream
   ▼
UART TX
```

가 상당히 자연스럽습니다.

---

# 10. UART TX + Stream

개념적으로:

```scala
val io = new Bundle {
  val input = slave Stream(Bits(8 bits))
  val tx = out Bool()
}
```

가 됩니다.

그리고 UART TX가 IDLE일 때:

```text
input.ready = 1
```

BUSY일 때:

```text
input.ready = 0
```

로 만들 수 있습니다.

그러면:

```text
Producer
   │
   │ valid + payload
   ▼
UART TX
   ▲
   │ ready
```

가 됩니다.

---

# 11. 그리고 `fire`가 발생하면 전송 시작

UART TX에서:

```scala
when(io.input.fire) {
    // payload를 저장
    // START bit 전송 시작
}
```

이 됩니다.

이것이 아주 깔끔합니다.

기존에는:

```text
valid
busy
```

를 따로 관리해야 했습니다.

Stream에서는:

```text
input.fire
```

하나로:

> "새로운 UART byte가 TX에 전달되었다."

를 표현할 수 있습니다.

---

# 12. 전체 UART 시스템이 이렇게 발전합니다

우리가 지금까지 만든 구조:

```text
        RX
UART ───────► FIFO ─────► CPU
```

를 조금 더 발전시키면:

```text
             RX
Serial ───► UartRx
              │
              ▼
          StreamFifo
              │
              ▼
             CPU
```

TX 방향:

```text
CPU
 │
 ▼
StreamFifo
 │
 ▼
UartTx ───► Serial
```

결국:

```text
                 ┌─────────────┐
Serial RX ──────►│   UartRx    │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │ StreamFifo  │
                 └──────┬──────┘
                        ▼
                       CPU


                       CPU
                        │
                        ▼
                 ┌─────────────┐
                 │ StreamFifo  │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │   UartTx    │
                 └──────┬──────┘
                        │
                        ▼
                    Serial TX
```

이것이 실제 FPGA에서 볼 수 있는 상당히 일반적인 구조입니다.

---

# 13. 이번 단계의 핵심

이번에는 코드를 많이 작성하는 것보다 **인터페이스 설계 관점**을 이해하는 것이 중요합니다.

### UART RX

```text
UART RX → FIFO
```

에서는 UART의 특성 때문에 `ready`를 단순하게 전달하기 어렵습니다.

### UART TX

```text
Stream → UART TX
```

에서는 `ready`를 이용해 아주 자연스럽게 backpressure를 만들 수 있습니다.

즉:

```text
RX 방향
UART → FIFO
```

와

```text
TX 방향
FIFO → UART
```

는 **같은 Stream 구조를 그대로 적용할 수 있는 것이 아닙니다.**

이 차이를 이해하는 것이 중요합니다.

---

# 다음 단계

다음에는 **`UartTx`를 `Stream` 입력으로 변경**해 보겠습니다.

그때 핵심 코드는:

```scala
when(io.input.fire) {
    // 새로운 byte 수락
}
```

그리고:

```scala
io.input.ready := state === State.IDLE
```

입니다.

이렇게 하면 우리가 지금까지 배운:

```text
Stream
 ↓
valid / ready
 ↓
fire
 ↓
FIFO
 ↓
UART
```

가 실제 하나의 하드웨어 설계로 연결됩니다.



