아닙니다. 정확히 구분해야 합니다.

```
dut.clockDomain.forkStimulus(period = 10)
dut.clockDomain.waitSampling(5)

println("[INFO] Reset/clock startup complete")
```

여기서 `waitSampling(5)`는 Reset이 OFF 될 때까지 기다리는 명령이 아닙니다. Clock의 Sampling Edge가 5번 발생할 때까지 기다리는 명령입니다.

## 1. 각 명령의 의미

① `forkStimulus(period = 10)`

```
dut.clockDomain.forkStimulus(period = 10)
```

- Simulation Clock을 생성합니다.
- Clock 주기는 10 time unit입니다.
- Clock을 계속 토글하는 별도의 Simulation 프로세스를 시작합니다.
- Reset도 ClockDomain의 설정에 따라 초기화하고 제어합니다.

② `waitSampling(5)`

```
dut.clockDomain.waitSampling(5)
```

- 해당 ClockDomain의 Sampling Edge를 5번 기다립니다.
- 기본적인 Rising Edge 기준으로 생각하면 됩니다.
- Reset 해제 여부를 직접 확인하지는 않습니다.

## 2. Reset은 언제 OFF 되는가?

여기서 중요한 점은 `forkStimulus()`가 Reset을 어떻게 제어하는지는 ClockDomain의 Reset 설정에 따라 달라진다는 것입니다.

예를 들어 기본적인 동기 Reset ClockDomain이라면 다음과 같이 생각할 수 있습니다.

개념적인 Simulation 타이밍

위 그림은 설명을 위한 개념도이며 실제 forkStimulus의 Reset 파형을 측정한 것은 아닙니다.

Reset이 해제되는 정확한 시점은 실제 ClockDomain 설정을 확인해야 합니다.

## 3. 질문에 대한 직접적인 답

```
:

println("[INFO] Reset/clock startup complete")
```

이 지점에서:

| 항목    | 의미                            |
| ----- | ----------------------------- |
| Clock | 5번의 Sampling Edge가 진행됨        |
| Reset | OFF라고 단정할 수 없음                |
| CPU   | Reset 해제 여부에 따라 실행 시작 여부가 달라짐 |

따라서 `Reset/clock startup complete`라는 로그는 엄밀하게는 Reset 해제를 확인했다는 뜻이 아닙니다.

`Clock startup complete`라고 출력하는 편이 더 정확합니다.

## 4. Reset OFF를 확실하게 확인하려면?

현재 `VexRiscvTop`의 ClockDomain 설정을 확인해야 합니다.

특히 다음 부분을 보여주세요.

```
ClockDomain.current
```

를 사용하는 부분과 `VexRiscvTop`에서 ClockDomain을 생성하거나 설정하는 부분입니다.

그 코드를 확인하면 `forkStimulus()` 이후 Reset이 언제 OFF 되는지, 그리고 `waitSampling(5)`가 필요한지까지 정확히 설명할 수 있습니다.

---

---


```
dut.clockDomain.forkStimulus(period = 10)
Simulation Clock을 생성합니다.
Clock 주기는 10 time unit입니다. 
Clock을 계속 토글하는 별도의 Simulation 프로세스를 시작합니다. 
Reset도 ClockDomain의 설정에 따라 초기화하고 제어합니다.
```

1 time unit 이 1 ns 라고 한다면,
Clock 주기는 10 time unit입니다 --> 1 clock cycle(1 high level + 1 low level) 이 10 ns 가 된다는 말인가?

Clock을 계속 토글하는 별도의 Simulation 프로세스를 시작합니다.  --> 별도의 프로세스가 10 ns 동안 high, 10 ns 동안 low, 10 ns 동안 high, ... 이런 식으로 진행하나?

기본적인 동기 Reset ClockDomain이라면, reset 신호의 on --> off 변화 시점도 기본 clock 의 rising edge 에서 발생하지?

---

네. 첫 번째와 두 번째 해석은 거의 정확합니다. 다만 Clock 주기와 토글 간격을 구분하면 더 명확해집니다.

