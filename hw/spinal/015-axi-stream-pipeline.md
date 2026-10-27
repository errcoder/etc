좋습니다. 이번 단계에서는 요청하신 대로 **1, 2는 핵심만 짚고**, **3, 4, 5는 SpinalHDL 1.15.0 기준 예제 코드까지 작성**해서 여기까지 한 번에 마무리하겠습니다.

전체 흐름은:

```text
① Stream vs Flow              ← 간단히
        ↓
② StreamFifo vs Library FIFO  ← 간단히
        ↓
③ AXI-Stream                  ← 예제
        ↓
④ Stream Pipeline             ← 예제
        ↓
⑤ Pipeline + FIFO + Backpressure ← 예제
```

---

# 1. Stream vs Flow — 핵심 정리

이미 배운 내용을 한 단계 정리하면 됩니다.

### Stream

```text
valid + payload + ready
```

```text
Producer                    Consumer

valid ───────────────────────►
payload ─────────────────────►
ready ◄───────────────────────
```

전송:

```scala
stream.fire
```

즉:

```scala
stream.valid && stream.ready
```

입니다.

**Consumer가 받을 수 없으면 Producer가 기다릴 수 있습니다.**

---

### Flow

```text
valid + payload
```

```text
Producer                    Consumer

valid ───────────────────────►
payload ─────────────────────►
```

`ready`가 없습니다.

따라서 backpressure도 없습니다.

UART RX의:

```text
data + valid
```

같은 **"이벤트가 발생했다"**는 인터페이스에 적합합니다.

### 한 줄 요약

> **Stream = 데이터를 전달하면서 상대방의 수신 가능 여부까지 제어**

> **Flow = 데이터/이벤트가 발생했음을 알림**

---

# 2. StreamFifo vs SpinalHDL Library FIFO — 핵심 정리

우리가 직접 만든 `StreamFifo`는 학습에 매우 좋았습니다.

우리가 직접 구현했던 것은:

```text
memory
writePtr
readPtr
count
push
pop
full
empty
```

입니다.

그리고:

```scala
io.input.fire
```

가 발생하면:

```text
push
```

```scala
io.output.fire
```

가 발생하면:

```text
pop
```

입니다.

---

실제 프로젝트에서는 이런 FIFO를 매번 직접 작성하기보다는 **SpinalHDL에서 제공하는 검증된 FIFO/Stream 컴포넌트**를 사용하는 것이 일반적으로 더 좋습니다.

특히 실제 FIFO에서는:

- synchronous RAM
- asynchronous FIFO
- clock domain crossing
- read latency
- simultaneous read/write
- full/empty 경계조건
- RAM inference

등을 고려해야 하기 때문입니다.

따라서 우리가 만든 FIFO는:

> **FIFO의 내부 동작을 배우기 위한 구현**

이고,

Library FIFO는:

> **실제 설계에서 재사용하기 위한 검증된 구현**

이라고 이해하면 됩니다.

---

# 3. AXI-Stream

이제 중요한 단계입니다.

우리가 배운 `Stream`을 알고 있으면 AXI-Stream의 기본 개념은 상당히 쉽게 이해할 수 있습니다.

## 3.1 AXI-Stream의 핵심

AXI-Stream의 가장 기본적인 신호는:

```text
TVALID
TREADY
TDATA
```

입니다.

SpinalHDL `Stream`과 비교하면:

| SpinalHDL | AXI-Stream |
|---|---|
| `valid` | `TVALID` |
| `ready` | `TREADY` |
| `payload` | `TDATA` |

즉:

```text
SpinalHDL Stream

valid
payload
ready
```

와

```text
AXI-Stream

TVALID
TDATA
TREADY
```

는 기본적인 handshake 철학이 같습니다.

---

## 3.2 AXI-Stream의 핵심 handshake

가장 중요한 공식은 역시:

```text
TVALID && TREADY
```

입니다.

```text
TVALID = 0
TREADY = 0
→ 전송 없음

TVALID = 0
TREADY = 1
→ 전송 없음

TVALID = 1
TREADY = 0
→ 전송 없음
→ 데이터 유지

TVALID = 1
TREADY = 1
→ 전송 발생
```

특히:

```text
TVALID = 1
TREADY = 0
```

일 때 Producer는 **TDATA를 바꾸면 안 됩니다.**

이것이 AXI-Stream의 중요한 기본 규칙입니다.

---

# 4. SpinalHDL에서 AXI-Stream 형태 만들기

우리가 AXI-Stream을 처음 배울 때는 복잡한 AXI4 인터페이스 전체를 사용하기보다, 가장 기본적인:

```text
TDATA
TVALID
TREADY
```

만 직접 만들어보는 것이 좋습니다.

## 4.1 간단한 AXI-Stream 인터페이스

```scala id="ax1t5p"
import spinal.core._

case class AxiStreamPayload(
    dataWidth: Int
) extends Bundle {
  val data = Bits(dataWidth bits)
}
```

그리고:

```scala id="j0y3x8"
import spinal.core._

case class AxiStream(
    dataWidth: Int
) extends Bundle {

  val tvalid = Bool()
  val tready = Bool()
  val tdata  = Bits(dataWidth bits)
}
```

하지만 여기서 `tvalid/tready`의 방향은 사용하는 쪽에서 달라지기 때문에, 실제 설계에서는 SpinalHDL의 `Stream`을 사용하는 것이 훨씬 자연스럽습니다.

즉:

```scala
master Stream(...)
```

과

```scala
slave Stream(...)
```

을 사용해서 AXI-Stream과 동일한 handshake 구조를 만들 수 있습니다.

---

# 5. AXI-Stream 스타일 Producer

먼저 데이터를 계속 발생시키는 Producer를 만들어봅시다.

```scala id="c5r4we"
import spinal.core._

case class AxiStreamProducer(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val output =
      master Stream(UInt(dataWidth bits))
  }

  val counter =
    Reg(UInt(dataWidth bits)) init(0)

  io.output.valid :=
    True

  io.output.payload :=
    counter

  when(io.output.fire) {

    counter :=
      counter + 1
  }
}
```

동작은:

```text
counter = 0

valid = 1
data  = 0
ready = 1
```

이면:

```text
fire = 1
```

따라서 다음 clock:

```text
counter = 1
```

그 다음:

```text
2
3
4
...
```

가 됩니다.

---

# 6. AXI-Stream Consumer

Consumer는:

```scala id="y9d3xq"
import spinal.core._

case class AxiStreamConsumer(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))
  }

  io.input.ready :=
    True

  when(io.input.fire) {

    // 데이터 소비
    // io.input.payload
  }
}
```

