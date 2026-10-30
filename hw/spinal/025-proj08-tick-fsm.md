좋습니다. 이번부터는 학습 방식을 명확하게 이렇게 잡겠습니다.

> **반복은 최소화하고, 핵심 개념은 빠짐없이, 하나의 프로젝트 안에서 관련 개념을 연결해서 빠르게 진행합니다.**

그리고 모든 예제는 **SpinalHDL Release 34 / v1.15.0** 기준으로 작성하고, simulation에서는 단순 컴파일뿐 아니라 **clock/scheduling/waveform 동작까지 확인**하겠습니다. 

# Project 8 — Counter + Clock Enable + FSM

이번 프로젝트의 목표는 세 가지를 따로 배우는 것이 아니라,

```text
Counter
   ↓
Clock Enable / Tick
   ↓
FSM
```

이라는 **하나의 설계 구조**를 이해하는 것입니다.

---

## 1. 이번 프로젝트에서 배울 핵심

### Counter

순차회로의 가장 기본적인 형태입니다.

```text
0 → 1 → 2 → 3 → 4 → ... → N → 0
```

배울 것:

- `Reg`
- `:=`
- `init`
- 카운터 rollover
- clock에 따른 상태 변화

---

### Clock Enable / Tick

Counter를 이용해서 **특정 주기에 한 번씩 이벤트를 발생**시킵니다.

예를 들어:

```text
clk :  ↑ ↓ ↑ ↓ ↑ ↓ ↑ ↓ ↑ ↓ ↑ ↓ ...
count: 0   1   2   3   0   1   2   3
                    │
                   tick
```

중요한 것은 **clock을 나누는 것이 아니라 enable 신호를 만드는 것**입니다.

```text
tick = True
```

가 되는 순간에만 다음 동작을 수행하도록 합니다.

---

### FSM

Counter가 만든 `tick`을 이용해서 FSM을 움직입니다.

```text
       tick
        │
        ▼
   ┌─────────┐
   │  IDLE   │
   └────┬────┘
        │ tick
        ▼
   ┌─────────┐
   │   RUN   │
   └────┬────┘
        │ tick
        ▼
   ┌─────────┐
   │  DONE   │
   └────┬────┘
        │
        └──────> IDLE
```

이 구조 하나만 제대로 이해하면 이후 FSM 기반 controller를 만들 때 상당히 편해집니다.

---

# 2. Project 8의 최종 구조

이번에는 너무 작은 예제를 여러 개 만들지 않고 **최종적으로 하나의 작은 Controller**를 만들어 보겠습니다.

```text
                 ┌─────────────┐
                 │   Counter   │
                 └──────┬──────┘
                        │
                        ▼
                      tick
                        │
                        ▼
                 ┌─────────────┐
                 │     FSM     │
                 │             │
                 │ IDLE        │
                 │ RUN         │
                 │ DONE        │
                 └──────┬──────┘
                        │
                        ▼
                      output
```

그리고 simulation에서는

```text
clock
counter
tick
state
output
```

을 확인합니다.

---

# 3. 먼저 Counter부터

가장 기본적인 Counter입니다.

```scala
import spinal.core._

class Counter(width: Int) extends Component {
  val io = new Bundle {
    val value = out UInt(width bits)
  }

  val counter = Reg(UInt(width bits)) init(0)

  counter := counter + 1

  io.value := counter
}
```

핵심은 이것입니다.

```scala
val counter = Reg(UInt(width bits)) init(0)
```

`counter`는 combinational signal이 아니라 **register**입니다.

따라서:

```scala
counter := counter + 1
```

은 현재 값을 읽어서 다음 clock에서 새로운 값을 저장합니다.

개념적으로:

```text
현재 clock
    │
    ▼
counter = 5
    │
    │ + 1
    ▼
next value = 6
    │
    ▼
다음 clock
    │
    ▼
counter = 6
```

입니다.

---

# 4. 그런데 이 Counter에는 문제가 하나 있습니다

`UInt(width bits)`의 최대값을 넘어가면 자연스럽게 rollover됩니다.

예를 들어 3-bit라면:

```text
0
1
2
3
4
5
6
7
0
1
2
...
```

이것은 binary counter로서는 정상입니다.

하지만 실제 설계에서는 다음과 같은 경우가 많습니다.

> **0부터 9까지만 세고 다시 0으로 돌아가고 싶다.**

즉:

```text
0 → 1 → 2 → ... → 8 → 9 → 0
```

이런 Counter를 **modulo counter**라고 생각할 수 있습니다.

---

# 5. Clock Enable / Tick

이번 프로젝트에서는 이 개념이 더 중요합니다.

예를 들어 `period = 4`라고 하면:

```text
clock     0 1 2 3 4 5 6 7 8 ...
counter   0 1 2 3 0 1 2 3 0 ...
tick      0 0 0 1 0 0 0 1 0 ...
```

즉:

```text
counter == 3
```

일 때 `tick`을 발생시키는 것입니다.

개념적으로:

```scala
tick := counter === 3
```

그리고 다음 clock에서 counter를 다시 0으로 만듭니다.

---

# 6. 왜 Clock Enable을 사용하는가?

여기서 중요한 설계 습관 하나를 익힙니다.

잘못된 접근은 이런 식으로 **새로운 clock을 만들어 사용하는 것**입니다.

```text
clk
 │
 ├──────> logic A
 │
 └─ divide ──> 새로운 clock
                  │
                  └──> logic B
```

초보 단계에서는 특히 clock domain을 불필요하게 늘리지 않는 것이 좋습니다.

대신:

```text
                 ┌─────────┐
clk ────────────>│ Counter │
                 └────┬────┘
                      │
                     tick
                      │
                      ▼
                    FSM
```

처럼 **동일한 clock을 사용하고 enable/tick으로 동작 시점만 제어**합니다.

이것이 FPGA RTL 설계에서 매우 중요한 패턴입니다.

---

# 7. FSM과 연결

이제 Counter에서 만들어진 `tick`을 FSM에 연결합니다.

예를 들어:

```text
IDLE
 │
 │ tick
 ▼
RUN
 │
 │ tick
 ▼
DONE
 │
 │ tick
 ▼
IDLE
```

여기서 중요한 것은 FSM이 매 clock마다 상태를 바꾸는 것이 아니라:

```text
tick == true
```

일 때만 상태를 전환한다는 것입니다.

따라서 실제 clock은 빠르게 동작하지만 FSM의 동작은 느려집니다.

```text
clk :   ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑
tick:   0 0 0 1 0 0 0 1 0 0 0 1
FSM :   IDLE────RUN────DONE────IDLE
```

이 구조를 이해하는 것이 이번 Project 8의 핵심입니다.

---

# 8. 이번 프로젝트의 최종 목표

최종적으로 다음과 같은 작은 컴포넌트를 만들겠습니다.

### `TickController`

```text
                TickController
        ┌──────────────────────────┐
        │                          │
clk ───>│ Counter                  │
        │    │                     │
        │    └──> tick             │
        │           │              │
        │           ▼              │
        │         FSM              │
        │                          │
        │    IDLE → RUN → DONE     │
        │                          │
        └───────────┬──────────────┘
                    │
                    ▼
                  output
```

그리고 testbench에서는 단순히 "동작한다"만 보지 않고:

```text
cycle | counter | tick | state | output
----------------------------------------
  0   |    0    |  0   | IDLE  |   0
  1   |    1    |  0   | IDLE  |   0
  2   |    2    |  0   | IDLE  |   0
  3   |    3    |  1   | RUN   |   1
  4   |    0    |  0   | RUN   |   1
 ...
```

처럼 **clock-by-clock 동작을 검증**하겠습니다.

---

## Project 8 학습 범위

이번에는 다음 정도만 확실하게 익히고 넘어가겠습니다.

```text
[필수]
✓ Reg
✓ Counter
✓ rollover / modulo counter
✓ Clock Enable / Tick
✓ FSM
✓ Counter + FSM 결합
✓ Simulation timing
✓ Waveform 확인

[이번에는 깊게 다루지 않음]
✗ 복잡한 FSM 이론
✗ 여러 종류의 counter 나열
✗ pipeline
✗ clock domain crossing
✗ 고급 timing optimization
```

즉 **핵심만 압축해서 배우고 바로 다음 Project 9 — ROM + RAM으로 넘어가는 것**이 목표입니다.

다음 단계에서는 먼저 **`Counter + TickController`의 완성된 SpinalHDL v1.15.0 코드를 작성하고**, 이어서 **SpinalSim testbench를 만들어 실제 simulation 결과와 waveform을 검증**하겠습니다.

---

---

좋습니다. 이제 **Project 8 — Counter + Clock Enable + FSM**을 실제 코드로 진행하겠습니다.

이번에는 설명을 길게 늘리지 않고, **코드 → simulation → 결과 해석** 순서로 핵심만 확인하겠습니다.

# Project 8-1. Tick Counter

먼저 `Counter`와 `tick`을 하나의 컴포넌트로 만듭니다.

목표는:

```text
period = 4

clock      0  1  2  3  4  5  6  7  8
counter    0  1  2  3  0  1  2  3  0
tick       0  0  0  1  0  0  0  1  0
```

입니다.

### SpinalHDL 1.15.0

