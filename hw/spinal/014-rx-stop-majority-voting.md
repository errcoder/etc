네. 정확히 이해하셨습니다.

현재 우리가 만든 `UartRx`는 **START bit를 bitTimer가 bit의 중간 지점에 도달했을 때 한 번 샘플링**해서 START bit가 정말 LOW인지 확인하는 구조입니다.

그리고 **majority voting(다수결 샘플링)** 으로 바꾸는 것은 충분히 가능합니다. 오히려 UART RX를 좀 더 견고하게 만드는 좋은 다음 단계입니다.

# 1. 현재 `UartRx`의 START 검출

현재 구조를 단순화하면:

```scala
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
    bitTimer := bitTimer + 1
  }
}
```

예를 들어

```text
cyclesPerBit = 10
```

이면:

```text
START bit
        LOW
         │
         ▼
clock:  0 1 2 3 4 5 6 7 8 9
                    ↑
                  sample
```

즉 **START bit의 중앙 부근에서 한 번**:

```text
rx == 0 ?
```

를 검사합니다.

이 방식의 장점은 매우 단순하다는 것입니다.

---

# 2. Majority Voting을 하면?

예를 들어 한 bit 동안 3번 샘플링한다고 생각해 봅시다.

```text
START bit

clock:    0 1 2 3 4 5 6 7 8 9
                    │ │ │
                    │ │ │
                   S1 S2 S3
```

세 샘플이:

```text
S1 = 0
S2 = 0
S3 = 1
```

이면

```text
0, 0, 1
```

에서 **0이 2개**이므로 최종 결과는:

```text
START = 0
```

입니다.

반대로:

```text
S1 = 1
S2 = 0
S3 = 1
```

이면:

```text
1, 0, 1
```

→ **1이 2개**

따라서:

```text
START = 1
```

로 판단하고 START bit가 잘못되었다고 볼 수 있습니다.

---

# 3. 왜 더 신뢰성이 높아지는가?

한 번만 샘플링하면 순간적인 glitch에 영향을 받을 수 있습니다.

예를 들어 실제 START bit가 LOW인데:

```text
실제 RX:

───────────────┐       ┌────────
               │ LOW   │
               └───────┘
```

중앙 샘플 순간에 잡음이 발생해서:

```text
                 noise
                   ↑
───────────────┐  ┌┐  ┌────────
               └──┘└──┘
```

샘플 하나가 잘못될 수 있습니다.

Single sampling:

```text
sample = 1
```

→ START 실패.

하지만 majority voting:

```text
sample 1 = 0
sample 2 = 1
sample 3 = 0
```

이면:

```text
0 + 1 + 0
```

→ 0이 2개

→ **START = 0**

으로 올바르게 판단할 가능성이 높아집니다.

---

# 4. 그런데 중요한 점

여기서 **"3번 샘플링하면 무조건 더 좋아진다"**라고 생각하면 안 됩니다.

UART에서는 **언제 샘플링하느냐**가 굉장히 중요합니다.

예를 들어:

```text
             START
        ┌───────────────┐
RX ─────┘               └────────
        0               10

        ↑       ↑       ↑
        S1      S2      S3
```

3개의 샘플이 **bit 중앙 주변에 대칭적으로 위치**해야 합니다.

예를 들어 `cyclesPerBit = 10`이라면:

```text
S1 = 3
S2 = 5
S3 = 7
```

처럼 잡을 수 있습니다.

즉:

```text
          START bit
     ┌─────────────────┐
     │                 │
clk  0 1 2 3 4 5 6 7 8 9
           ↑   ↑   ↑
           S1  S2  S3
```

이런 구조가 됩니다.

---

# 5. START bit만 majority voting할 수도 있다

그리고 여기서 아주 중요한 설계 선택이 있습니다.

**모든 DATA bit에 majority voting을 적용할 필요는 없습니다.**

우리가 현재 배우는 UART RX에서는 우선:

```text
IDLE
  ↓
START 검출
  ↓
START 중앙 확인
  ↓
DATA bit sampling
  ↓
STOP
```