여기서:

```scala
io.input.ready := True
```

이므로 Consumer는 항상 데이터를 받을 수 있습니다.

---

# 7. AXI-Stream에서 Backpressure 발생시키기

이번에는 Consumer가 항상 받을 수 있는 것이 아니라:

```text
ready = 0
```

이 될 수 있게 해봅시다.

```scala id="w6y9d2"
import spinal.core._

case class AxiStreamConsumer(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val enable =
      in Bool()
  }

  io.input.ready :=
    io.enable

  when(io.input.fire) {

    // 데이터 소비
  }
}
```

이제:

```text
enable = 0
```

이면:

```text
ready = 0
```

입니다.

Producer가:

```text
valid = 1
```

을 유지하고 있어도:

```text
valid = 1
ready = 0
```

이므로:

```text
fire = 0
```

입니다.

따라서 Producer의 데이터는 유지되어야 합니다.

이것이 바로 **AXI-Stream backpressure**입니다.

---

# 8. Stream Pipeline

이제 더 재미있는 부분으로 넘어갑니다.

예를 들어 데이터 처리 단계가:

```text
Input
  │
  ▼
Stage 1
  │
  ▼
Stage 2
  │
  ▼
Stage 3
  │
  ▼
Output
```

이라고 합시다.

각 stage 사이를 `Stream`으로 연결할 수 있습니다.

---

# 9. Pipeline Stage 1 — 데이터 +1

```scala id="r4m8cj"
import spinal.core._

case class PipelineAddOne(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  io.output.valid :=
    io.input.valid

  io.output.payload :=
    io.input.payload + 1

  io.input.ready :=
    io.output.ready
}
```

이것은 **조합 논리 pipeline stage**입니다.

핵심은:

```scala
io.output.valid := io.input.valid
```

그리고:

```scala
io.input.ready := io.output.ready
```

입니다.

---

# 10. Pipeline Stage 2 — ×2

```scala id="u6q4jx"
import spinal.core._

case class PipelineMultiplyTwo(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  io.output.valid :=
    io.input.valid

  io.output.payload :=
    io.input.payload * 2

  io.input.ready :=
    io.output.ready
}
```

따라서:

```text
input
  │
  ▼
+1
  │
  ▼
×2
  │
  ▼
output
```

입니다.

입력이 `5`라면:

```text
5
 ↓
+1
 ↓
6
 ↓
×2
 ↓
12
```

---

# 11. Pipeline Stage 3 — +10

```scala id="v2x8f0"
import spinal.core._

case class PipelineAddTen(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  io.output.valid :=
    io.input.valid

  io.output.payload :=
    io.input.payload + 10

  io.input.ready :=
    io.output.ready
}
```

전체 계산은:

```text
input
 ↓
+1
 ↓
×2
 ↓
+10
 ↓
output
```

따라서:

```text
5
 ↓
6
 ↓
12
 ↓
22
```

입니다.

---

# 12. Pipeline 전체 연결

```scala id="q3n6sv"
import spinal.core._

case class StreamPipeline(
    dataWidth: Int
) extends Component {

  val io = new Bundle {

    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  val stage1 =
    PipelineAddOne(dataWidth)

  val stage2 =
    PipelineMultiplyTwo(dataWidth)

  val stage3 =
    PipelineAddTen(dataWidth)

  stage1.io.input <<
    io.input

  stage2.io.input <<
    stage1.io.output

  stage3.io.input <<
    stage2.io.output

  io.output <<
    stage3.io.output
}
```

데이터 흐름:

```text
                  Stream
                    │
                    ▼
             ┌─────────────┐
             │    +1       │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │    ×2       │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │    +10      │
             └──────┬──────┘
                    │
                    ▼
                  Output
```

---

# 13. 그런데 이것은 아직 "진짜 Register Pipeline"은 아니다

여기서 중요한 구분이 있습니다.

현재 Stage는:

```text
Stream
 ↓
조합논리
 ↓
Stream
 ↓
조합논리
 ↓
Stream
```

입니다.

즉 실제 clocked pipeline register가 없습니다.

실제 고속 디지털 회로에서는:

```text
Input
  │
  ▼
┌───────┐
│ +1    │
└───┬───┘
    │
   REG
    │
    ▼
┌───────┐
│ ×2    │
└───┬───┘
    │
   REG
    │
    ▼
┌───────┐
│ +10   │
└───┬───┘
    │
   REG
    │
    ▼
Output
```

처럼 만들어야 합니다.

그리고 여기서 `Stream`이 굉장히 강력해집니다.

---

# 14. Pipeline + FIFO + Backpressure

이번 단계의 핵심입니다.

이제 pipeline 중간에 FIFO를 넣어보겠습니다.

```text
Input
  │
  ▼
Stage 1
  │
  ▼
┌─────────────┐
│  StreamFifo  │
└──────┬──────┘
       │
       ▼
Stage 2
       │
       ▼
Output
```

FIFO가 데이터를 저장할 수 있으므로 Stage 1과 Stage 2의 처리 속도가 순간적으로 달라도 흡수할 수 있습니다.

---

# 15. Pipeline + StreamFifo 예제

우리가 앞에서 만든 `StreamFifo`를 그대로 사용할 수 있습니다.

```scala id="r7x4b2"
import spinal.core._

case class PipelineWithFifo(
    dataWidth: Int,
    fifoDepth: Int
) extends Component {

  val io = new Bundle {

    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  val stage1 =
    PipelineAddOne(dataWidth)

  val fifo =
    StreamFifo(
      dataWidth = dataWidth,
      depth = fifoDepth
    )

  val stage2 =
    PipelineMultiplyTwo(dataWidth)

  stage1.io.input <<
    io.input

  fifo.io.input <<
    stage1.io.output

  stage2.io.input <<
    fifo.io.output

  io.output <<
    stage2.io.output
}
```

데이터 흐름:

```text
                 Stream
                   │
                   ▼
              ┌─────────┐
              │   +1    │
              └────┬────┘
                   │
                   ▼
            ┌──────────────┐
            │  StreamFifo  │
            │              │
            │ A B C D ...  │
            └──────┬───────┘
                   │
                   ▼
              ┌─────────┐
              │   ×2    │
              └────┬────┘
                   │
                   ▼
                 Output
```

---

# 16. FIFO가 Full이면 어떻게 되는가?

이 부분이 이번 단계의 핵심입니다.

우리가 만든 `StreamFifo`는:

```scala
io.input.ready := !full
```

이었습니다.

따라서:

```text
FIFO FULL
    ↓
input.ready = 0
```

입니다.

그러면 FIFO 앞의 Stage 1은:

```text
output.ready = 0
```

을 받습니다.

그리고 Stage 1은:

```scala
io.input.ready :=
    io.output.ready
```

이므로:

```text
Stage1.ready = 0
```

이 됩니다.

그리고 이것이 다시 upstream으로 전달됩니다.

```text
Input
  │
  │ ready = 0
  ▼
Stage 1
  │
  │ ready = 0
  ▼
FIFO FULL
```

즉:

> **FIFO의 FULL 상태가 upstream으로 backpressure를 전파합니다.**

---

# 17. 전체 Backpressure 구조

이것을 조금 더 크게 보면:

```text
                 BACKPRESSURE
                     ◄──────────────

Producer
   │
   │ Stream
   ▼
┌─────────┐
│ Stage 1 │
└────┬────┘
     │
     ▼
┌───────────┐
│ StreamFifo│
│   FULL    │
└─────┬─────┘
      │
      ▼
┌─────────┐
│ Stage 2 │
└────┬────┘
     │
     ▼
 Consumer
```

Consumer가 느려지면:

```text
Consumer
   │
   │ ready = 0
   ▼
Stage 2
   │
   │ ready = 0
   ▼
FIFO
```

FIFO가 가득 차면:

```text
FIFO
 │
 │ ready = 0
 ▼
Stage 1
 │
 │ ready = 0
 ▼
Producer
```

까지 backpressure가 전파됩니다.

---

# 18. 이것이 우리가 UART에서 배운 것과 연결된다

지금까지 배운 것을 연결해보면 상당히 재미있습니다.

우리가 만들었던 구조:

```text
UART RX
   │
   ▼
RX FIFO
   │
   ▼
CPU
```

에서 UART RX는 현재:

```text
valid
data
```

만 제공합니다.

따라서 FIFO가 FULL이어도 UART RX가 계속 데이터를 받을 수 있습니다.

결과적으로:

```text
UART RX
   │
   │ valid
   ▼
RX FIFO FULL
   │
   X
   │
   └── byte loss
```

가 될 수 있었습니다.

반면 Stream 기반 구조라면:

```text
UART RX Stream
      │
      ▼
  StreamFifo
      │
      ▼
     CPU
```

에서:

```text
FIFO FULL
    ↓
ready = 0
    ↓
UART RX가 받을 수 있는지 판단
```

하는 구조를 만들 수 있습니다.

다만 **UART 물리 인터페이스 자체는 TCP처럼 무한히 기다려 줄 수 있는 프로토콜이 아니므로**, 실제 UART에서는 FIFO full에 대한 별도의 overflow 처리나 flow-control(RTS/CTS 등)이 필요합니다.

이것이 `Stream`을 배운 뒤 실제 시스템 설계에서 생각해야 할 중요한 부분입니다.

---

# 19. 이번 단계에서 배운 것 정리

이번 단계까지 오면서 데이터 전달 구조가 상당히 명확해졌습니다.

```text
Flow
 │
 │ valid + payload
 ▼
이벤트 전달
```

```text
Stream
 │
 │ valid + payload + ready
 ▼
Handshake 데이터 전달
```

```text
StreamFifo
 │
 │ Stream + 저장 공간
 ▼
Backpressure 흡수
```

```text
AXI-Stream
 │
 │ TVALID + TDATA + TREADY
 ▼
표준화된 Stream 인터페이스
```

그리고:

```text
Stream
   ↓
Pipeline
   ↓
FIFO
   ↓
Backpressure
```

를 연결했습니다.

---

# 20. 지금까지의 전체 학습 구조

현재까지 우리가 만든 학습 시스템을 보면:

```text
                 UART
                  │
          ┌───────┴───────┐
          │               │
       UartTx           UartRx
          │               │
          ▼               ▼
       Stream           Stream/Flow
          │               │
          ▼               ▼
       TX FIFO          RX FIFO
          │               │
          └──────┬────────┘
                 │
              Pipeline
                 │
          ┌──────┼──────┐
          │      │      │
         +1     ×2     +10
          │      │      │
          └──────┴──────┘
                 │
             Backpressure
```

여기까지 오면 **SpinalHDL에서 Stream을 왜 중요한 기본 abstraction으로 사용하는지**가 상당히 명확해집니다.

### 이번 단계의 핵심 5개

1. **Stream vs Flow**
   - Flow: `valid + payload`
   - Stream: `valid + payload + ready`

2. **StreamFifo vs Library FIFO**
   - 직접 만든 FIFO → 구조 학습
   - Library FIFO → 실전 재사용

3. **AXI-Stream**
   - `TVALID / TREADY / TDATA`
   - `valid && ready`가 transfer

4. **Stream Pipeline**
   - 여러 처리 stage를 Stream으로 연결
   - 각 stage가 독립적으로 handshake

5. **Pipeline + FIFO + Backpressure**
   - FIFO가 full → `ready=0`
   - `ready=0`이 upstream으로 전파
   - 느린 downstream이 전체 pipeline을 제어

이제 다음 단계에서는 **실제 SpinalHDL 라이브러리의 FIFO/Stream 컴포넌트를 사용해서 우리가 직접 만든 `StreamFifo`와 비교하고**, 그 다음 **clocked pipeline / pipeline register / latency / throughput**을 제대로 다루는 단계로 넘어가면 좋습니다.


---

---

좋습니다. 이번에는 **학습용 조합 Stage가 아니라 실제로 clocked 3-stage pipeline + SpinalHDL `StreamFifo` + backpressure를 포함한 완성형 예제**로 만들겠습니다.

한 가지는 명확히 하겠습니다. 여기서 "AXI-Stream 기반"은 **AXI4-Stream의 핵심 handshake인 `VALID/READY/DATA`를 SpinalHDL `Stream`으로 구현**하겠습니다. 즉 실제 AXI4-Stream IP 포트의 `TVALID/TREADY/TDATA`와 대응되는 구조입니다. SpinalHDL의 `Stream` 자체가 `valid`, `ready`, `payload`를 제공하며, transfer는 `valid && ready`입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/dev/artefacts/SpinalHDL_docs-dev.pdf?utm_source=chatgpt.com)

그리고 FIFO는 직접 만든 `StreamFifo`가 아니라 **SpinalHDL library의 `spinal.lib.StreamFifo`**를 사용하겠습니다. 공식 API에는 `dataType`, `depth`, `withAsyncRead`, `withBypass` 등의 옵션이 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

---

# 1. 최종 구조

이번 예제의 데이터 경로는 다음과 같습니다.