## 1. `period = 10`의 의미

`period`는 Clock의 전체 주기(High + Low) 입니다.

시뮬레이션 time unit이 1 ns라고 가정하면 다음과 같습니다.

- 전체 Clock 주기: 10 ns
- High 유지 시간: 5 ns
- Low 유지 시간: 5 ns
- Clock 전이 간격: 5 ns

따라서 파형은 다음처럼 진행됩니다.

## Clock waveform

개념도: 0\~5 ns High, 5\~10 ns Low가 반복됩니다.

따라서 질문하신 것처럼 10 ns마다 한 번의 전체 Clock cycle이 발생합니다. 다만 Clock은 10 ns마다 토글되는 것이 아니라, 5 ns마다 토글됩니다.

`forkStimulus()`는 이 Clock을 계속 생성하는 별도의 시뮬레이션 프로세스를 실행합니다. 그래서 테스트 코드가 `waitSampling()` 등으로 기다리는 동안에도 Clock은 계속 동작합니다.


## 2. Reset의 ON → OFF 시점

여기서는 두 가지를 구분해야 합니다.

- Reset 신호의 전기적 레벨 변화 시점
- 동기 Reset이 실제 레지스터에 적용되는 시점

동기 Reset(`resetKind = SYNC`)이라면 Reset 신호가 ON 또는 OFF로 바뀌더라도, 레지스터가 그 변화를 반영하는 것은 Clock의 유효 엣지에서입니다.

예를 들어 Rising Edge 기준으로 보면 다음과 같습니다.

## Synchronous Reset 개념

개념도입니다. Reset의 신호 전이는 Clock 엣지와 반드시 일치할 필요는 없습니다.

즉, Reset 신호가 Rising Edge 직전에 OFF가 되면 해당 Rising Edge에서 레지스터가 정상 동작을 재개합니다. Rising Edge 직후 OFF가 되었다면 다음 Rising Edge부터 정상 동작합니다.

중요한 점: `forkStimulus()`가 Reset을 해제하는 시점을 반드시 Rising Edge에 맞춘다고 단정하면 안 됩니다. Reset 신호의 해제 시점과 동기 Reset이 레지스터에 반영되는 시점은 별개의 문제입니다.

## 3. 한 가지 더: 현재 VexRiscvTop의 기본 ClockDomain

사용자가 보여준 코드에는 별도의 `ClockDomainConfig`가 지정되어 있지 않습니다.

SpinalHDL의 기본 ClockDomain 설정은 다음과 같습니다.

| 항목                 | 기본 설정  |
| ------------------ | ------ |
| Clock Edge         | Rising |
| Reset Kind         | ASYNC  |
| Reset Active Level | HIGH   |

즉, 현재 코드의 기본 ClockDomain은 질문에서 가정한 동기 Reset이 아니라 비동기 Active-High Reset입니다.

그리고 `forkStimulus(period = 10)`의 Reset 유지 시간은 문서상 기본 16 Clock periods로 설명되어 있습니다. 따라서 `waitSampling(5)`는 Reset을 5주기 동안 유지하라는 뜻이 아니라, Reset이 해제된 뒤 정상 Sampling Edge를 5번 기다리는 동작으로 이해하면 됩니다.

정리하면:

1. `period = 10` → 전체 Clock 주기 10 ns, High 5 ns + Low 5 ns.
2. Clock 생성 프로세스 → 5 ns 간격으로 계속 토글.
3. 동기 Reset → Reset 신호의 레벨 변화와 레지스터가 Reset을 반영하는 Clock Edge는 구분해야 함.
4. 현재 `VexRiscvTop`의 기본 Reset → 동기 방식이 아니라 비동기 Active-High 방식.

---

---

가령, 시스템이 시작되고 18 ns 후에 reset off 된다면 20 ns (기본 clock 의 세번째 rising edge) 부터 DUT 는 동작을 시작한다. 그렇지?