이 구조입니다.

따라서 처음에는:

```text
START bit
    ↓
3-point majority voting
```

만 적용해도 상당히 좋은 학습 단계입니다.

예를 들어:

```text
START

sample #1     sample #2     sample #3
    │             │             │
    ▼             ▼             ▼
   clk 3         clk 5         clk 7

        ┌─────────────────┐
RX ─────┘                 └──────
             LOW
```

그리고:

```scala
startSample0
startSample1
startSample2
```

를 저장한 후 majority를 계산합니다.

논리적으로는:

```text
majority =
    (s0 & s1) |
    (s1 & s2) |
    (s0 & s2)
```

입니다.

예를 들어:

| s0 | s1 | s2 | 결과 |
|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 |
| 1 | 0 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 |
| 1 | 1 | 1 | 1 |

---

# 6. 더 발전시키면 DATA bit도 majority voting 가능

사실 UART RX에서 더 흥미로운 것은 이것입니다.

**START뿐 아니라 DATA bit 자체도 3-point majority voting 할 수 있습니다.**

예를 들어:

```text
DATA bit
           ┌───────────────────┐
RX ────────┘                   └──────

clock:    0 1 2 3 4 5 6 7 8 9
                  ↑ ↑ ↑
                  │ │ │
                 S1 S2 S3
```

그리고:

```text
S1 = rx[3]
S2 = rx[5]
S3 = rx[7]
```

을 얻어서:

```text
bitValue =
    (S1 & S2) |
    (S2 & S3) |
    (S1 & S3)
```

으로 결정합니다.

그러면 한 번의 순간적인 glitch:

```text
0 0 1
```

또는

```text
1 0 0
```

등을 어느 정도 무시할 수 있습니다.

---

# 7. 하지만 이것보다 더 중요한 UART RX 구조가 있다

실제 UART receiver를 좀 더 제대로 설계할 때는 보통:

```text
UART RX
   │
   ▼
Oversampling
   │
   ├── 8x
   ├── 16x
   └── 32x
         │
         ▼
   Majority Voting
         │
         ▼
      Data bit
```

같은 구조를 사용합니다.

대표적으로 **16× oversampling**을 생각할 수 있습니다.

```text
1 UART bit
┌──────────────────────────────────────────────┐
│                                              │
│ 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15     │
│                 ↑  ↑  ↑                     │
│                 │  │  │                     │
│                S1 S2 S3                    │
└──────────────────────────────────────────────┘
```

예를 들어:

```text
sample 7
sample 8
sample 9
```

을 majority voting하는 식입니다.

이렇게 하면 clock mismatch, noise, edge jitter 등에 좀 더 강해집니다.

---

# 8. 우리의 현재 학습 단계에서는 이렇게 가는 것을 추천

현재 우리가 만든 `UartRx`는:

```text
                 현재
                   │
                   ▼
        ┌────────────────────┐
        │ START 중앙 1회 샘플 │
        └────────────────────┘
                   │
                   ▼
             DATA 1회 샘플
```

입니다.

여기서 바로 16× oversampling으로 가기보다는 단계적으로:

### Step 1 — 현재

```text
START → 중앙 1회
DATA  → 중앙 1회
```

### Step 2 — START majority

```text
START → 3-point majority
DATA  → 중앙 1회
```

### Step 3 — DATA majority

```text
START → 3-point majority
DATA  → 3-point majority
```

### Step 4 — Oversampling

```text
RX
 ↓
8× 또는 16× sampling
 ↓
3-point majority
 ↓
DATA
```

이렇게 발전시키는 것이 **SpinalHDL 학습 측면에서도 아주 좋습니다.**

---

# 9. 특히 현재 `UartRx`에 적용하기 좋은 이유

현재 코드가 이미:

```scala
val cyclesPerBit =
    clockFrequencyHz / baudRate
```

를 가지고 있고,

```scala
val bitTimer =
    Reg(UInt(timerWidth bits)) init(0)
```

로 UART bit 내부의 시간을 세고 있습니다.