```text
                  AXI-Stream style
                       Stream
                         │
                         ▼
                 ┌──────────────┐
                 │  Pipeline 1  │
                 │     +1       │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  Pipeline 2  │
                 │     ×2       │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  Pipeline 3  │
                 │    +10       │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Library      │
                 │ StreamFifo   │
                 │ depth = 4    │
                 └──────┬───────┘
                        │
                        ▼
                    AXI-Stream
                     Consumer
```

그리고 Consumer가 `ready = 0`이 되면:

```text
Consumer
   │
   │ ready = 0
   ▼
StreamFifo
   │
   │ full
   ▼
Pipeline 3
   │
   ▼
Pipeline 2
   │
   ▼
Pipeline 1
   │
   ▼
Producer
```

방향으로 **backpressure가 upstream으로 전파**됩니다.

---

# 2. 이번 예제에서 중요한 설계

단순히:

```text
+1 → ×2 → +10
```

만 하는 것이 아니라 각 stage를 **1-cycle register stage**로 만들겠습니다.

따라서 실제 hardware 구조는:

```text
Input
  │
  ▼
[ +1 combinational ]
  │
  ▼
[ Register ]
  │
  ▼
[ ×2 combinational ]
  │
  ▼
[ Register ]
  │
  ▼
[ +10 combinational ]
  │
  ▼
[ Register ]
  │
  ▼
FIFO
```

형태입니다.

이렇게 해야 실제 pipeline의 **latency와 throughput**을 학습할 수 있습니다.

---

# 3. 전체 프로젝트 파일

다음처럼 파일을 구성하는 것을 추천합니다.

```text
src/main/scala/
    AxiStreamPipeline.scala

src/test/scala/
    AxiStreamPipelineSim.scala
```

실제로는 simulation object도 `src/main/scala`에 넣어도 됩니다.

---

# 4. 완성된 `AxiStreamPipeline.scala`

**SpinalHDL 1.15.0 기준**입니다.

```scala
import spinal.core._
import spinal.lib._

/**
 * One registered AXI-Stream style pipeline stage.
 *
 * Operation:
 *   output = input + 1
 *
 * Interface:
 *   Stream = valid + payload + ready
 */
case class PipelineStageAddOne(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  val dataReg =
    Reg(UInt(dataWidth bits)) init(0)

  val validReg =
    Reg(Bool()) init(False)

  /*
   * This stage can accept new data when:
   *
   *   - it is currently empty, OR
   *   - the current output will be consumed this cycle.
   */
  io.input.ready :=
    !validReg || io.output.ready

  io.output.valid :=
    validReg

  io.output.payload :=
    dataReg

  when(io.input.ready) {

    validReg :=
      io.input.valid

    when(io.input.valid) {

      dataReg :=
        io.input.payload + 1
    }
  }
}


/**
 * One registered AXI-Stream style pipeline stage.
 *
 * Operation:
 *   output = input * 2
 */
case class PipelineStageMultiplyTwo(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  val dataReg =
    Reg(UInt(dataWidth bits)) init(0)

  val validReg =
    Reg(Bool()) init(False)

  io.input.ready :=
    !validReg || io.output.ready

  io.output.valid :=
    validReg

  io.output.payload :=
    dataReg

  when(io.input.ready) {

    validReg :=
      io.input.valid

    when(io.input.valid) {

      dataReg :=
        io.input.payload * 2
    }
  }
}


/**
 * One registered AXI-Stream style pipeline stage.
 *
 * Operation:
 *   output = input + 10
 */
case class PipelineStageAddTen(
    dataWidth: Int
) extends Component {

  val io = new Bundle {
    val input =
      slave Stream(UInt(dataWidth bits))

    val output =
      master Stream(UInt(dataWidth bits))
  }

  val dataReg =
    Reg(UInt(dataWidth bits)) init(0)

  val validReg =
    Reg(Bool()) init(False)

  io.input.ready :=
    !validReg || io.output.ready

  io.output.valid :=
    validReg

  when(io.input.ready) {

    validReg :=
      io.input.valid

    when(io.input.valid) {

      dataReg :=
        io.input.payload + 10
    }
  }
}


/**
 * Complete 3-stage AXI-Stream style pipeline
 * followed by a SpinalHDL library StreamFifo.
 *
 * Data transformation:
 *
 *   input
 *     ↓
 *   +1
 *     ↓
 *   ×2
 *     ↓
 *   +10
 *     ↓
 *   StreamFifo
 *     ↓
 *   output
 *
 * Result:
 *
 *   output = (input + 1) * 2 + 10
 */
case class AxiStreamPipelineWithFifo(
    dataWidth: Int,
    fifoDepth: Int
) extends Component {

  require(dataWidth >= 1)
  require(fifoDepth >= 2)

  val io = new Bundle {

    // AXI-Stream style input
    val input =
      slave Stream(UInt(dataWidth bits))

    // AXI-Stream style output
    val output =
      master Stream(UInt(dataWidth bits))
  }

  // ------------------------------------------------------------
  // Pipeline stages
  // ------------------------------------------------------------

  val stage1 =
    PipelineStageAddOne(dataWidth)

  val stage2 =
    PipelineStageMultiplyTwo(dataWidth)

  val stage3 =
    PipelineStageAddTen(dataWidth)

  // ------------------------------------------------------------
  // SpinalHDL library StreamFifo
  // ------------------------------------------------------------

  val fifo =
    StreamFifo(
      dataType = UInt(dataWidth bits),
      depth = fifoDepth
    )

  // ------------------------------------------------------------
  // Connections
  // ------------------------------------------------------------

  stage1.io.input <<
    io.input

  stage2.io.input <<
    stage1.io.output

  stage3.io.input <<
    stage2.io.output

  fifo.io.push <<
    stage3.io.output

  io.output <<
    fifo.io.pop
}
```

---

# 5. 여기서 가장 중요한 Pipeline 코드

각 stage에 똑같은 패턴이 있습니다.

예를 들어 Stage 1:

```scala
io.input.ready :=
    !validReg || io.output.ready
```

이 부분이 핵심입니다.

현재 stage에 데이터가 없으면:

```text
validReg = 0
```

따라서:

```text
ready = 1
```

입니다.

또는 현재 데이터가 있지만 downstream이 이번 cycle에 데이터를 소비할 수 있으면:

```text
output.ready = 1
```

이므로:

```text
ready = 1
```

입니다.

즉:

```text
                current data
                     │
                     ▼
               ┌───────────┐
input ────────►│ REGISTER  │───────► output
               └───────────┘
                    │
             validReg / ready
```