```scala
import spinal.core._

class TickCounter(period: Int) extends Component {
  require(period >= 2)

  val io = new Bundle {
    val tick  = out Bool()
    val value = out UInt(log2Up(period) bits)
  }

  val counter = Reg(UInt(log2Up(period) bits)) init(0)

  // Generate one-cycle tick when the counter reaches period - 1.
  io.tick := counter === (period - 1)

  // Counter rollover
  when(io.tick) {
    counter := 0
  } otherwise {
    counter := counter + 1
  }

  io.value := counter
}
```

## 핵심 코드

이 부분이 핵심입니다.

```scala
io.tick := counter === (period - 1)
```

그리고:

```scala
when(io.tick) {
  counter := 0
} otherwise {
  counter := counter + 1
}
```

따라서 `period = 4`이면:

```text
counter = 0
    ↓
counter = 1
    ↓
counter = 2
    ↓
counter = 3
    │
    ├── tick = 1
    ↓
counter = 0
```

가 됩니다.

---

# Project 8-2. FSM과 연결

이제 이 `tick`을 FSM의 시간 기준으로 사용합니다.

이번 FSM은 간단하게:

```text
IDLE
  │
  │ tick
  ▼
RUN
  │
  │ tick
  ▼
DONE
  │
  │ tick
  └──────> IDLE
```

로 하겠습니다.

SpinalHDL에서는 `StateMachine`을 사용합니다.

```scala
import spinal.core._
import spinal.lib.fsm._

class TickController(period: Int) extends Component {
  val io = new Bundle {
    val tick   = out Bool()
    val state  = out UInt(2 bits)
    val active = out Bool()
  }

  val timer = new TickCounter(period)

  io.tick := timer.io.tick

  val fsm = new StateMachine {
    val IDLE = new State with EntryPoint
    val RUN  = new State
    val DONE = new State

    IDLE
      .whenIsActive {
        when(timer.io.tick) {
          goto(RUN)
        }
      }

    RUN
      .whenIsActive {
        when(timer.io.tick) {
          goto(DONE)
        }
      }

    DONE
      .whenIsActive {
        when(timer.io.tick) {
          goto(IDLE)
        }
      }
  }

  io.active := fsm.isActive(fsm.RUN)

  io.state := 0
  when(fsm.isActive(fsm.IDLE)) {
    io.state := 0
  } elsewhen(fsm.isActive(fsm.RUN)) {
    io.state := 1
  } otherwise {
    io.state := 2
  }
}
```

여기서 중요한 부분은 이것입니다.

```scala
when(timer.io.tick) {
  goto(RUN)
}
```

FSM이 **매 clock마다 전환되는 것이 아니라 `tick`이 발생했을 때만 전환**됩니다.

---

# 8-3. 동작을 그림으로 보면

`period = 4`라고 하면:

```text
Clock       0   1   2   3   4   5   6   7   8   9  10  11
            │   │   │   │   │   │   │   │   │   │   │   │
Counter     0   1   2   3   0   1   2   3   0   1   2   3
Tick        0   0   0   1   0   0   0   1   0   0   0   1

FSM        IDLE───────┐
                      │
                      ▼
                      RUN────────┐
                                 │
                                 ▼
                                 DONE────────┐
                                             │
                                             ▼
                                             IDLE
```

즉 **4개의 clock마다 FSM이 한 단계 진행**합니다.

이것이 이번 프로젝트의 핵심 구조입니다.

---

# 8-4. 그런데 여기서 중요한 simulation 문제가 하나 있습니다

우리가 이전 FIFO/UART에서 경험했던 것처럼 **simulation에서는 clock scheduling을 정확하게 봐야 합니다.**

특히:

```scala
dut.io.xxx #= ...
dut.clockDomain.waitSampling()
```

의 순서와 `Reg`의 업데이트 시점을 혼동하면 예상과 실제 waveform이 달라질 수 있습니다.

이번에는 testbench도 그 부분을 명확하게 검증하도록 만들겠습니다.

예를 들어:

```scala
import spinal.core._
import spinal.core.sim._

object TickControllerSim extends App {
  SimConfig
    .withWave
    .compile(new TickController(period = 4))
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      for (cycle <- 0 until 13) {
        dut.clockDomain.waitSampling()

        println(
          f"cycle=$cycle%2d " +
          f"counter=${dut.timer.io.value.toInt}%d " +
          f"tick=${dut.io.tick.toBoolean} " +
          f"state=${dut.io.state.toInt}%d " +
          f"active=${dut.io.active.toBoolean}"
        )
      }
    }
}
```

여기서 `withWave`를 사용했기 때문에 waveform도 확인할 수 있습니다.

---

# 8-5. 예상 결과

중요한 것은 `tick`이 **한 clock 동안만 1**이라는 것입니다.

대략:

```text
cycle   counter   tick   state
--------------------------------
0          0       0      0
1          1       0      0
2          2       0      0
3          3       1      0
4          0       0      1
5          1       0      1
6          2       0      1
7          3       1      1
8          0       0      2
9          1       0      2
10         2       0      2
11         3       1      2
12         0       0      0
```

여기서 **아주 중요한 관찰점**이 있습니다.

`cycle 3`에서 `tick=1`이지만 FSM의 `state`가 바로 `RUN`으로 보이지 않을 수 있습니다.

왜냐하면:

```text
현재 clock
   │
   ├─ tick = 1을 조합적으로 판단
   │
   ├─ FSM next-state 결정
   │
   ▼
clock edge
   │
   ▼
FSM register 업데이트
```

라는 **순차회로의 clocked state update**가 있기 때문입니다.

이것이 바로 우리가 이전 UART/FIFO에서 다뤘던 **simulation scheduling과 registered state의 핵심 개념**입니다.

---

## 이번 Project 8에서 꼭 기억할 것

```text
Counter
   │
   │ 특정 주기마다
   ▼
tick
   │
   ▼
FSM state transition
```

그리고:

> **Clock Enable은 새로운 clock을 만드는 것이 아니라, 기존 clock에서 "언제 동작할 것인가"를 결정하는 신호다.**

이것만 확실히 이해하면 Project 8의 핵심은 거의 끝난 것입니다.

다음은 이 코드를 **실제로 컴파일/시뮬레이션해서 출력과 VCD waveform을 검증하는 단계**입니다.


---

SpinalHDL 시뮬레이션에서 리셋 신호를 끄는(Deassert) 방법은 **클럭 도메인을 선언한 방식과 테스트벤치 제어 방식**에 따라 몇 가지로 나뉩니다.

---

### 1. `forkStimulus`를 통한 자동 해제 (가장 추천)

가장 일반적인 방법인 `forkStimulus`를 사용하면, 시뮬레이션 시작 시 **기본 16클럭 동안 리셋이 유지된 후 자동으로 리셋이 off(deassert)** 됩니다.

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 10 타임 unités 주기로 클럭 생성 및 초기 16클럭 동안 리셋 자동 수행 후 off
  dut.clockDomain.forkStimulus(period = 10) 
  
  dut.clockDomain.waitSampling()
  // 이 시점부터는 리셋이 자동으로 off된 상태입니다.
}

```

* **리셋 유지 클럭 수 변경하기:** `resetCycles` 인자를 주어 리셋 지속 시간을 조절할 수 있습니다.
```scala
// 예: 5클럭 동안만 리셋 유지 후 자동 off
dut.clockDomain.forkStimulus(period = 10, sleepDuration = 0, resetCycles = 5)

```

---

### 2. `deassertReset()` 메서드로 직접 제어하기

직접 리셋 타이밍을 세밀하게 제어하고 싶다면 `forkStimulus` 대신 클럭과 리셋을 따로 코딩한 뒤, **`deassertReset()`** 함수를 호출하여 리셋을 끌 수 있습니다.

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 1. 클럭 토글 스레드 시작
  dut.clockDomain.forkClock(period = 10)
  
  // 2. 리셋 수동 조작
  dut.clockDomain.assertReset()   // 리셋 ON (Active)
  dut.clockDomain.waitSampling(5) // 5클럭 대기
  
  dut.clockDomain.deassertReset() // 리셋 OFF (Deassert)
  
  // 이후 시뮬레이션 로직 진행...
}

```

---

### 3. 외부 I/O 포트로 연결된 리셋 직접 제어하기 (`#=`)

만약 하드웨어 디자인 내부에서 외부 입력 핀(`io.reset`)을 받아 `ClockDomain`에 매핑한 형태라면, 시뮬레이션 안에서 해당 I/O 핀에 직접 값을 주어 리셋을 off 시킬 수 있습니다.

**하드웨어 정의 예시:**

```scala
class ACompo extends Component {
  val io = new Bundle {
    val clk   = in Bool()
    val reset = in Bool()
    // ...
  }
  val myClockDomain = ClockDomain(io.clk, io.reset)
  val core = new ClockingArea(myClockDomain) {
    val r = Reg(UInt(8 bits)) init(0)
    r.simPublic()
  }
}

```

**시뮬레이션 제어 예시:**

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 클럭 토글 시작
  fork {
    dut.io.clk #= false
    while(true) {
      sleep(5)
      dut.io.clk #= !dut.io.clk.toBoolean
    }
  }

  // Active-High 리셋인 경우 (1이 리셋, 0이 정상 동작)
  dut.io.reset #= true   // 리셋 ON
  sleep(50)
  dut.io.reset #= false  // 리셋 OFF (정상 동작 시작)
}

```