그런데 시뮬레이션 코드에서:
```
dut.clockDomain.forkStimulus(period = 10)
실행한 후에
dut.clockDomain.waitSampling(5)
실행하면,
```
시뮬레이션 프로세스는 5 clock cycles 기다리는 동안 clock 프로세스는 계속 진행하므로 DUT 는 세번째 clock cycle 부터 동작을 시작하여 계속 진행한다.
그래서 시뮬레이션 프로세스가 DUT 의 동작 관찰 시점을 놓치게 된다. 
그래서 시뮬레이션 코드에서
```
dut.clockDomain.forkStimulus(period = 10)
실행한 후에
dut.clockDomain.waitSampling(1? 5? 10? 20?) 
```
"그냥 적당히 충분히 기다린다" 가 아니라, 정확한 숫자가 필요하다. 너는 어떻게 분석하나?

---

질문의 핵심은 정확합니다. 테스트벤치가 기다리는 동안 DUT와 Clock은 멈추지 않으므로, 초기 대기 시간을 잘못 정하면 초기 동작을 관찰하지 못할 수 있습니다.

따라서 `waitSampling(5)`를 관례적으로 넣는 방식은 피해야 합니다. 먼저 `forkStimulus()`가 Reset을 언제 해제하는지 확인하고, 그 시점을 기준으로 관찰을 시작해야 합니다.

다만 여기서 한 가지 중요한 구분이 있습니다.

- `waitSampling(N)`은 N번의 Sampling Edge가 지날 때까지 테스트벤치 프로세스를 기다리게 합니다.
- Clock 생성 프로세스는 별도로 계속 동작합니다.
- 따라서 `waitSampling(5)`가 DUT를 5주기 정지시키는 것은 아닙니다.

이제 정확한 숫자를 결정하려면 `forkStimulus()`의 구현과 Reset 해제 시점을 기준으로 분석해야 합니다.

## 1. 먼저 질문의 가정으로 시간표를 만들겠습니다

다음 조건을 가정하겠습니다.

- Clock period = 10 ns
- Rising Edge = 0, 10, 20, 30, 40, 50, 60 ns ...
- Reset 해제 = 18 ns
- Reset은 Active-High

시간 흐름

| 시간    | 이벤트                     |
| ----- | ----------------------- |
| 0 ns  | Rising Edge             |
| 10 ns | Rising Edge             |
| 18 ns | Reset OFF               |
| 20 ns | 첫 번째 정상 동작 Rising Edge  |
| 30 ns | 두 번째 정상 동작 Rising Edge  |
| 40 ns | 세 번째 정상 동작 Rising Edge  |
| 50 ns | 네 번째 정상 동작 Rising Edge  |
| 60 ns | 다섯 번째 정상 동작 Rising Edge |

네, 이 가정에서는 DUT가 20 ns부터 정상 동작을 시작합니다. 비동기 Reset이므로 18 ns에 Reset이 해제되면 Reset 상태에서 벗어나고, 순차 로직은 다음 유효 Rising Edge인 20 ns부터 정상적으로 동작합니다.

## 2. `waitSampling(5)`는 정확히 무엇을 기다리는가?

여기서 중요한 사실이 있습니다.

SpinalHDL의 `waitSampling()`은 단순히 Clock Edge만 기다리는 것이 아닙니다.

Reset이 해제되어 있고 Clock Enable이 활성화된 Sampling Edge를 기다립니다. 따라서 Reset이 활성화된 동안의 Rising Edge는 카운트하지 않습니다.

따라서 위 시간표에서는 다음과 같습니다.

| 코드                | 기다리는 Sampling Edge    | 반환 시점 |
| ----------------- | --------------------- | ----- |
| `waitSampling(1)` | 20 ns                 | 20 ns |
| `waitSampling(2)` | 20, 30 ns             | 30 ns |
| `waitSampling(5)` | 20, 30, 40, 50, 60 ns | 60 ns |

즉, 질문하신 우려가 맞습니다.