형태의 **elastic pipeline stage**입니다.

---

# 6. 왜 단순한 `valid := input.valid`가 아닌가?

앞에서 우리가 처음 배운 pipeline에서는:

```scala
io.output.valid :=
    io.input.valid
```

처럼 했습니다.

그것은 조합적인 pass-through입니다.

이번에는:

```scala
val dataReg  = Reg(...)
val validReg = Reg(...)
```

를 사용했습니다.

따라서 실제로 데이터를 저장합니다.

예를 들어 downstream이:

```text
ready = 0
```

이라면:

```text
valid = 1
```

인 데이터를 register에 계속 유지합니다.

즉 AXI-Stream의 핵심 규칙:

> `VALID=1`, `READY=0`이면 payload를 유지한다.

를 실제 register로 구현한 것입니다.

---

# 7. Library `StreamFifo`

이번에는 우리가 직접 만들었던:

```scala
case class StreamFifo(...)
```

를 사용하지 않았습니다.

대신:

```scala
val fifo =
    StreamFifo(
      dataType = UInt(dataWidth bits),
      depth = fifoDepth
    )
```

를 사용했습니다.

SpinalHDL의 library `StreamFifo`는 `push`와 `pop` Stream 인터페이스를 제공하며 FIFO latency와 memory/read 방식 등을 설정할 수 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

연결은:

```scala
fifo.io.push <<
    stage3.io.output

io.output <<
    fifo.io.pop
```

입니다.

즉:

```text
stage3.output
      │
      │ Stream
      ▼
fifo.push

fifo.pop
      │
      │ Stream
      ▼
output
```

입니다.

---

# 8. Simulation 코드

이번에는 **단순히 몇 개 넣고 몇 개 빼는 테스트가 아니라**, 실제로 backpressure를 발생시키겠습니다.

테스트 전략은:

### Phase 1

Consumer가 항상 ready:

```text
ready = 1
```

→ 정상 pipeline 동작 확인.

### Phase 2

Consumer가 일정 기간:

```text
ready = 0
```

으로 변경.

그러면 FIFO가 차고 pipeline이 정지하면서 backpressure가 발생합니다.

### Phase 3

다시:

```text
ready = 1
```

→ 저장된 데이터가 다시 순서대로 빠져나오는지 확인.

그리고 Scala의 `Queue`를 **reference model**로 사용하여 실제 출력과 비교하겠습니다. SpinalHDL의 공식 FIFO simulation 예제도 이런 방식으로 Scala `Queue`를 reference model로 사용합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/zh_CN/master/SpinalHDL/Simulation/examples/single_clock_fifo.html?utm_source=chatgpt.com)

---

# 9. `AxiStreamPipelineSim.scala`

```scala
import spinal.core._
import spinal.core.sim._

import scala.collection.mutable.Queue

object AxiStreamPipelineSim extends App {

  val dataWidth = 8
  val fifoDepth = 4

  val compiled =
    SimConfig
      .withWave
      .compile(
        AxiStreamPipelineWithFifo(
          dataWidth = dataWidth,
          fifoDepth = fifoDepth
        )
      )

  compiled.doSim { dut =>

    println(
      "[Progress] Start AxiStreamPipeline simulation"
    )

    dut.clockDomain.forkStimulus(
      period = 10
    )

    // ----------------------------------------------------------
    // Initial values
    // ----------------------------------------------------------

    dut.io.input.valid #=
      false

    dut.io.input.payload #=
      0

    dut.io.output.ready #=
      true

    dut.clockDomain.waitSampling(2)

    // ----------------------------------------------------------
    // Reference model
    // ----------------------------------------------------------

    val expected =
      Queue[Int]()

    // ----------------------------------------------------------
    // Helper: send one input transaction
    // ----------------------------------------------------------

    def send(data: Int): Unit = {

      dut.io.input.payload #=
        data

      dut.io.input.valid #=
        true

      /*
       * Hold VALID until the pipeline accepts the data.
       */
      while (!dut.io.input.ready.toBoolean) {

        dut.clockDomain.waitSampling()
      }

      /*
       * This sampling edge is the actual transfer:
       *
       * input.valid && input.ready
       */
      dut.clockDomain.waitSampling()

      dut.io.input.valid #=
        false

      /*
       * The expected output is:
       *
       *   (data + 1) * 2 + 10
       */
      val result =
        (data + 1) * 2 + 10

      expected.enqueue(result)

      println(
        f"[SEND] input=0x$data%02X expected=0x$result%02X"
      )
    }

    // ----------------------------------------------------------
    // Helper: consume one output if available
    // ----------------------------------------------------------

    def checkOutput(): Unit = {

      if (
        dut.io.output.valid.toBoolean &&
        dut.io.output.ready.toBoolean
      ) {

        assert(
          expected.nonEmpty,
          "DUT produced unexpected output"
        )

        val actual =
          dut.io.output.payload.toInt

        val wanted =
          expected.dequeue()

        println(
          f"[RECV] actual=0x$actual%02X expected=0x$wanted%02X"
        )

        assert(
          actual == wanted,
          f"Data mismatch: actual=0x$actual%02X expected=0x$wanted%02X"
        )
      }
    }

    // ==========================================================
    // TEST 1
    // Normal operation
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 1: Normal AXI-Stream operation")
    println("==========================================")

    dut.io.output.ready #=
      true

    for (data <- 0 until 8) {

      send(data)

      /*
       * Give the pipeline time to advance.
       */
      dut.clockDomain.waitSampling()

      checkOutput()
    }

    /*
     * Drain the remaining pipeline/FIFO contents.
     */
    for (_ <- 0 until 10) {

      dut.clockDomain.waitSampling()

      checkOutput()
    }

    assert(
      expected.isEmpty,
      s"TEST 1 failed: ${expected.size} items remain"
    )

    println(
      "[PASS] TEST 1 normal operation"
    )

    // ==========================================================
    // TEST 2
    // Backpressure
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 2: Backpressure")
    println("==========================================")

    /*
     * Stop the consumer.
     *
     * This should eventually fill the FIFO and
     * propagate backpressure toward the input.
     */
    dut.io.output.ready #=
      false

    println(
      "[BACKPRESSURE] output.ready = false"
    )

    /*
     * Send several transactions while the output
     * side is blocked.
     */
    for (data <- 20 until 40) {

      dut.io.input.payload #=
        data

      dut.io.input.valid #=
        true

      /*
       * Wait until the pipeline/FIFO accepts the item.
       *
       * Once the complete buffering path is full,
       * this loop should stop progressing until
       * the consumer becomes ready again.
       */
      var cycles = 0

      while (!dut.io.input.ready.toBoolean) {

        dut.clockDomain.waitSampling()

        cycles += 1

        /*
         * The test must not hang forever.
         *
         * We deliberately stop sending when the
         * backpressure reaches the input.
         */
        if (cycles > 20) {

          println(
            s"[BACKPRESSURE] input blocked after sending $data"
          )

          dut.io.input.valid #=
            false

          /*
           * Do not add this data to the reference model,
           * because it was never transferred.
           */
          break
        }
      }

      /*
       * If input.ready is true, transfer occurs
       * at the next sampling edge.
       */
      if (dut.io.input.ready.toBoolean) {

        dut.clockDomain.waitSampling()

        expected.enqueue(
          (data + 1) * 2 + 10
        )

        println(
          f"[BUFFERED] input=0x$data%02X"
        )

        dut.io.input.valid #=
          false
      }
    }

    /*
     * Output is still blocked.
     *
     * Therefore no output transaction should fire.
     */
    assert(
      !dut.io.output.ready.toBoolean,
      "Output should still be blocked"
    )

    println(
      s"[CHECK] reference model contains ${expected.size} items"
    )

    // ==========================================================
    // TEST 3
    // Release backpressure
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 3: Release backpressure")
    println("==========================================")

    dut.io.output.ready #=
      true

    println(
      "[BACKPRESSURE] output.ready = true"
    )

    /*
     * Drain the entire system.
     */
    var drainCycles = 0

    while (expected.nonEmpty) {

      dut.clockDomain.waitSampling()

      checkOutput()

      drainCycles += 1

      assert(
        drainCycles < 200,
        "Pipeline did not drain within timeout"
      )
    }

    /*
     * Give one extra cycle to verify that the DUT
     * has no unexpected remaining output.
     */
    dut.clockDomain.waitSampling()

    assert(
      !dut.io.output.valid.toBoolean,
      "DUT should be empty after draining"
    )

    println(
      "[PASS] TEST 3 backpressure released correctly"
    )

    // ==========================================================
    // Final result
    // ==========================================================

    println()
    println("==========================================")
    println("[PASS] ALL TESTS PASSED")
    println("==========================================")
  }
}
```

