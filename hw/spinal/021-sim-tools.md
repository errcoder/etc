일단, spinal.lib.sim.{StreamMonitor, StreamDriver, StreamReadyRandomizer, ScoreboardInOrder} 
각각의 기능과 용법을 학습한다. 다음 Example 코드를 사용하여 설명하시오:

```scala
object Example extends App {
  val dut = SimConfig.withWave.compile(StreamFifo(Bits(8 bits), 2))

  dut.doSim("simple test") { dut =>
    SimTimeout(10000)

    val scoreboard = ScoreboardInOrder[Int]()

    dut.io.flush #= false

    // drive random data and add pushed data to scoreboard
    StreamDriver(dut.io.push, dut.clockDomain) { payload =>
      payload.randomize()
      true
    }
    StreamMonitor(dut.io.push, dut.clockDomain) { payload =>
      scoreboard.pushRef(payload.toInt)
    }

    // randomize ready on the output and add popped data to scoreboard
    StreamReadyRandomizer(dut.io.pop, dut.clockDomain)
    StreamMonitor(dut.io.pop, dut.clockDomain) { payload =>
      scoreboard.pushDut(payload.toInt)
    }

    dut.clockDomain.forkStimulus(10)

    dut.clockDomain.waitActiveEdgeWhere(scoreboard.matches == 100)
  }
}
```

좋습니다. 이번에는 **코드를 수정하지 않고**, SpinalSim에서 제공하는 네 가지 helper를 먼저 정확하게 학습하겠습니다.

대상은 다음 4개입니다.

1. `StreamDriver`
2. `StreamMonitor`
3. `StreamReadyRandomizer`
4. `ScoreboardInOrder`

그리고 마지막에는 질문에 주신 `Example` 코드가 **전체적으로 어떤 순서로 동작하는지**까지 연결하겠습니다.

---

# 1. 먼저 전체 구조를 이해하기

질문의 Example은 `StreamFifo`를 테스트하고 있습니다.

```text
                    ┌─────────────────┐
                    │    StreamFifo   │
                    │                 │
  push Stream ─────►│                 │─────► pop Stream
                    └─────────────────┘
       ▲                                      │
       │                                      │
 StreamDriver                         StreamReadyRandomizer
       │                                      │
       │                                      ▼
 StreamMonitor                         StreamMonitor
       │                                      │
       ▼                                      ▼
 pushRef()                              pushDut()
       │                                      │
       └──────────────┬───────────────────────┘
                      ▼
              ScoreboardInOrder
```

핵심 아이디어는 아주 간단합니다.

> **Driver는 DUT에 데이터를 넣고, Monitor는 실제 handshake를 관찰하며, Scoreboard는 "넣은 순서"와 "나온 순서"가 같은지를 검사한다.**

---

# 2. `StreamDriver`

## 2-1. 역할

`StreamDriver`는 이름 그대로 **Stream에 데이터를 공급하는 테스트벤치 driver**입니다.

즉 DUT의

```scala
Stream(...)
```

입력에 대해 테스트 코드가 직접 `valid`, `payload` 등을 관리하지 않아도 되도록 도와줍니다.

질문의 코드:

```scala
StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  payload.randomize()
  true
}
```

여기서:

```scala
dut.io.push
```

는 DUT 입장에서 **입력 Stream**입니다.

Driver는 이 Stream의 producer 역할을 합니다.

---

# 3. Stream에서 Driver가 실제로 하는 일

Stream의 기본 handshake는:

```text
fire = valid && ready
```

입니다.

따라서 개념적으로 Driver는 다음과 같은 일을 합니다.

```text
Driver
  │
  ├── payload 설정
  ├── valid = true
  │
  │       DUT
  │        │
  │        └── ready
  │
  └── valid && ready
          │
          ▼
        전송
```

중요한 것은 **`ready`가 낮다고 해서 payload를 바꾸면 안 된다는 것**입니다.

Stream protocol에서는 producer가 `valid=1`을 올린 뒤 consumer가 `ready=1`이 될 때까지 payload를 안정적으로 유지해야 합니다.

`StreamDriver`가 이 과정을 처리해 줍니다.