`waitSampling(5)`를 실행하면 DUT는 20 ns부터 이미 동작하고, 테스트벤치는 60 ns가 될 때까지 기다립니다. 따라서 20\~50 ns 사이에 발생한 동작을 별도로 관찰하지 않았다면 놓치게 됩니다.

## 3. 그러면 `waitSampling(1)`을 사용하면 충분한가?

반드시 그렇지는 않습니다.

`waitSampling(1)`은 첫 번째 정상 Sampling Edge까지 기다리는 용도입니다. 하지만 그 Edge에서 발생하는 동작을 관찰하려는 경우에는 시뮬레이터의 이벤트 처리 순서와 관찰 시점을 고려해야 합니다.

특히 기존 테스트벤치처럼 다음 구조를 사용한다면 주의해야 합니다.

```
dut.clockDomain.forkStimulus(period = 10)

dut.clockDomain.waitSampling(5)

// DUT 관찰 시작
for (cycle <- 1 to 100) {
  dut.clockDomain.waitFallingEdge()
  // 신호 관찰
}
```

이 코드는 처음 5개의 정상 Sampling Edge를 지나간 다음에야 관찰을 시작합니다.

초기 동작부터 관찰해야 한다면 이 구조는 적절하지 않습니다.

## 4. 내가 권장하는 분석 방법

여기서는 `waitSampling(N)`으로 초기화 시간을 추측하는 대신, 역할을 분리하는 것이 좋습니다.

1. `forkStimulus()`로 Clock과 Reset 프로세스를 시작합니다.
2. 곧바로 관찰 프로세스를 준비합니다.
3. Reset이 해제되는 시점부터 첫 번째 정상 동작을 관찰합니다.
4. 그 이후에는 실제 DUT의 동작 조건에 맞춰 매 사이클 관찰합니다.

그리고 한 가지 정정할 부분이 있습니다. 앞서 제가 `forkStimulus()`의 Reset 유지 시간을 충분히 명시하지 않았습니다. 
현재 확인한 SpinalHDL 시뮬레이션 문서에서는 기본 `forkStimulus(period)`의 Reset 유지 시간이 16 Clock periods라고 설명합니다. 
별도의 `resetCycles` 인자로 변경할 수도 있습니다.

그러므로 현재 프로젝트에서 `forkStimulus(period = 10)`을 호출하면 `waitSampling(5)`가 Reset 해제 전부터 단순히 5번 세는 동작은 아닙니다. 
Reset이 해제된 후의 Sampling Edge 5개를 기다리는 것입니다.

제안: 다음 단계에서는 현재 `VexRiscvTopSim.scala`를 기준으로, 첫 번째 DBus 명령부터 놓치지 않는 관찰 구조를 만들겠습니다. 
특히 기존 코드의 `waitSampling(5)`를 무조건 `waitSampling(1)`로 바꾸는 식으로 처리하지 않고, Reset 해제와 DBus handshake 관찰 시점을 정확히 분리하겠습니다.

---

---

그러므로 현재 프로젝트에서 `forkStimulus(period = 10)`을 호출하면 `waitSampling(5)`가 Reset 해제 전부터 단순히 5번 세는 동작은 아닙니다. Reset이 해제된 후의 Sampling Edge 5개를 기다리는 것입니다 --> 그러니까, reset 이 해제된 후 DUT 는 바로 동작을 시작 했는데, 관찰자는 Sampling Edge 5개를 기다리니까 DUT 초기 동작 관찰 기회를 놓치는 거 아니냐. Sampling Edge 라는 것이 정확이 뭐냐? 기본 clock 의 rising edge 아니냐?

---

네. 정확히 이해하셨습니다. 질문하신 두 가지 모두 맞습니다.

제가 앞서 설명한 `waitSampling(5)`의 의미를 다시 명확하게 정리하겠습니다.

## 1. Sampling Edge란 무엇인가?

기본 ClockDomain 설정에서 Sampling Edge는 Clock의 Rising Edge입니다.

다만 Reset과 Clock Enable 조건이 추가됩니다.

SpinalHDL 문서의 정의는 다음과 같습니다.