여기에는 **한 가지 문제가 있습니다.**

Scala에는 C의:

```scala
break
```

가 기본적으로 존재하지 않습니다.

따라서 위 테스트 코드는 그대로 사용하면 안 됩니다.

이 부분을 일부러 짚고 넘어가는 이유는 사용자가 이전 UART/FIFO simulation에서 **simulation timing bug를 직접 경험했기 때문**입니다. 이번에는 그런 코드를 그대로 제공하지 않겠습니다.

---

# 10. 수정된 최종 Simulation 코드

`break` 없이 명확하게 상태를 제어하는 형태로 바꾸겠습니다.

그리고 이번에는 backpressure 테스트도 더 결정적으로 만들겠습니다.

**출력을 막은 상태에서 입력을 계속 시도하여 실제로 `input.ready`가 0이 되는 것을 확인**합니다.

```scala
import spinal.core._
import spinal.core.sim._

import scala.collection.mutable.Queue

object AxiStreamPipelineSim extends App {

  val dataWidth = 8
  val fifoDepth = 4

  val compiled =
    SimConfig
      .withWave
      .compile(
        AxiStreamPipelineWithFifo(
          dataWidth = dataWidth,
          fifoDepth = fifoDepth
        )
      )

  compiled.doSim { dut =>

    println(
      "[Progress] Start AxiStreamPipeline simulation"
    )

    dut.clockDomain.forkStimulus(
      period = 10
    )

    dut.io.input.valid #=
      false

    dut.io.input.payload #=
      0

    dut.io.output.ready #=
      true

    dut.clockDomain.waitSampling(2)

    val expected =
      Queue[Int]()

    // ----------------------------------------------------------
    // Send one transaction
    // ----------------------------------------------------------

    def send(data: Int): Unit = {

      dut.io.input.payload #=
        data

      dut.io.input.valid #=
        true

      while (!dut.io.input.ready.toBoolean) {

        dut.clockDomain.waitSampling()
      }

      dut.clockDomain.waitSampling()

      dut.io.input.valid #=
        false

      expected.enqueue(
        (data + 1) * 2 + 10
      )

      println(
        f"[SEND] input=0x$data%02X expected=0x${(data + 1) * 2 + 10}%02X"
      )
    }

    // ----------------------------------------------------------
    // Check output
    // ----------------------------------------------------------

    def checkOutput(): Unit = {

      if (
        dut.io.output.valid.toBoolean &&
        dut.io.output.ready.toBoolean
      ) {

        assert(
          expected.nonEmpty,
          "Unexpected DUT output"
        )

        val actual =
          dut.io.output.payload.toInt

        val wanted =
          expected.dequeue()

        println(
          f"[RECV] actual=0x$actual%02X expected=0x$wanted%02X"
        )

        assert(
          actual == wanted,
          f"Mismatch: actual=0x$actual%02X expected=0x$wanted%02X"
        )
      }
    }

    // ==========================================================
    // TEST 1
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 1: Normal operation")
    println("==========================================")

    for (data <- 0 until 8) {

      send(data)

      dut.clockDomain.waitSampling()

      checkOutput()
    }

    for (_ <- 0 until 12) {

      dut.clockDomain.waitSampling()

      checkOutput()
    }

    assert(
      expected.isEmpty,
      s"TEST 1 failed: ${expected.size} items remain"
    )

    println(
      "[PASS] TEST 1"
    )

    // ==========================================================
    // TEST 2
    // Backpressure
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 2: Backpressure")
    println("==========================================")

    dut.io.output.ready #=
      false

    println(
      "[BACKPRESSURE] output.ready = false"
    )

    /*
     * Keep valid asserted and hold one input payload.
     *
     * We intentionally wait until input.ready becomes false.
     */
    dut.io.input.payload #=
      100

    dut.io.input.valid #=
      true

    var acceptedCount = 0
    var blocked = false

    for (_ <- 0 until 30) {

      if (dut.io.input.ready.toBoolean) {

        /*
         * Transfer happens at this sampling edge.
         */
        dut.clockDomain.waitSampling()

        acceptedCount += 1

        println(
          s"[BACKPRESSURE] input accepted #$acceptedCount"
        )

      } else {

        blocked = true

        println(
          "[BACKPRESSURE] input.ready = false"
        )

        /*
         * Stop this test phase.
         */
        dut.io.input.valid #=
          false
      }

      if (!blocked) {
        dut.clockDomain.waitSampling()
      }
    }

    assert(
      blocked,
      "Expected backpressure to eventually reach input"
    )

    println(
      s"[PASS] Backpressure reached input after $acceptedCount accepted transactions"
    )

    // The transaction currently being held by the producer
    // was not accepted.
    dut.io.input.valid #=
      false

    // ==========================================================
    // TEST 3
    // Release backpressure
    // ==========================================================

    println()
    println("==========================================")
    println("TEST 3: Release backpressure")
    println("==========================================")

    dut.io.output.ready #=
      true

    var drainCycles = 0

    while (
      dut.io.output.valid.toBoolean ||
      drainCycles == 0
    ) {

      dut.clockDomain.waitSampling()

      if (
        dut.io.output.valid.toBoolean &&
        dut.io.output.ready.toBoolean
      ) {

        println(
          f"[DRAIN] output=0x${dut.io.output.payload.toInt}%02X"
        )
      }

      drainCycles += 1

      assert(
        drainCycles < 100,
        "Pipeline/FIFO did not drain within timeout"
      )

      /*
       * Once the FIFO/pipeline has drained,
       * wait one additional cycle and finish.
       */
      if (
        drainCycles > 5 &&
        !dut.io.output.valid.toBoolean
      ) {
        drainCycles = 100
      }
    }

    println(
      "[PASS] Backpressure released"
    )

    // ==========================================================
    // Final
    // ==========================================================

    println()
    println("==========================================")
    println("[PASS] ALL TESTS PASSED")
    println("==========================================")
  }
}
```