---

# 4. `StreamDriver` callback의 의미

다음 코드를 봅시다.

```scala
StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  payload.randomize()
  true
}
```

여기서 callback:

```scala
{ payload =>
    payload.randomize()
    true
}
```

는 **새로운 transaction을 준비하는 함수**입니다.

`payload`는:

```scala
Bits(8 bits)
```

입니다.

따라서:

```scala
payload.randomize()
```

하면 8-bit random data가 만들어집니다.

예를 들어:

```text
0x3A
0x81
0x05
0xF7
...
```

같은 값들이 생성될 수 있습니다.

그리고 마지막의:

```scala
true
```

는 매우 중요합니다.

### `true`

```text
"이 transaction을 전송해라"
```

라는 의미입니다.

반대로 callback에서:

```scala
false
```

를 반환하면 해당 transaction을 전송하지 않습니다.

---

# 5. 간단한 `StreamDriver` 예제

예를 들어 random data가 아니라 특정 데이터를 보내고 싶다고 해봅시다.

개념적으로:

```scala
StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  payload #= 0x55
  true
}
```

그러면 driver가 계속 `0x55`를 공급합니다.

다른 형태로 transaction sequence를 만들 수도 있습니다.

```scala
val data = Array(0x11, 0x22, 0x33)

var index = 0

StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  if (index < data.length) {
    payload #= data(index)
    index += 1
    true
  } else {
    false
  }
}
```

개념적으로:

```text
0x11 → 0x22 → 0x33
```

을 순서대로 Stream에 공급합니다.

---

# 6. `StreamMonitor`

이제 두 번째입니다.

```scala
StreamMonitor(dut.io.push, dut.clockDomain) { payload =>
  scoreboard.pushRef(payload.toInt)
}
```

`StreamDriver`가 **DUT에 데이터를 넣는 역할**이었다면,

`StreamMonitor`는 **Stream에서 실제로 발생한 transaction을 관찰하는 역할**입니다.

중요한 차이입니다.

---

# 7. Monitor는 단순히 `valid`를 보는 것이 아니다

Stream에서 transaction이 발생하는 조건은:

```scala
valid && ready
```

입니다.

즉:

```text
valid = 1
ready = 0
```

이면 **전송되지 않았습니다.**

Monitor가 이것을 transaction으로 취급하면 안 됩니다.

반면:

```text
valid = 1
ready = 1
```

이면:

```text
fire = 1
```

이고 실제 transaction입니다.

`StreamMonitor`는 이런 **실제 Stream transaction**을 관찰하는 데 사용합니다.

---

# 8. 왜 Driver와 Monitor를 같이 사용하는가?

질문의 코드가 아주 좋은 예입니다.

```scala
StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  payload.randomize()
  true
}

StreamMonitor(dut.io.push, dut.clockDomain) { payload =>
  scoreboard.pushRef(payload.toInt)
}
```

처음 보면 이런 의문이 생깁니다.

> "Driver가 random data를 만들었는데 왜 다시 Monitor로 읽어서 scoreboard에 넣지?"

이것이 바로 테스트벤치에서 중요한 부분입니다.

Driver는:

```text
무엇을 보내려고 하는가?
```

를 담당합니다.

Monitor는:

```text
실제로 무엇이 Stream을 통과했는가?
```

를 관찰합니다.

따라서 Monitor를 사용하면 테스트벤치가 실제 DUT 인터페이스에서 발생한 transaction을 기준으로 scoreboard에 등록할 수 있습니다.

---

# 9. `ScoreboardInOrder`

이제 핵심입니다.

```scala
val scoreboard = ScoreboardInOrder[Int]()
```

이것은 **transaction의 순서를 기억하면서 DUT 결과를 비교하는 도구**입니다.

쉽게 말하면:

```text
Reference Queue
----------------
A
B
C
D
```

와

```text
DUT Queue
----------------
A
B
C
D
```

를 비교하는 역할입니다.

---

# 10. `pushRef()`

```scala
scoreboard.pushRef(value)
```

는:

> "앞으로 DUT가 이 값을 이 순서대로 출력해야 한다."