$$ SamplingEdge = RisingEdge \land ResetInactive \land ClockEnableActive $$

즉, 기본 설정에서는 다음과 같습니다.

| 조건                                         | Sampling Edge로 인정? |
| ------------------------------------------ | ------------------ |
| Rising Edge + Reset ON                     | 아니요                |
| Rising Edge + Reset OFF                    | 예                  |
| Falling Edge + Reset OFF                   | 아니요                |
| Rising Edge + Reset OFF + Clock Enable OFF | 아니요                |

이것은 SpinalHDL의 `waitSampling()`과 `waitRisingEdge()`가 서로 다른 이유이기도 합니다.

## 2. 질문하신 상황을 시간순으로 보면

다음 조건을 가정하겠습니다.

- Clock period = 10 ns
- Rising Edge = 0, 10, 20, 30, 40, 50, 60 ns ...
- Reset OFF = 18 ns
- Clock Enable = 항상 ON

시간 흐름

| 시간    | Clock  | Reset | Sampling |
| ----- | ------ | ----- | -------- |
| 0 ns  | Rising | ON    | 불가       |
| 10 ns | Rising | ON    | 불가       |
| 18 ns | -      | OFF   | -        |
| 20 ns | Rising | OFF   | 1번째      |
| 30 ns | Rising | OFF   | 2번째      |
| 40 ns | Rising | OFF   | 3번째      |
| 50 ns | Rising | OFF   | 4번째      |
| 60 ns | Rising | OFF   | 5번째      |

따라서:

```
dut.clockDomain.waitSampling(1)
```

은 20 ns에 반환합니다.

```
dut.clockDomain.waitSampling(5)
```

은 60 ns에 반환합니다.

그리고 DUT는 20 ns부터 이미 동작하고 있습니다.

따라서 질문하신 대로 20\~50 ns의 초기 동작은 `waitSampling(5)`가 끝난 다음에 관찰하기 시작하면 이미 지나간 상태입니다.

이 부분은 제가 앞서 설명한 내용 중 가장 중요하게 강조했어야 하는 부분입니다.

## 3. `waitSampling()`과 `waitRisingEdge()` 비교

이 차이를 기억하시면 좋겠습니다.

| 함수                   | 기다리는 이벤트                                       |
| -------------------- | ---------------------------------------------- |
| `waitSampling(5)`    | Reset 및 Clock Enable 조건을 만족하는 Sampling Edge 5개 |
| `waitRisingEdge(5)`  | Reset 상태와 무관하게 Rising Edge 5개                  |
| `waitFallingEdge(5)` | Reset 상태와 무관하게 Falling Edge 5개                 |

초기 Reset 동작을 포함한 시간 제어가 필요하다면 `waitRisingEdge()`가 더 직접적인 기준이 될 수 있습니다.

## 4. 현재 VexRiscv 테스트벤치에 적용할 원칙

저는 다음 구조를 권장합니다.

```
dut.clockDomain.forkStimulus(period = 10)

// 초기 대기 시간을 임의로 지정하지 않는다.
// 관찰 프로세스를 가능한 한 일찍 시작한다.

for (cycle <- 1 to 100) {
  dut.clockDomain.waitFallingEdge()

  // DBus 명령과 응답 관찰
}
```

이렇게 하면 Reset이 해제된 뒤 첫 번째 Falling Edge부터 관찰할 수 있습니다.

단, 첫 번째 DBus 명령이 언제 발생하는지는 CPU의 Reset 해제 후 내부 동작에 따라 달라집니다. 
따라서 Falling Edge 관찰을 시작했다고 해서 첫 DBus 명령이 반드시 그 시점에 존재한다는 뜻은 아닙니다.

결론: 지금 지적하신 문제가 정확합니다. `waitSampling(5)`는 초기 안정화를 위한 무해한 대기가 아닙니다. 
DUT가 동작하는 시간을 소비하는 것이므로, 초기 동작을 관찰해야 하는 테스트벤치에서는 근거 없이 넣으면 안 됩니다.

---