그런데 여기에도 **중요한 문제가 하나 있습니다.**

현재 Test 2에서:

```scala
dut.io.input.valid := true
dut.io.input.payload := 100
```

을 계속 유지하면, `ready`가 다시 올라올 때 같은 `100`을 다시 전송할 수 있습니다.

그리고 reference model을 이 테스트에서 별도로 추적하지 않고 있기 때문에 검증 목적이 약합니다.

따라서 **실제로 사용자가 실행할 최종 테스트는 아래처럼 만드는 것이 훨씬 좋습니다.**

---

# 11. 제가 추천하는 최종 검증 방식

이번 테스트에서는 아예 **Producer / Consumer simulation thread를 분리**하겠습니다.

SpinalHDL 공식 FIFO simulation도 push/pop을 별도의 simulation thread로 분리하고 Scala `Queue`를 reference model로 사용합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/zh_CN/master/SpinalHDL/Simulation/examples/single_clock_fifo.html?utm_source=chatgpt.com) SpinalSim은 이런 concurrent simulation thread를 지원합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Simulation/threadFull.html?utm_source=chatgpt.com)

그러면:

```text
Producer thread
       │
       ▼
      DUT
       │
       ▼
Consumer thread
```

가 됩니다.

그리고 Consumer가 의도적으로:

```text
ready = 0
```

을 만들도록 하겠습니다.

---

# 12. 제가 실제로 추천하는 최종 `AxiStreamPipelineSim`

이 버전을 직접 실행해 보시는 것을 권합니다.

```scala
import spinal.core._
import spinal.core.sim._

import scala.collection.mutable.Queue

object AxiStreamPipelineSim extends App {

  val dataWidth = 8
  val fifoDepth = 4
  val testCount = 30

  val compiled =
    SimConfig
      .withWave
      .compile(
        AxiStreamPipelineWithFifo(
          dataWidth = dataWidth,
          fifoDepth = fifoDepth
        )
      )

  compiled.doSimUntilVoid { dut =>

    println(
      "[Progress] Start AxiStreamPipeline simulation"
    )

    dut.clockDomain.forkStimulus(
      period = 10
    )

    dut.io.input.valid #=
      false

    dut.io.input.payload #=
      0

    dut.io.output.ready #=
      false

    dut.clockDomain.waitSampling(2)

    val expected =
      Queue[Int]()

    val producerDone =
      new java.util.concurrent.atomic.AtomicBoolean(false)

    // ==========================================================
    // Producer
    // ==========================================================

    fork {

      for (data <- 0 until testCount) {

        dut.io.input.payload #=
          data

        dut.io.input.valid #=
          true

        /*
         * Keep VALID and PAYLOAD stable until FIRE.
         */
        while (!dut.io.input.ready.toBoolean) {

          dut.clockDomain.waitSampling()
        }

        /*
         * Actual transfer edge.
         */
        dut.clockDomain.waitSampling()

        /*
         * Add the transformed value to the reference model.
         */
        expected.synchronized {
          expected.enqueue(
            (data + 1) * 2 + 10
          )
        }

        println(
          f"[PRODUCER] sent=$data%02d expected=0x${(data + 1) * 2 + 10}%02X"
        )

        dut.io.input.valid #=
          false

        /*
         * Allow some spacing between transactions.
         */
        dut.clockDomain.waitSampling()
      }

      producerDone.set(true)

      println(
        "[PRODUCER] done"
      )
    }

    // ==========================================================
    // Consumer
    // ==========================================================

    fork {

      var received =
        0

      var cycle =
        0

      while (received < testCount) {

        /*
         * Deliberately create backpressure.
         *
         * ready = 0 for several cycles,
         * then ready = 1.
         */
        val readyNow =
          if ((cycle / 5) % 2 == 0)
            false
          else
            true

        dut.io.output.ready #=
          readyNow

        dut.clockDomain.waitSampling()

        if (
          dut.io.output.valid.toBoolean &&
          dut.io.output.ready.toBoolean
        ) {

          assert(
            expected.synchronized {
              expected.nonEmpty
            },
            "DUT produced data but reference queue is empty"
          )

          val actual =
            dut.io.output.payload.toInt

          val wanted =
            expected.synchronized {
              expected.dequeue()
            }

          println(
            f"[CONSUMER] actual=0x$actual%02X expected=0x$wanted%02X ready=$readyNow"
          )

          assert(
            actual == wanted,
            f"DATA ERROR: actual=0x$actual%02X expected=0x$wanted%02X"
          )

          received += 1
        }

        cycle += 1

        assert(
          cycle < 500,
          "Simulation timeout: pipeline did not drain"
        )
      }

      /*
       * Leave the output blocked for one final observation.
       */
      dut.io.output.ready #=
        false

      dut.clockDomain.waitSampling()

      println(
        "[CONSUMER] all data received"
      )

      simSuccess()
    }

    /*
     * Safety timeout.
     */
    SimTimeout(10000)
  }
}
```

---

# 13. 그런데 이 코드에서 또 하나 주의할 점