라고 scoreboard에 등록합니다.

예를 들어:

```scala
scoreboard.pushRef(0x11)
scoreboard.pushRef(0x22)
scoreboard.pushRef(0x33)
```

이면 내부적으로 개념적으로:

```text
Reference Queue

┌──────┐
│ 0x11 │ ← 다음에 나와야 함
├──────┤
│ 0x22 │
├──────┤
│ 0x33 │
└──────┘
```

이 됩니다.

---

# 11. `pushDut()`

반대로:

```scala
scoreboard.pushDut(value)
```

는:

> "DUT에서 실제로 이 값이 나왔다."

라고 알려주는 것입니다.

예를 들어 reference가:

```text
0x11
0x22
0x33
```

인데 DUT에서:

```text
0x11
```

이 나오면:

```text
REF = 0x11
DUT = 0x11
```

이므로 match입니다.

그러면 다음 비교는:

```text
REF = 0x22
```

가 됩니다.

---

# 12. 순서가 틀리면?

Reference:

```text
0x11
0x22
0x33
```

DUT:

```text
0x11
0x33
```

이라면 두 번째 transaction에서:

```text
REF : 0x22
DUT : 0x33
```

이므로 즉시 mismatch입니다.

질문에서 이전에 발생했던:

```text
REF : 65
DUT : 66
```

같은 메시지가 바로 이런 상황입니다.

10진수로 표시되었기 때문에:

```text
65 = 0x41
66 = 0x42
```

입니다.

즉 scoreboard 입장에서는:

```text
"0x41이 나와야 하는데 0x42가 나왔다."
```

라고 판단한 것입니다.

---

# 13. `ScoreboardInOrder`의 이름이 중요한 이유

이름 그대로:

```text
Scoreboard
    +
InOrder
```

입니다.

즉 **순서가 중요한 scoreboard**입니다.

예를 들어 reference가:

```text
A
B
C
```

인데 DUT가:

```text
B
A
C
```

를 출력하면 값 자체는 모두 존재하지만 실패합니다.

왜냐하면:

```text
A → B → C
```

순서를 보장해야 하기 때문입니다.

FIFO 테스트에는 특히 적합합니다.

---

# 14. `StreamReadyRandomizer`

이제 네 번째가 아니라 세 번째 도구입니다.

질문의 코드:

```scala
StreamReadyRandomizer(dut.io.pop, dut.clockDomain)
```

이것은 **consumer 쪽의 `ready`를 random하게 변화시키는 테스트 도구**입니다.

Stream의 output을 보면:

```text
FIFO
 │
 ▼
pop Stream
 │
 ├── valid
 ├── payload
 └── ready
```

여기서 일반적인 테스트라면:

```scala
dut.io.pop.ready #= true
```

라고 할 수 있습니다.

그러면 FIFO에서 데이터가 나오는 즉시 계속 받아갑니다.

하지만 실제 시스템에서는 consumer가 항상 준비되어 있지 않을 수 있습니다.

그래서:

```scala
StreamReadyRandomizer(dut.io.pop, dut.clockDomain)
```

를 사용하면:

```text
ready = 1
ready = 0
ready = 1
ready = 1
ready = 0
ready = 0
ready = 1
...
```

처럼 random backpressure를 만들어냅니다.

---

# 15. 이것이 FIFO 테스트에서 중요한 이유

FIFO가 제대로 만들어졌는지 확인하려면 단순히:

```text
push A
push B
push C
pop A
pop B
pop C
```

만 테스트해서는 부족합니다.

실제 Stream 환경에서는:

```text
FIFO valid = 1
Consumer ready = 0
```

상황이 발생합니다.

이때 FIFO는:

```text
payload를 잃어버리면 안 됨
```

그리고:

```text
valid를 유지해야 함
```

입니다.

따라서:

```scala
StreamReadyRandomizer(dut.io.pop, dut.clockDomain)
```

는 DUT에 **random backpressure를 가해서 Stream protocol이 제대로 구현되어 있는지 검사**합니다.

---

# 16. `StreamMonitor(dut.io.pop ...)`

이제 다음 코드입니다.