따라서 majority voting을 넣는다고 해서 UART RX 전체를 다시 설계할 필요는 없습니다.

현재:

```text
START
   │
   └── halfBitCount까지 기다림
          │
          └── rx 한 번 검사
```

를:

```text
START
   │
   ├── sample 1
   │
   ├── sample 2
   │
   ├── sample 3
   │
   └── majority
          │
          ├── LOW → DATA
          └── HIGH → IDLE
```

로 확장하면 됩니다.

---

## 결론

**가능하고, 실제로 좋은 개선 방향입니다.**

현재 `UartRx`의 START 검사는:

> **START bit의 중간 지점에서 1회 sampling**

입니다.

이를:

> **START bit 중앙 주변의 3개 지점을 sampling → majority voting**

으로 바꾸면 순간적인 noise/glitch에 대한 내성이 좋아집니다.

그리고 장기적으로는:

```text
현재
1x sampling
    ↓
START 3-point majority
    ↓
DATA 3-point majority
    ↓
8x/16x oversampling
    ↓
실용적인 UART RX
```

순서로 발전시키는 것이 좋습니다.

**다음 단계로는 `UartRx`를 바로 16×로 바꾸기보다, 현재 `cyclesPerBit = 10` 구조를 유지하면서 `START 3-point majority voting` 버전을 먼저 만들어 시뮬레이션하는 것이 가장 좋습니다.** 
이렇게 하면 기존 `UartRx`와 waveform을 직접 비교하면서 majority voting이 실제로 어떻게 동작하는지도 확인할 수 있습니다.

---

---

좋습니다. **기존 `UartRx`의 구조와 DATA/STOP 동작은 그대로 유지**하고, **START bit 검증 부분만 3-point majority voting으로 변경**하겠습니다.

그리고 그 다음 학습 순서는 요청하신 대로:

1. `Stream` vs `Flow`
2. `StreamFifo` vs SpinalHDL 라이브러리 FIFO
3. `AXI-Stream`
4. `Stream` 기반 pipeline

순서로 진행하겠습니다.

# 1. 수정된 전체 `UartRx`

대상 버전은 계속 **SpinalHDL Release 34 / v1.15.0**입니다.

기존 구조에서 중요한 점은 그대로입니다.

- `IDLE`
- `START`
- `DATA`
- `STOP`
- `bitTimer`
- `bitCounter`
- `shiftReg`
- DATA bit는 기존처럼 bit 중앙에서 1회 sampling
- STOP도 기존 방식 유지
- `io.valid`도 기존처럼 1-clock pulse