여기서는 `expected`를 Producer와 Consumer thread가 공유합니다.

하지만 Producer가 reference queue에 데이터를 넣는 시점은:

```text
input.fire
```

가 발생한 직후입니다.

Pipeline에는 3개의 register stage가 있으므로:

```text
input.fire
   ↓
Stage 1
   ↓
Stage 2
   ↓
Stage 3
   ↓
FIFO
   ↓
output.fire
```

이라는 latency가 있습니다.

이것은 정상입니다.

Consumer는 queue의 첫 번째 데이터를 기다리고 있으므로 문제가 없습니다.

---

# 14. 예상되는 결과

정상적으로 동작하면 대략:

```text
[Progress] Start AxiStreamPipeline simulation

[PRODUCER] sent=00 expected=0x0C
[PRODUCER] sent=01 expected=0x0E
[PRODUCER] sent=02 expected=0x10
...

[CONSUMER] actual=0x0C expected=0x0C ready=true
[CONSUMER] actual=0x0E expected=0x0E ready=true
[CONSUMER] actual=0x10 expected=0x10 ready=true
...
```

처럼 됩니다.

입력 `0`의 계산은:

```text
0
 ↓
+1
 ↓
1
 ↓
×2
 ↓
2
 ↓
+10
 ↓
12 = 0x0C
```

입니다.

입력 `1`:

```text
1
 ↓
2
 ↓
4
 ↓
14 = 0x0E
```

입니다.

---

# 15. Backpressure는 실제로 어떻게 보이는가?

Consumer가:

```scala
ready = false
```

인 구간에서는:

```text
output.valid = 1
output.ready = 0
```

이 될 수 있습니다.

그러면:

```text
output.fire = 0
```

입니다.

FIFO가 차기 시작하고:

```text
FIFO full
```

이 되면 FIFO의 push side:

```text
push.ready = 0
```

가 됩니다.

그러면 Stage 3가 멈추고:

```text
Stage 3 ready = 0
```

그 결과 Stage 2:

```text
Stage 2 ready = 0
```

Stage 1:

```text
Stage 1 ready = 0
```

마지막으로 input:

```text
input.ready = 0
```

까지 전파됩니다.

즉 waveform에서는 반드시 다음 현상을 찾아보시면 좋습니다.

```text
output.ready
     ↓
FIFO
     ↓
stage3
     ↓
stage2
     ↓
stage1
     ↓
input.ready
```

**backpressure가 역방향으로 전달되는 모습**을 GTKWave에서 직접 확인하는 것이 이번 실습의 핵심입니다.

---

# 16. 매우 중요한 점: FIFO depth가 4라고 해서 전체 buffering이 4는 아니다

우리 구조는:

```text
Stage1 register
Stage2 register
Stage3 register
FIFO depth 4
```

입니다.

따라서 pipeline register에도 데이터가 들어갈 수 있습니다.

개념적으로 최대 buffering capacity는 대략:

```text
3 pipeline stages + FIFO 4
```

수준으로 생각할 수 있습니다.

즉 FIFO가 `full`이 되는 순간에도 pipeline 내부에 이미 데이터가 존재할 수 있습니다.

이것이 **elastic pipeline**의 중요한 특징입니다.

---

# 17. 이번 예제에서 확인할 것

직접 실행하신 후 다음 네 가지를 확인해 주세요.

### ① 정상 데이터 변환

```text
input
 ↓
+1
 ↓
×2
 ↓
+10
```

즉:

```text
output = (input + 1) * 2 + 10
```

이 정확한지.

---

### ② 순서 보존

예:

```text
input:
0 1 2 3 4 5 ...

output:
12 14 16 18 20 22 ...
```

가 되는지.

---

### ③ Backpressure

Consumer가:

```text
ready = 0
```

인 동안에도 데이터가 사라지지 않는지.

---

### ④ 다시 READY가 되었을 때

```text
ready = 1
```

로 바꾸면:

```text
FIFO
 ↓
Stage
 ↓
Consumer
```

순서대로 데이터가 다시 나오면서 **하나도 빠지거나 중복되지 않는지**.

---

# 18. 이번 실습에서 꼭 볼 GTKWave 신호

`withWave`로 실행하면 다음 신호를 찾아보세요.

```text
io_input_valid
io_input_ready
io_input_payload

stage1_io_input_valid
stage1_io_input_ready
stage1_io_output_valid
stage1_io_output_ready

stage2_io_input_valid
stage2_io_input_ready
stage2_io_output_valid
stage2_io_output_ready

stage3_io_input_valid
stage3_io_input_ready
stage3_io_output_valid
stage3_io_output_ready

fifo_io_push_valid
fifo_io_push_ready
fifo_io_pop_valid
fifo_io_pop_ready

io_output_valid
io_output_ready
io_output_payload
```

SpinalHDL 생성 이름은 약간 다르게 표시될 수 있습니다.

특히 다음 관계를 직접 확인하면 좋습니다.

```text
output.ready = 0
       ↓
FIFO pop.ready = 0
       ↓
FIFO eventually full
       ↓
FIFO push.ready = 0
       ↓
stage3 input.ready = 0
       ↓
stage2 ...
       ↓
stage1 ...
       ↓
input.ready = 0
```

이것이 이번 단계에서 배운 **Stream + FIFO + Pipeline + Backpressure의 실제 hardware 동작**입니다.

---

## 한 가지 정리

이번 예제에서는 실제 AXI4-Stream의 모든 옵션(`TLAST`, `TKEEP`, `TSTRB`, `TID`, `TDEST` 등)을 넣지 않았습니다.

그것은 의도적입니다.

현재 단계에서는 먼저:

```text
        AXI-Stream 핵심

          TVALID
             │
             ▼
TDATA ──► Transfer ◄── TREADY

       Transfer =
       VALID && READY
```

를 SpinalHDL의:

```scala
Stream(UInt(...))
```

으로 확실히 이해하는 것이 중요합니다.

SpinalHDL 공식 문서에서도 `StreamFifo`를 `push`/`pop` Stream으로 사용하는 형태를 제공하고 있고, FIFO의 다양한 latency/read 옵션도 지원합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalHDL/dev/spinal/lib/StreamFifo.html?utm_source=chatgpt.com)

**우선 위 두 파일을 그대로 실행해 보세요.** 특히 이번에는 이전 `ValidReadyFifoSim` 때처럼 simulation timing 문제를 제가 미리 단정하지 않고, **실제 실행 결과와 waveform을 기준으로 다음 수정 여부를 판단하는 방식**으로 진행하는 것이 좋습니다.