```scala
StreamMonitor(dut.io.pop, dut.clockDomain) { payload =>
  scoreboard.pushDut(payload.toInt)
}
```

이 Monitor는 FIFO의 output Stream을 감시합니다.

그리고 실제 transaction이 발생하면:

```scala
scoreboard.pushDut(...)
```

를 호출합니다.

즉:

```text
FIFO output
     │
     ▼
StreamMonitor
     │
     ▼
실제 출력 데이터
     │
     ▼
pushDut()
     │
     ▼
Scoreboard
```

입니다.

---

# 17. 전체 Example을 한 줄씩 분석

이제 질문에서 준 코드를 그대로 보겠습니다.

```scala
object Example extends App {
```

Scala application의 시작점입니다.

---

### DUT 생성

```scala
val dut = SimConfig.withWave.compile(StreamFifo(Bits(8 bits), 2))
```

8-bit 데이터와 depth 2인 `StreamFifo`를 simulation용으로 compile합니다.

```text
Bits(8 bits)
depth = 2
```

입니다.

그리고:

```scala
withWave
```

이므로 waveform도 생성할 수 있습니다.

---

### Simulation 시작

```scala
dut.doSim("simple test") { dut =>
```

simulation body입니다.

---

### Timeout

```scala
SimTimeout(10000)
```

simulation이 너무 오래 진행되는 것을 방지합니다.

```text
최대 simulation time = 10000
```

이라는 안전장치라고 이해하면 됩니다.

---

# 18. Scoreboard 생성

```scala
val scoreboard = ScoreboardInOrder[Int]()
```

정수 데이터를 비교하는 scoreboard를 만듭니다.

개념적으로:

```text
Reference                 DUT
─────────                 ───
expected data             actual data
```

를 비교합니다.

---

# 19. Flush 초기화

```scala
dut.io.flush #= false
```

FIFO의 flush를 사용하지 않는다는 의미입니다.

---

# 20. Push Driver

```scala
StreamDriver(dut.io.push, dut.clockDomain) { payload =>
  payload.randomize()
  true
}
```

이제 FIFO에 random data를 계속 넣습니다.

흐름:

```text
Random Data
    │
    ▼
StreamDriver
    │
    ▼
push Stream
    │
    ▼
FIFO
```

---

# 21. Push Monitor

```scala
StreamMonitor(dut.io.push, dut.clockDomain) { payload =>
  scoreboard.pushRef(payload.toInt)
}
```

실제 push Stream transaction을 관찰합니다.

그리고:

```scala
scoreboard.pushRef(...)
```

로 expected queue에 넣습니다.

예를 들어 실제 push가:

```text
0xA1
0x37
0x92
0x10
```

이었다면:

```text
Scoreboard Reference

A1
37
92
10
```

이 됩니다.

---

# 22. Output Ready Randomizer

```scala
StreamReadyRandomizer(dut.io.pop, dut.clockDomain)
```

FIFO의 output consumer가 random하게 ready를 발생시키도록 합니다.

따라서 FIFO 입장에서는:

```text
"지금 데이터를 받을 수 있어!"

```

였다가,

```text
"지금은 못 받아!"
```

가 반복됩니다.

이것이 바로 **backpressure 테스트**입니다.

---

# 23. Pop Monitor

```scala
StreamMonitor(dut.io.pop, dut.clockDomain) { payload =>
  scoreboard.pushDut(payload.toInt)
}
```

FIFO에서 실제로 나온 데이터를 감시합니다.

예를 들어:

```text
FIFO → 0xA1
```

가 실제 handshake되면:

```scala
scoreboard.pushDut(0xA1)
```

가 실행됩니다.

Scoreboard는:

```text
Expected: A1
Actual:   A1
```

을 비교합니다.

---

# 24. Clock 생성

```scala
dut.clockDomain.forkStimulus(10)
```

simulation clock을 생성합니다.

여기서 `10`은 clock half-period 설정으로 이해하면 됩니다.

즉 waveform에서는 clock이 계속 생성됩니다.

---

# 25. 마지막 줄이 핵심

```scala
dut.clockDomain.waitActiveEdgeWhere(scoreboard.matches == 100)
```