변경되는 부분은 **START 검증뿐**입니다.

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

  // ------------------------------------------------------------
  // START majority voting registers
  // ------------------------------------------------------------

  val startSample0 =
    Reg(Bool()) init(False)

  val startSample1 =
    Reg(Bool()) init(False)

  val startSample2 =
    Reg(Bool()) init(False)

  // ------------------------------------------------------------
  // Outputs
  // ------------------------------------------------------------

  io.data  := shiftReg
  io.valid := False

  // ------------------------------------------------------------
  // State machine
  // ------------------------------------------------------------

  switch(state) {

    // ==========================================================
    // IDLE
    // ==========================================================

    is(State.IDLE) {

      bitTimer := 0
      bitCounter := 0

      when(!io.rx) {

        // START bit detected.
        //
        // Begin sampling the START bit.
        bitTimer := 0

        startSample0 := False
        startSample1 := False
        startSample2 := False

        state := State.START
      }
    }

    // ==========================================================
    // START
    // ==========================================================

    is(State.START) {

      /*
       * 3-point majority voting
       *
       * For cyclesPerBit = 10:
       *
       *       START bit
       *   ┌─────────────────┐
       *   │                 │
       *   └─────────────────┘
       *
       * clk
       *     0 1 2 3 4 5 6 7 8 9
       *           ↑   ↑   ↑
       *          S0  S1  S2
       *
       * The three samples are taken around the center
       * of the START bit.
       */

      when(bitTimer === (halfBitCount - 2)) {

        startSample0 := io.rx

        bitTimer := bitTimer + 1

      } elsewhen(bitTimer === (halfBitCount - 1)) {

        startSample1 := io.rx

        bitTimer := bitTimer + 1

      } elsewhen(bitTimer === halfBitCount) {

        startSample2 := io.rx

        bitTimer := 0

        /*
         * Majority voting
         *
         * At least two of the three samples must be LOW.
         *
         * LOW = 0
         */
        when(
          (!startSample0 && !startSample1) ||
          (!startSample1 && !startSample2) ||
          (!startSample0 && !startSample2)
        ) {

          bitCounter := 0
          state := State.DATA

        } otherwise {

          // Not a valid START bit.
          state := State.IDLE
        }

      } otherwise {

        bitTimer := bitTimer + 1
      }
    }

    // ==========================================================
    // DATA
    // ==========================================================

    is(State.DATA) {

      io.data := shiftReg

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        // UART is LSB first.
        shiftReg(bitCounter) := io.rx

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

    // ==========================================================
    // STOP
    // ==========================================================

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

## 중요한 수정 포인트

여기서 한 가지 아주 중요한 SpinalHDL/순차논리 특성이 있습니다.

예를 들어:

```scala
startSample0 := io.rx
```

를 하고 바로 다음 줄에서:

```scala
when(!startSample0 && !startSample1) {
```

라고 하면 **방금 저장한 `io.rx` 값이 즉시 `startSample0`에 반영되는 것이 아닙니다.**

`Reg`는 clock edge에서 업데이트되기 때문입니다.

따라서 위 코드는 실제로:

```text
sample0
sample1
sample2
```

를 저장하고, `sample2`가 저장되는 clock에서 majority 조건을 계산할 때 `startSample0`, `startSample1`은 이전 clock의 값이지만 **`startSample2`는 아직 이전 값**입니다.

즉, 위 코드처럼 작성하면 마지막 샘플이 majority 계산에 제대로 들어가지 않는 문제가 생깁니다.

따라서 **정확한 구현은 한 단계 더 주의해야 합니다.**

가장 안전하게는 마지막 샘플 `io.rx`를 직접 majority 식에 넣어야 합니다.

즉 START 부분은 다음과 같이 수정하는 것이 맞습니다.

```scala
is(State.START) {

  when(bitTimer === (halfBitCount - 2)) {

    startSample0 := io.rx

    bitTimer := bitTimer + 1

  } elsewhen(bitTimer === (halfBitCount - 1)) {

    startSample1 := io.rx

    bitTimer := bitTimer + 1

  } elsewhen(bitTimer === halfBitCount) {

    startSample2 := io.rx

    bitTimer := 0

    // startSample0, startSample1은 이미 저장된 값.
    // 마지막 샘플은 아직 Reg에 반영되지 않았으므로
    // 현재 io.rx를 직접 majority voting에 사용한다.

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
```

따라서 **최종적으로 사용해야 하는 전체 코드는 아래 버전**입니다.

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

  // START majority voting samples
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

    // ==========================================================
    // IDLE
    // ==========================================================

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

    // ==========================================================
    // START
    // ==========================================================

    is(State.START) {

      /*
       * 3-point majority voting.
       *
       * Example: cyclesPerBit = 10
       *
       * START bit:
       *
       * clock:  0 1 2 3 4 5 6 7 8 9
       *               ↑ ↑ ↑
       *              S0 S1 S2
       *
       * Three samples are taken around the center.
       */

      when(bitTimer === (halfBitCount - 2)) {

        startSample0 := io.rx

        bitTimer :=
          bitTimer + 1

      } elsewhen(bitTimer === (halfBitCount - 1)) {

        startSample1 := io.rx

        bitTimer :=
          bitTimer + 1

      } elsewhen(bitTimer === halfBitCount) {

        startSample2 := io.rx

        bitTimer := 0

        /*
         * Majority vote:
         *
         * At least two samples must be LOW.
         *
         * startSample0
         * startSample1
         * current io.rx = sample2
         */

        when(
          (!startSample0 && !startSample1) ||
          (!startSample1 && !io.rx) ||
          (!startSample0 && !io.rx)
        ) {

          bitCounter := 0

          state :=
            State.DATA

        } otherwise {

          state :=
            State.IDLE
        }
      } otherwise {

        bitTimer :=
          bitTimer + 1
      }
    }

    // ==========================================================
    // DATA
    // ==========================================================

    is(State.DATA) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        // UART LSB first
        shiftReg(bitCounter) :=
          io.rx

        when(bitCounter === 7) {

          state :=
            State.STOP

        } otherwise {

          bitCounter :=
            bitCounter + 1
        }

      } otherwise {

        bitTimer :=
          bitTimer + 1
      }
    }

    // ==========================================================
    // STOP
    // ==========================================================

    is(State.STOP) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        when(io.rx) {

          io.valid := True
        }

        state :=
          State.IDLE

      } otherwise {

        bitTimer :=
          bitTimer + 1
      }
    }
  }
}
```

### 한 가지 설계상의 주의점

`cyclesPerBit = 10`일 때 현재 샘플 위치는 대략:

```text
START
clock:  0 1 2 3 4 5 6 7 8 9
            ↑ ↑ ↑
            3 4 5