이것은:

> scoreboard의 match 개수가 100이 될 때까지 simulation을 진행한다.

는 의미입니다.

즉:

```text
match = 0
   ↓
match = 1
   ↓
match = 2
   ↓
...
   ↓
match = 100
   ↓
simulation 종료
```

입니다.

---

# 26. 전체 동작을 한 장으로 정리

질문의 Example은 사실 다음 구조입니다.

```text
                    TESTBENCH
──────────────────────────────────────────────

                 Random Data
                     │
                     ▼
              ┌──────────────┐
              │ StreamDriver │
              └──────┬───────┘
                     │
                     │ push Stream
                     ▼
             ┌─────────────────┐
             │                 │
             │   StreamFifo    │
             │                 │
             │   depth = 2     │
             │                 │
             └────────┬────────┘
                      │
                      │ pop Stream
                      ▼
              ┌──────────────┐
              │   Monitor    │
              └──────┬───────┘
                     │
                     ▼
                 pushDut()
                     │
                     ▼
              ┌──────────────┐
              │  Scoreboard  │
              │              │
              │   InOrder    │
              └──────────────┘


push Stream
    │
    └──► StreamMonitor
             │
             └──► pushRef()
```

그리고 output 쪽에는:

```text
StreamReadyRandomizer
          │
          ▼
        ready
          │
          ▼
       StreamFifo
```

가 추가됩니다.

---

# 27. 네 가지 도구의 역할 비교

| 도구 | 역할 | 방향 |
|---|---|---|
| `StreamDriver` | Stream에 transaction을 공급 | Testbench → DUT |
| `StreamMonitor` | 실제 Stream transaction 관찰 | Stream → Testbench |
| `StreamReadyRandomizer` | `ready`를 random하게 생성 | Testbench → DUT |
| `ScoreboardInOrder` | 예상값과 실제값을 순서대로 비교 | Testbench 내부 |

가장 중요한 구분은 이것입니다.

### `StreamDriver`

**보내는 사람**

```text
"이 데이터를 보내겠습니다."
```

### `StreamMonitor`

**감시하는 사람**

```text
"실제로 이 데이터가 지나갔습니다."
```

### `StreamReadyRandomizer`

**받을 수 있는지 랜덤하게 결정하는 사람**

```text
"이번에는 받을 수 있습니다."
"이번에는 못 받습니다."
```

### `ScoreboardInOrder`

**채점하는 사람**

```text
"예상값과 실제값이 같은가?"
```

---

# 28. 특히 중요한 `Driver` vs `Monitor`

앞서 우리가 UART simulation에서 겪었던 문제 때문에 이 차이는 아주 중요합니다.

다음은 개념적으로 다릅니다.

```scala
// 내가 보내려고 하는 데이터
scoreboard.pushRef(value)
```

와

```scala
// 실제 Stream handshake를 통과한 데이터
StreamMonitor(...) {
  payload =>
    scoreboard.pushRef(payload.toInt)
}
```

두 번째 방식은 **실제 Stream transaction을 관찰한 결과를 기준으로 scoreboard를 구성**할 수 있습니다.

따라서 앞으로 Stream 기반 simulation을 작성할 때는:

```text
Driver
  ↓
DUT
  ↓
Monitor
  ↓
Scoreboard
```

라는 구조를 기본 패턴으로 기억하는 것이 좋습니다.

---

# 29. 이번 학습에서 반드시 기억할 4가지

```text
① StreamDriver
   → 데이터를 넣는다.

② StreamMonitor
   → 실제 fire transaction을 관찰한다.

③ StreamReadyRandomizer
   → ready를 랜덤하게 만들어 backpressure를 시험한다.

④ ScoreboardInOrder
   → expected와 actual을 순서대로 비교한다.
```

그리고 Stream의 가장 중요한 공식:

```text
fire = valid && ready
```

입니다.

따라서 **`valid`만 올라갔다고 transaction이 발생한 것이 아닙니다.**

이 개념이 앞으로 우리가 작성할 `UartEndpoint`, `UartDuplexSystem`, FIFO simulation을 정확하게 이해하는 데 매우 중요합니다.