```

가 됩니다.

즉 **정확히 bit 중앙 주변을 3번 sampling**한다는 의도입니다.

다만 이 버전은 `cyclesPerBit`가 짝수라는 현재 학습 환경을 전제로 이해하는 것이 좋습니다. 나중에 범용 UART RX로 발전시키면서 샘플 위치 계산을 별도 설계하는 것이 좋습니다.

---

# 2. 이제 학습 주제로 복귀

UART RX majority voting은 여기서 일단 마무리하고, 원래 학습 로드맵으로 돌아가겠습니다.

현재까지:

```text
Timer
  ↓
UART TX
  ↓
UART RX
  ↓
UART Loopback
  ↓
Simple FIFO
  ↓
Valid/Ready
  ↓
Stream
  ↓
StreamFifo
  ↓
UART + Stream + FIFO 통합
  ↓
★ 현재
START majority voting
```

이제:

```text
Stream
  ↓
Flow
  ↓
StreamFifo
  ↓
SpinalHDL Library FIFO
  ↓
AXI-Stream
  ↓
Stream Pipeline
```

으로 진행하면 됩니다.

---

# 3. Stream vs Flow

가장 먼저 이 둘의 차이를 확실히 잡겠습니다.

## `Stream`

`Stream`은:

```text
valid + payload + ready
```

입니다.

```scala
val stream =
    master Stream(Bits(8 bits))
```

개념적으로:

```text
Producer                         Consumer

valid ──────────────────────────────►
data  ──────────────────────────────►
ready ◄──────────────────────────────

              transfer
                 ↓
          valid && ready
```

따라서 **backpressure가 존재합니다.**

Consumer가 준비되지 않으면:

```text
valid = 1
ready = 0
```

Producer는 데이터를 유지하면서 기다립니다.

---

# 4. `Flow`

`Flow`는 훨씬 단순합니다.

```text
valid + payload
```

만 있습니다.

즉:

```scala
val flow =
    master Flow(Bits(8 bits))
```

개념적으로:

```text
Producer                  Consumer

valid ─────────────────────►
data  ─────────────────────►
```

**ready가 없습니다.**

따라서:

```text
Flow
 ↓
backpressure 없음
```

입니다.

---

# 5. Stream vs Flow 핵심 비교

| 항목 | Stream | Flow |
|---|---|---|
| payload | O | O |
| valid | O | O |
| ready | **O** | **X** |
| backpressure | **O** | **X** |
| `fire` | O | X |
| 용도 | 데이터 전송 제어 | 이벤트/데이터 발생 전달 |
| Consumer가 기다리게 할 수 있음 | O | X |

가장 중요한 차이는 단 하나입니다.

> **Stream = valid + ready**

> **Flow = valid**

---

# 6. 언제 Flow를 사용하는가?

예를 들어 UART RX의:

```text
rx.data
rx.valid
```

을 생각해봅시다.

UART RX가 한 byte를 완성하면:

```text
valid = 1
data  = 0x41
```

이벤트를 발생시킵니다.

그런데 UART RX 자체에는:

```text
ready
```

가 없습니다.

즉 현재 우리가 만든:

```scala
val io = new Bundle {
  val rx = in Bool()

  val data  = out Bits(8 bits)
  val valid = out Bool()
}
```

은 개념적으로 **Flow와 상당히 비슷한 인터페이스**입니다.

반면:

```text
FIFO
```

는 저장 공간이 있기 때문에:

```text
나는 받을 수 있다
나는 지금 받을 수 없다
```

를 표현해야 합니다.

그래서:

```text
Stream
```

이 매우 적합합니다.

---

# 7. StreamFifo vs SpinalHDL Library FIFO

우리가 직접 만든:

```scala
case class StreamFifo(
    dataWidth: Int,
    depth: Int
)
```

는 **학습용 FIFO**입니다.

우리가 직접:

```text
memory
writePtr
readPtr
count
empty
full
push
pop
```

를 구현했습니다.

반면 SpinalHDL에는 이미 검증된 FIFO/Stream 관련 라이브러리 컴포넌트들이 있습니다.

이제 다음 단계에서는:

```text
내가 만든 StreamFifo
        VS
SpinalHDL Library FIFO
```

를 비교해서 **"실무에서는 직접 FIFO를 언제 만들고, 언제 library component를 사용하는가?"**를 배우면 됩니다.

여기서 중요한 것은 단순히 API 이름을 외우는 것이 아닙니다.

우리가 만든 FIFO의:

```text
push
pop
count
full
empty
```

가 라이브러리에서는 어떤 추상화로 표현되는지를 보는 것입니다.

---

# 8. 그리고 AXI-Stream

그 다음에는 아주 중요한 연결이 생깁니다.

우리가 지금 배운:

```text
Stream
```

의 핵심:

```text
valid
ready
payload
```

와 AXI-Stream의 핵심 handshake:

```text
TVALID
TREADY
TDATA
```

가 사실상 같은 철학을 가지고 있습니다.

```text
SpinalHDL Stream             AXI-Stream

valid       ───────────────► TVALID
ready       ◄─────────────── TREADY
payload     ───────────────► TDATA
```

그래서 지금까지 배운 `Stream`은 나중에 AXI-Stream을 이해하기 위한 **아주 좋은 기반**입니다.

---

# 9. 마지막으로 Stream Pipeline

예를 들어:

```text
Input
  │
  ▼
┌─────────┐
│ Stage 1 │
└────┬────┘
     │ Stream
     ▼
┌─────────┐
│ Stage 2 │
└────┬────┘
     │ Stream
     ▼
┌─────────┐
│ Stage 3 │
└────┬────┘
     │
     ▼
 Output
```

각 stage 사이에:

```text
valid
payload
ready
```

가 연결됩니다.

그러면 한 stage가 느려졌을 때:

```text
Stage 3
   ↓
ready = 0
```

가 upstream으로 전파되어:

```text
Stage 2
   ↓
ready = 0

Stage 1
   ↓
ready = 0
```

가 될 수 있습니다.

이것이 바로 **backpressure가 pipeline 전체를 제어하는 구조**입니다.

---

## 앞으로의 학습 흐름

따라서 이제부터는 다음 순서로 진행하면 됩니다.

```text
① Stream vs Flow
        ↓
② StreamFifo vs Library FIFO
        ↓
③ AXI-Stream
        ↓
④ Stream Pipeline
        ↓
⑤ Pipeline + FIFO + Backpressure
        ↓
⑥ UART + FIFO + Pipeline 통합
```

특히 **Stream vs Flow → Library FIFO → AXI-Stream** 순서가 중요합니다. `Stream`의 `valid/ready/fire`를 확실히 이해한 상태에서 AXI-Stream으로 넘어가면 AXI-Stream이 훨씬 쉽게 보입니다.