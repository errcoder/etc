UartTx.io.data 가 0x41 인 경우,
io.tx 에서 전송되는 bit 순서는 0(START), 1, 0,0,0,0,0,1,0, 1(STOP) 이고 10 bits 전부 전송하는데 걸리는 clk 횟수는 100 clk cycle 소요되지?
그런데, wave.vcd 파형을 GTKWave로 관찰하니 clk 범위가 10 사이클 범위만 관찰되고 io.tx 는 IDLE -> START (1 -> 0) 파형만 보인다.
START + DATA(8 bits) + STOP 전체 전송되는 io.tx(10 bits) 파형을 관찰하려면 최소 10x10 = 100 clk 파형의 범위가 표시되어여 하는 것 아니냐?

UartTxSim 코딩을 잘못한 것이냐? 

맞습니다. **제가 처음 코드를 작성할 때 그 부분을 놓쳤습니다.**  
특히 `while (dut.io.busy.toBoolean)` 같은 simulation 제어 코드는 RTL 자체와 별개로 **SpinalSim의 scheduling 시점까지 고려해서 검증했어야 했습니다.**

이번 경우에는 제가 처음부터 다음을 확인했어야 합니다.

- `valid`를 어느 clock에서 인가하는가
- `state`와 `busy`가 어느 simulation 시점에 갱신되는가
- `waitSampling()` 직후 `busy`를 읽을 때 어떤 값이 관찰되는가
- simulation이 실제 UART frame 전체를 기록하기 전에 종료되지 않는가
- 생성된 VCD에서 START → DATA → STOP 전체가 실제로 존재하는가
- `0x41`에 대해 **10 bit × 10 clock = 100 clock**이 실제 waveform으로 확인되는가

실제로 첨부하신 VCD를 확인해 보니 simulation은 `START`에 들어간 뒤 `bitTimer`가 4까지밖에 가지 않고 종료됐습니다. wave

그래서 앞으로는 특히 **simulation 코드까지 RTL 코드와 동일하게 검증 대상**으로 취급하겠습니다.

### 앞으로 적용할 원칙

1. **SpinalHDL Release 34 / v1.15.0 기준으로 작성**
2. 단순히 "컴파일될 것 같은 코드"가 아니라 **simulation scheduling까지 고려**
3. `while` 기반의 simulation 종료 조건은 특별히 주의
4. 가능하면 학습용 예제에서는 **명확한 clock 수를 지정하여 waveform을 충분히 생성**
5. UART/FIFO/FSM 같은 코드는 **예상 동작을 먼저 계산하고 waveform 결과와 대조**
6. 사용자가 실제 VCD를 제공하면 **추측하지 않고 VCD를 직접 분석**
7. 코드에서 애매한 부분이나 버전 의존적인 부분은 **v1.15.0 기준으로 확인한 후 제시**

그리고 중요한 점 하나는, **앞으로 제가 이전에 작성한 코드라고 해서 그대로 재사용하지 않겠습니다.** 필요하면 처음부터 다시 검토해서 더 안전한 코드로 제시하겠습니다.

이번 건은 제가 놓친 것이 맞습니다. 지적해 주셔서 감사합니다. 👍

---

네. **사용자님의 계산이 맞습니다.**  
그리고 지금 보신 현상은 **`UartTx` 자체가 10-bit 전송을 안 한 것이라기 보다는, GTKWave에서 처음 10 clock 정도만 확대되어 보이는 가능성이 매우 높습니다.**

다만 `UartTxSim` 코드도 같이 확인해 보면 더 명확합니다.

# 1. 먼저 `0x41`의 실제 UART TX bit 순서

`0x41`은 binary로:

```text
0x41 = 0100_0001
```

UART는 **LSB first**이므로 DATA bit 순서는:

```text
D0 D1 D2 D3 D4 D5 D6 D7
 1  0  0  0  0  0  1  0
```

그리고 UART frame은:

```text
START   DATA                         STOP
  0     1 0 0 0 0 0 1 0              1
```

즉:

```text
io.tx:

0 | 1 | 0 | 0 | 0 | 0 | 0 | 1 | 0 | 1
  START       DATA                STOP
```

정확히 **10 bit**입니다.

---

# 2. 1 bit당 10 clock이 맞다

현재 Simulation 설정은:

```scala
val clockFrequency = 10_000_000
val baudRate = 1_000_000
```

따라서:

```text
clockFrequency = 10 MHz
baudRate       = 1 Mbps
```

이고,

```scala
cyclesPerBit =
  clockFrequency / baudRate
```

이므로:

```text
10,000,000 / 1,000,000
= 10
```

즉,

```text
UART 1 bit
    ↓
10 system clock cycles
```

입니다.

---

# 3. 따라서 전체 frame은 정확히 100 clock

UART frame:

```text
START + DATA8 + STOP
```

이므로:

```text
1 + 8 + 1 = 10 bits
```

1 bit당 10 clock이니까:

```text
10 bits × 10 clocks/bit
= 100 clocks
```

따라서 **전체 UART frame을 보려면 최소 약 100 clock cycle을 관찰해야 합니다.**

사용자님의 생각이 정확합니다.

---

# 4. 그런데 `forkStimulus(10)`의 `10`은 "10 clock만 실행"이라는 뜻이 아니다

여기서 중요한 부분입니다.

Simulation 코드:

```scala
dut.clockDomain.forkStimulus(10)
```

이 `10`은 **simulation clock의 주기(period)** 입니다.

즉 개념적으로:

```text
clock period = 10 simulation time units
```

입니다.

그래서:

```text
1 clock = 10 time units
10 clocks = 100 time units
100 clocks = 1000 time units
```

입니다.

즉:

```text
forkStimulus(10)
```

때문에 simulation이 10 clock만 실행되는 것이 **아닙니다.**

---

# 5. 현재 `UartTxSim`은 실제로 100 clock 이상 실행한다

이 부분을 보세요.

```scala
while (dut.io.busy.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

`busy`는:

```scala
io.busy := state =/= State.IDLE
```

입니다.

즉:

```text
IDLE  → busy = 0
START → busy = 1
DATA  → busy = 1
STOP  → busy = 1
IDLE  → busy = 0
```

입니다.

따라서:

```scala
while (dut.io.busy.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

가 **UART 전송이 끝날 때까지 계속 기다립니다.**

그 다음:

```scala
dut.clockDomain.waitSampling(5)
```

까지 있으므로, 정상적으로라면 waveform에는 최소한:

```text
UART transmission
      +
5 clock
```

정도가 존재합니다.

---

# 6. 그런데 왜 GTKWave에서는 10 clock 정도만 보이는가?

이게 핵심입니다.

**GTKWave의 화면에 처음부터 전체 simulation 시간이 자동으로 확대되어 표시된다는 보장은 없습니다.**

예를 들어 실제 VCD가:

```text
0
        ↓
        ┌───────────────────────────────────────────────┐
        │               100 clocks                      │
        └───────────────────────────────────────────────┘
                                                        ↓
                                                     105 clocks
```

까지 기록되어 있어도 GTKWave가 처음 열렸을 때:

```text
0 ─────────────────────────── 100 ns
```

정도만 확대해서 보여줄 수 있습니다.

현재 clock이 10 time unit 주기라면:

```text
0
10
20
30
40
50
60
70
80
90
100
```

까지만 화면에 보이면 **딱 10 clock**처럼 보입니다.

그래서 사용자가 보신:

```text
clk
_   _   _   _   _   _   _   _   _   _
 \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/ \_/

io.tx
───────┐
       └────────────────
       START
```

같은 모습은 **simulation이 10 clock만 실행됐다는 증거가 아닙니다.**

---

# 7. GTKWave에서 전체 waveform을 보려면

GTKWave에서 **전체 시간 범위로 Zoom Out / Fit**을 해보세요.

보통 toolbar의:

```text
Zoom Fit
```

기능을 사용하면 됩니다.

그러면 대략:

```text
Time
0                                                   1050
|-----------------------------------------------------|

clk
_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾_‾...

tx
──────────┐
          │
          └────────┐
                   │
                   └────────┐
                            ...
```

처럼 전체 simulation이 보일 겁니다.

---

# 8. 그런데 한 가지 더 중요한 문제가 있습니다

현재 `UartTxSim`은:

```scala
dut.io.valid #= true

dut.clockDomain.waitSampling()

dut.io.valid #= false
```

입니다.

이 순간을 정확히 이해해야 합니다.

### 처음

```text
state = IDLE
busy  = 0
tx    = 1
```

UART idle 상태에서는 `tx = 1`입니다.

---

### `valid = true`

```text
valid = 1
```

그리고 다음 clock에서:

```scala
when(io.valid) {
  shiftReg := io.data
  state := State.START
}
```

가 실행됩니다.

따라서:

```text
IDLE
 ↓
START
```

가 됩니다.

그리고 START 상태에서:

```scala
io.tx := False
```

이므로:

```text
tx:

1
↓
0
```

가 됩니다.

바로 이것이 사용자가 GTKWave에서 처음 발견한:

```text
IDLE → START

1 → 0
```

부분입니다.

**하지만 이게 전체 전송의 끝이 아닙니다.**

이제부터 약 100 clock 동안 계속 전송해야 합니다.

---

# 9. 전체 waveform을 더 쉽게 확인하려면 Simulation을 조금 바꿔보자

학습 단계에서는 `while`보다는 아예 **120 clock을 기다리도록 작성하는 것**이 waveform 확인에 훨씬 좋습니다.

예를 들어:

```scala
import spinal.core._
import spinal.core.sim._

object UartTxSim extends App {

  val clockFrequency = 10_000_000
  val baudRate       = 1_000_000

  val cyclesPerBit =
    clockFrequency / baudRate

  SimConfig
    .withWave
    .doSim(
      UartTx(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      dut.io.data  #= 0
      dut.io.valid #= false

      dut.clockDomain.forkStimulus(10)

      // UART idle
      dut.clockDomain.waitSampling(5)

      // Send 0x41
      dut.io.data  #= 0x41
      dut.io.valid #= true

      dut.clockDomain.waitSampling()

      dut.io.valid #= false

      // 전체 UART frame 관찰
      // START + DATA(8) + STOP = 10 bits
      // 1 bit = 10 clocks
      // 따라서 100 clocks
      dut.clockDomain.waitSampling(cyclesPerBit * 12)

      println("UART TX simulation finished")
    }
}
```

여기서:

```scala
cyclesPerBit * 12
```

은:

```text
10 × 12 = 120 clocks
```

입니다.

즉 100 clock frame을 전부 보고도 20 clock 정도 여유가 있습니다.

---

# 10. GTKWave에서 기대해야 하는 waveform

`0x41`을 보내면 대략 다음과 같은 모양이 나와야 합니다.

```text
UART TX

IDLE
TX = 1
───────────────────┐
                   │
                   ↓
START
TX = 0
                   ┌──────────────────────────────────────
                   │
                   │ 10 clocks
                   │
                   ↓
DATA D0 = 1
TX = 1
───────────────────┐
                   │
                   │ 10 clocks
                   ↓
DATA D1 = 0
TX = 0
                   │
                   │ 10 clocks
                   ↓
DATA D2 = 0
TX = 0
                   │
                   │
                   ↓
DATA D3 = 0
TX = 0
                   │
                   ↓
DATA D4 = 0
TX = 0
                   │
                   ↓
DATA D5 = 0
TX = 0
                   │
                   ↓
DATA D6 = 1
TX = 1
                   │
                   ↓
DATA D7 = 0
TX = 0
                   │
                   ↓
STOP
TX = 1
───────────────────
```

좀 더 압축해서 표현하면:

```text
             10 clocks each
        <-------------------->

TX = 1    IDLE
           |
           ↓
          0000000000       START
          1111111111       D0
          0000000000       D1
          0000000000       D2
          0000000000       D3
          0000000000       D4
          0000000000       D5
          1111111111       D6
          0000000000       D7
          1111111111       STOP
                                               
        <--------- 100 clocks --------->
```

따라서 `io.tx`는 최종적으로:

```text
0 1 0 0 0 0 0 1 0 1
```

각 bit가 **10 clock씩 유지되는 형태**가 되어야 합니다.

---

# 11. 아주 중요한 포인트: "bit"와 "clock"을 구분해야 한다

이번 질문에서 이 부분을 정확히 이해하면 UART가 훨씬 쉬워집니다.

우리는 두 개의 시간 단위를 가지고 있습니다.

### System Clock

```text
10 MHz

      ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
CLK   │ │ │ │ │ │ │ │ │ │
──────┘ └─┘ └─┘ └─┘ └─┘ └──
```

10 MHz니까:

```text
1 clock = 100 ns
```

---

### UART Bit

Baud rate가:

```text
1 Mbps
```

이므로:

```text
1 bit = 1 us
```

그리고:

```text
1 us / 100 ns
= 10 clocks
```

따라서:

```text
           UART bit
       <------------>

CLK    _|‾|_|‾|_|‾|_|‾|_|‾|_
       ← 10 system clocks →

TX     ─────────────────────
             1 bit
```

입니다.

---

# 12. 그래서 결론

질문하신 내용에 하나씩 답하면:

| 질문 | 답 |
|---|---|
| `0x41`의 TX 순서가 `0,1,0,0,0,0,0,1,0,1`인가? | **맞습니다.** |
| START + DATA8 + STOP = 10 bit인가? | **맞습니다.** |
| 1 bit = 10 clock인가? | **맞습니다.** |
| 전체 frame = 100 clock인가? | **맞습니다.** |
| GTKWave에서 최소 100 clock 정도를 보면 전체 frame을 볼 수 있는가? | **맞습니다.** |
| 현재 `UartTxSim`이 10 clock만 실행하는가? | **아닙니다.** |
| `forkStimulus(10)`이 10 clock만 실행한다는 뜻인가? | **아닙니다.** |
| GTKWave에서 10 clock 정도만 보이는 이유는? | **현재 화면의 time range/zoom이 전체 VCD보다 좁게 잡혀 있을 가능성이 높습니다.** |

그리고 **현재 `UartTxSim` 코드는 `while (dut.io.busy.toBoolean)` 때문에 정상적으로라면 UART transmission이 끝날 때까지 simulation을 계속 진행합니다.**

---

## 다음 단계에서 확인하면 아주 좋습니다

GTKWave에서 단순히 `io.tx`만 보는 것보다 다음 신호를 같이 넣어 보세요.

```text
clk
io.valid
io.data
io.tx
io.busy
state
bitTimer
bitCounter
shiftReg
```

그러면 정확히:

```text
IDLE
  ↓
START
  ↓
DATA D0
  ↓
DATA D1
  ↓
...
  ↓
DATA D7
  ↓
STOP
  ↓
IDLE
```

로 FSM이 움직이는 것이 눈에 보입니다.

특히 **`bitTimer`, `bitCounter`, `shiftReg`까지 같이 보면 "왜 `tx`가 10 clock 동안 같은 값을 유지하는지"가 완전히 연결되어 보입니다.** 이 부분을 이해하면 지금 작성한 `UartTx` 코드가 하드웨어에서 어떻게 동작하는지 거의 완전히 이해하게 됩니다.

---

---

wave.vcd 파일을 첨부한다. 직접 조사해 보아라.

네. `wave.vcd`를 직접 확인해 보니, **원인이 명확합니다.**

결론부터 말씀드리면:

> **`UartTx` 회로 자체의 문제라기보다, 이전에 작성한 `UartTxSim`의 `while (dut.io.busy.toBoolean)` 부분 때문에 simulation이 너무 일찍 끝난 것입니다.**

그리고 이 VCD는 아주 좋은 증거를 보여주고 있습니다.

# 1. 실제 `wave.vcd`는 270 ps에서 끝납니다

VCD의 timescale이:

```text
$timescale 1ps $end
```

입니다. wave

그리고 마지막 timestamp가:

```text
#270
```

입니다. wave

즉 simulation은:

```text
0 ps ~ 270 ps
```

까지만 기록되었습니다.

---

# 2. Clock은 10 ps 주기입니다

VCD를 보면:

```text
#170
1/
#175
0/
#180
1/
#185
0/
#190
1/
#195
0/
...
```

입니다. wave

즉:

```text
CLK period = 10 ps
```

입니다.

따라서 270 ps까지라면 대략:

```text
270 ps / 10 ps
= 27 clock cycles
```

입니다.

그런데 중요한 것은 **UART 전송이 시작된 시점**입니다.

---

# 3. `0x41`을 넣은 시점은 #210입니다

VCD에서:

```text
#210
b01000001 +
1,
1/
```

입니다. wave

여기서:

```text
io_data  = 0x41
io_valid = 1
```

이 됩니다.

즉 이 시점부터 UART 전송 요청이 들어갑니다.

그리고 다음 clock인 `#220`에서:

```text
b01000001 %
b01 &
...
0,
0-
1.
1/
```

입니다. wave

여기서 매우 중요한 것이:

```text
shiftReg = 0x41
state    = START
io_valid = 0
io_tx    = 0
io_busy  = 1
```

이라는 것입니다.

즉 정확하게:

```text
IDLE
  ↓
START
```

전환이 발생했습니다.

그리고 `io.tx`도:

```text
1
↓
0
```

으로 바뀌었습니다.

따라서 **UART TX 회로가 START bit를 정상적으로 출력하고 있습니다.**

---

# 4. 그런데 왜 DATA로 넘어가지 않았을까?

그 다음 waveform을 보면:

```text
#230   bitTimer = 1
#240   bitTimer = 2
#250   bitTimer = 3
#260   bitTimer = 4
#270
```

입니다. wave

즉:

```text
START
│
├── bitTimer = 0
├── bitTimer = 1
├── bitTimer = 2
├── bitTimer = 3
├── bitTimer = 4
└── simulation 종료
```

입니다.

**START bit가 10 clock 동안 유지되기 전에 simulation이 끝나버렸습니다.**

원래라면:

```text
START
bitTimer = 0
bitTimer = 1
bitTimer = 2
...
bitTimer = 9
        ↓
DATA
```

로 가야 합니다.

---

# 5. 그러면 `while (busy)`가 왜 문제인가?

이전에 사용했던 코드가:

```scala
dut.io.valid #= true

dut.clockDomain.waitSampling()

dut.io.valid #= false

while (dut.io.busy.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

였죠.

여기서 초보자에게 상당히 헷갈리는 부분이 하나 있습니다.

`waitSampling()` 직후에 `busy`를 읽는 순간의 **simulation scheduling / clock-domain 업데이트 순서** 때문에, 우리가 기대하는 것과 실제 관찰 시점이 어긋날 수 있습니다.

특히 이 코드에서는:

```scala
dut.clockDomain.waitSampling()

dut.io.valid #= false

while (dut.io.busy.toBoolean) {
```

가 너무 빠르게 `busy`를 검사합니다.

결과적으로:

```text
valid 요청
     ↓
clock edge
     ↓
START로 전환되는 과정
     ↓
simulation thread에서 busy 검사
     ↓
busy가 아직 기대한 값으로 관찰되지 않음
     ↓
while 종료
     ↓
마지막 waitSampling(5)
     ↓
simulation 종료
```

와 같은 상황이 발생할 수 있습니다.

실제 VCD가 이것을 그대로 보여주고 있습니다.

---

# 6. 특히 VCD가 결정적인 증거입니다

`START` 진입 시점인 `#220`에서:

```text
io_tx   = 0
io_busy = 1
state   = START
```

가 이미 기록되어 있습니다. wave

따라서 회로는 **정상적으로 START 상태에 들어갔습니다.**

그런데 simulation은:

```text
#220
START

#230
bitTimer = 1

#240
bitTimer = 2

#250
bitTimer = 3

#260
bitTimer = 4

#270
END
```

에서 끝났습니다. wave

즉 **UART TX가 DATA를 못 보내는 것이 아니라 simulation이 START bit 도중 종료된 것입니다.**

---

# 7. 그래서 지금은 `busy`를 기다리지 말고 120 clock을 직접 기다리는 것이 좋습니다

지금은 학습 단계이므로 가장 명확하게 이렇게 해보는 것을 추천합니다.

```scala
import spinal.core._
import spinal.core.sim._

object UartTxSim extends App {

  val clockFrequency = 10_000_000
  val baudRate       = 1_000_000

  val cyclesPerBit =
    clockFrequency / baudRate

  SimConfig
    .withWave
    .doSim(
      UartTx(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      dut.io.data  #= 0
      dut.io.valid #= false

      dut.clockDomain.forkStimulus(10)

      // IDLE 상태 관찰
      dut.clockDomain.waitSampling(5)

      // --------------------------------
      // 0x41 전송 요청
      // --------------------------------
      dut.io.data  #= 0x41
      dut.io.valid #= true

      dut.clockDomain.waitSampling()

      dut.io.valid #= false

      // --------------------------------
      // UART frame 전체 관찰
      //
      // START = 1 bit
      // DATA  = 8 bits
      // STOP  = 1 bit
      //
      // 총 10 bits
      //
      // 1 bit = 10 clocks
      //
      // 총 100 clocks
      // --------------------------------

      dut.clockDomain.waitSampling(
        cyclesPerBit * 12
      )

      println("UART TX simulation finished")
    }
}
```

이렇게 하면:

```text
cyclesPerBit = 10
```

이고:

```text
12 × 10 = 120 clocks
```

동안 simulation이 진행됩니다.

---

# 8. 그러면 GTKWave에서 이런 waveform을 볼 수 있어야 합니다

대략:

```text
                 START        DATA                         STOP
                   │                                      │
TX                 ↓                                      ↓

1 ────────┐        ┌──────────┐              ┌───────────
          │        │          │              │
          │        │          │              │
0         └────────┘          └─── ... ─────┘

          <10clk>  <10clk> <10clk> ... <10clk>
```

정확한 값은:

```text
START   D0 D1 D2 D3 D4 D5 D6 D7   STOP
  0      1  0  0  0  0  0  1  0     1
```

각각 10 clock씩:

```text
0   × 10 clocks
1   × 10 clocks
0   × 10 clocks
0   × 10 clocks
0   × 10 clocks
0   × 10 clocks
0   × 10 clocks
1   × 10 clocks
0   × 10 clocks
1   × 10 clocks
```

따라서 총:

```text
10 × 10
= 100 clocks
```

입니다.

---

# 9. 그리고 이번 VCD에서 아주 좋은 것을 하나 발견할 수 있습니다

VCD에 다음 내부 신호들도 전부 들어 있습니다.

```text
state
bitTimer
bitCounter
shiftReg
```

VCD 변수 정의에도 실제로:

```text
bitTimer   [3:0]
bitCounter [2:0]
shiftReg   [7:0]
state_1
```

가 기록되어 있습니다. wave

이건 아주 좋습니다.

다음 simulation에서는 GTKWave에서 최소한 다음을 같이 관찰해 보세요.

```text
clk
io_valid
io_tx
io_busy
state_1
bitTimer
bitCounter
shiftReg
```

그러면:

```text
               state
                │
                ▼
IDLE → START → DATA → STOP → IDLE
         │       │       │
         │       │       └── tx = 1
         │       │
         │       └── bitCounter = 0...7
         │
         └── tx = 0
             bitTimer = 0...9
```

라는 구조가 waveform에서 그대로 보이게 됩니다.

---

# 10. 이번에 확인된 핵심

따라서 제가 앞서 말씀드렸던 **"GTKWave의 zoom 문제일 가능성이 높다"**는 설명은 이 `wave.vcd`를 실제로 확인해 보니 **정확하지 않았습니다.**

이번 VCD를 보면 실제로 simulation 자체가 `#270`에서 끝났습니다. wave

즉:

> **GTKWave가 100 clock을 숨기고 있었던 것이 아니라, VCD 자체에 전체 100 clock이 기록되지 않았습니다.**

그리고 그 원인은 **`UartTxSim`의 `busy` 대기 방식**으로 보는 것이 맞습니다.

이건 오히려 좋은 발견입니다. 이제 **"SpinalHDL 회로의 clocked logic"과 "SpinalHDL simulation의 scheduling"이 서로 어떻게 연결되는지**를 배울 수 있는 단계입니다.

다음에는 이 `UartTxSim`을 조금 더 제대로 고쳐서, **`0x41`의 START → D0 → D1 → ... → D7 → STOP이 실제 100 clock 동안 어떻게 움직이는지**를 GTKWave waveform과 1:1로 대응시켜 설명하는 것이 좋습니다.


---

---

**SpinalHDL Release 34 / v1.15.0을 기준**으로 코드를 작성하겠습니다. v1.15.0은 2026년 9월 2일 공개된 릴리스이며 Scala 2.12/2.13 빌드를 제공합니다. 
[Hivebook](https://hivebook.wiki/wiki/spinalhdl-scala-hdl-generating-vhdl-and-verilog?utm_source=chatgpt.com)

아래 예제는 **초보자가 Timer → UART TX → UART RX 순서로 학습할 수 있도록 직접 RTL을 작성하는 방식**으로 구성하겠습니다. 
SpinalHDL의 `spinal.lib.UartCtrl`을 바로 사용하는 예제도 마지막에 별도로 보여드리겠습니다. 공식 UART 구현 역시 TX/RX를 별도의 컴포넌트와 sampling tick, FSM으로 구성합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/Intermediates%20ones/uart.html?utm_source=chatgpt.com)

---

# 1. 공통 프로젝트 환경

## 1.1 버전

이번 예제에서 고정하는 버전은 다음과 같습니다.

| 항목 | 버전 |
|---|---|
| SpinalHDL | **1.15.0** |
| Release | **Release 34** |
| Scala | **2.13 계열 권장** |
| JDK | 17 LTS 권장 |
| Build | sbt |
| Simulation | SpinalSim + Verilator |

SpinalHDL 1.15.0에서는 `_2.12`, `_2.13` artifact가 제공되며 Scala 3 빌드는 제공되지 않습니다. 
[Hivebook](https://hivebook.wiki/wiki/spinalhdl-scala-hdl-generating-vhdl-and-verilog?utm_source=chatgpt.com)

---

# 2. `build.sbt`

세 프로젝트에서 공통으로 사용할 수 있는 `build.sbt` 예제입니다.

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

    Compile / console / scalacOptions --= Seq(
      "-Xfatal-warnings"
    ),

    Compile / run / fork := true
  )
```

### 중요한 점

**v1.15.0에서는 compiler plugin도 사용하는 것을 권장합니다.**

실제 v1.15.0 환경에서 이름 안정성까지 고려한다면 다음 dependency를 추가하는 것이 좋습니다. v1.15.0에서 `spinalhdl-idsl-plugin`이 제공되며, plugin이 없으면 생성된 이름이 예상과 다르게 mangling될 수 있습니다. [Hivebook](https://hivebook.wiki/wiki/spinalhdl-scala-hdl-generating-vhdl-and-verilog?utm_source=chatgpt.com)

```scala
Compile / compilerPlugin(
  "com.github.spinalhdl" %% "spinalhdl-idsl-plugin" % spinalVersion
)
```

따라서 최종적으로는:

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

# 3. 프로젝트 5 — Timer

Timer는 UART를 이해하기 전에 반드시 만들어 보는 것을 권장합니다.

Timer를 통해 다음을 한꺼번에 연습할 수 있습니다.

- `Reg`
- `UInt`
- Counter
- Clock
- Reset
- Enable
- 비교 연산
- Terminal count
- Pulse 생성

---

# 4. Timer의 기본 구조

이번에 만들 Timer는 다음과 같이 동작하도록 하겠습니다.

```text
                ┌──────────────────┐
                │      TIMER       │
                │                  │
 enable ───────►│                  │
 clear  ───────►│     Counter      ├──────► value
 limit  ───────►│        │         │
                │        ▼         │
                │   counter==limit │
                │        │         │
                └────────┼─────────┘
                         │
                         ▼
                        tick
```

동작 규칙:

1. `clear = 1`이면 counter를 0으로 만든다.
2. `enable = 1`이면 counter가 증가한다.
3. counter가 `limit`에 도달하면 `tick`을 1-cycle pulse로 발생시킨다.
4. tick이 발생하면 counter를 다시 0으로 만든다.

즉:

```text
counter

0
1
2
3
4
5  ← limit
│
└── tick = 1
    counter = 0
```

---

# 5. Timer SpinalHDL 코드

## `Timer.scala`

```scala
import spinal.core._

case class Timer(width: Int) extends Component {

  val io = new Bundle {
    val enable = in Bool()
    val clear  = in Bool()
    val limit  = in UInt(width bits)

    val tick   = out Bool()
    val value  = out UInt(width bits)
  }

  // Timer counter register
  val counter = Reg(UInt(width bits)) init(0)

  // Default value
  io.tick := False

  // Clear has highest priority
  when(io.clear) {

    counter := 0

  } otherwise {

    when(io.enable) {

      when(counter === io.limit) {

        counter := 0
        io.tick  := True

      } otherwise {

        counter := counter + 1

      }
    }
  }

  io.value := counter
}
```

---

# 6. Timer 코드 한 줄씩 이해하기

## 6.1 Component

```scala
case class Timer(width: Int) extends Component
```

여기서 `width`는 Timer counter의 비트 폭입니다.

예:

```scala
Timer(8)
```

이면:

```text
8-bit Timer
```

가 됩니다.

---

## 6.2 IO

```scala
val io = new Bundle {
  val enable = in Bool()
  val clear  = in Bool()
  val limit  = in UInt(width bits)

  val tick   = out Bool()
  val value  = out UInt(width bits)
}
```

회로 관점에서 보면:

```text
                 Timer
          ┌─────────────────┐
 enable ─►│                 │
 clear  ─►│                 │
 limit  ─►│                 │
          │                 │
          │    Counter      ├──► value
          │                 │
          │                 ├──► tick
          └─────────────────┘
```

---

# 7. Timer에서 가장 중요한 부분

```scala
val counter = Reg(UInt(width bits)) init(0)
```

이것이 실제 저장소입니다.

즉 단순한 Scala 변수:

```scala
var counter = 0
```

가 아닙니다.

실제 hardware register가 만들어집니다.

```text
             ┌──────────────┐
             │              │
             │   Counter    │
             │   Register   │
             │              │
             └──────┬───────┘
                    │
                    ▼
                 value
```

---

# 8. Timer의 terminal count

핵심은:

```scala
when(counter === io.limit) {

  counter := 0
  io.tick  := True

}
```

입니다.

예를 들어:

```text
limit = 4
```

이면:

```text
Clock     counter     tick
────────────────────────────
  0          0          0
  1          1          0
  2          2          0
  3          3          0
  4          4          1
  5          0          0
  6          1          0
```

이것이 UART에서 매우 중요한 개념이 됩니다.

UART에서는 이와 같은 counter를 이용하여:

> **"현재 clock이 UART 한 bit 시간이 되었는가?"**

를 판단하기 때문입니다.

---

# 9. Timer Simulation

## `TimerSim.scala`

```scala
import spinal.core._
import spinal.core.sim._

object TimerSim extends App {

  SimConfig
    .withWave
    .doSim(Timer(8)) { dut =>

      dut.io.enable #= false
      dut.io.clear  #= true
      dut.io.limit  #= 4

      dut.clockDomain.forkStimulus(10)

      // Reset-like clear
      dut.clockDomain.waitSampling(2)

      dut.io.clear #= false
      dut.io.enable #= true

      for (_ <- 0 until 15) {

        dut.clockDomain.waitSampling()

        println(
          s"value = ${dut.io.value.toInt}, tick = ${dut.io.tick.toBoolean}"
        )
      }
    }
}
```

---

# 10. Timer에서 반드시 확인해야 할 것

Simulation waveform을 열어 다음을 확인합니다.

```text
clock
   ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
───┘ └─┘ └─┘ └─┘ └─┘ └─┘

value
 0   1   2   3   4   0   1

tick
 0   0   0   0   1   0   0
```

**Timer 프로젝트에서 가장 중요한 학습 포인트는 `Reg + Counter + Compare + Pulse`입니다.**

---

# 11. 프로젝트 6 — UART TX

이제 Timer에서 배운 counter를 UART에 적용합니다.

UART TX는 훨씬 좋은 학습 예제입니다.

왜냐하면 다음 개념이 동시에 들어가기 때문입니다.

```text
Register
Counter
FSM
Shift Register
Baud Rate
Bit Timing
Serial Protocol
```

---

# 12. UART TX 기본 원리

이번 예제는 가장 기본적인:

> **115200 baud, 8 data bits, no parity, 1 stop bit**

UART를 대상으로 합니다.

이를 흔히:

```text
115200-N-8-1
```

이라고 합니다.

UART frame은:

```text
Idle   Start   D0 D1 D2 D3 D4 D5 D6 D7   Stop
 1       0      │                 │        1
```

입니다.

UART는 **LSB first**입니다.

예를 들어:

```text
0x41 = 0100_0001
```

을 전송하면:

```text
D0 D1 D2 D3 D4 D5 D6 D7
 1  0  0  0  0  0  1  0
```

순서로 전송합니다.

---

# 13. UART TX의 FSM

이번에는 다음 FSM을 사용하겠습니다.

```text
               start
 IDLE ──────────────────► START
  ▲                         │
  │                         ▼
  │                       DATA
  │                         │
  │                  8 bits transmitted
  │                         │
  │                         ▼
  └────────────────────── STOP
```

상태:

```text
IDLE
START
DATA
STOP
```

---

# 14. UART TX interface

우리가 만들 TX의 interface는 단순하게 하겠습니다.

```text
             UartTx
       ┌─────────────────┐
 data ─►                 │
 valid ─►                │
       │                 ├──► tx
 busy ◄─                 │
       └─────────────────┘
```

### 입력

```text
data
valid
```

### 출력

```text
tx
busy
```

`valid`가 한 cycle 동안 올라가면 `data`를 전송하기 시작합니다.

---

# 15. UART TX 코드

## `UartTx.scala`

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

  // Number of system-clock cycles per UART bit
  val cyclesPerBit = clockFrequencyHz / baudRate

  require(
    cyclesPerBit >= 2,
    "clockFrequencyHz / baudRate must be at least 2"
  )

  val timerWidth =
    log2Up(cyclesPerBit)

  val bitTimer = Reg(UInt(timerWidth bits)) init(0)

  val bitCounter = Reg(UInt(3 bits)) init(0)

  val shiftReg = Reg(Bits(8 bits)) init(0)

  object State extends SpinalEnum {
    val IDLE, START, DATA, STOP = newElement()
  }

  val state = RegInit(State.IDLE)

  io.tx   := True
  io.busy := state =/= State.IDLE

  // Default timer behavior
  when(bitTimer === cyclesPerBit - 1) {
    bitTimer := 0
  } otherwise {
    bitTimer := bitTimer + 1
  }

  switch(state) {

    // ------------------------------------------------
    // IDLE
    // ------------------------------------------------

    is(State.IDLE) {

      bitTimer   := 0
      bitCounter := 0

      when(io.valid) {

        shiftReg := io.data
        state := State.START

      }
    }

    // ------------------------------------------------
    // START BIT
    // ------------------------------------------------

    is(State.START) {

      io.tx := False

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0
        bitCounter := 0
        state := State.DATA

      }
    }

    // ------------------------------------------------
    // DATA BITS
    // ------------------------------------------------

    is(State.DATA) {

      io.tx := shiftReg(0)

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        shiftReg := shiftReg |>> 1

        when(bitCounter === 7) {

          state := State.STOP

        } otherwise {

          bitCounter := bitCounter + 1

        }
      }
    }

    // ------------------------------------------------
    // STOP BIT
    // ------------------------------------------------

    is(State.STOP) {

      io.tx := True

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0
        state := State.IDLE

      }
    }
  }
}
```

---

# 16. UART TX에서 중요한 부분

## Baud rate 계산

예를 들어:

```scala
clockFrequencyHz = 100_000_000
baudRate = 115_200
```

이면:

```text
100,000,000 / 115,200
≈ 868
```

즉 약 868개의 system clock마다 UART bit 하나를 전송합니다.

```text
System clock

|_|-|_|-|_|-|_|-|_|-|_|-|_|-|

<-------- 868 clocks -------->

UART bit
______________________________
```

---

# 17. TX FSM의 실제 동작

예를 들어:

```text
data = 0x41
```

이면:

```text
IDLE
 │
 │ valid
 ▼
START
 │
 │ 1 bit
 ▼
DATA
 │
 ├── D0 = 1
 ├── D1 = 0
 ├── D2 = 0
 ├── D3 = 0
 ├── D4 = 0
 ├── D5 = 0
 ├── D6 = 1
 └── D7 = 0
 │
 ▼
STOP
 │
 │ 1 bit
 ▼
IDLE
```

---

# 18. UART TX Simulation

## `UartTxSim.scala`

```scala
import spinal.core._
import spinal.core.sim._

object UartTxSim extends App {

  val clockFrequency = 10_000_000
  val baudRate = 1_000_000

  SimConfig
    .withWave
    .doSim(
      UartTx(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      dut.io.data #= 0
      dut.io.valid #= false

      dut.clockDomain.forkStimulus(10)

      // Initial cycles
      dut.clockDomain.waitSampling(5)

      // Send ASCII 'A'
      dut.io.data #= 0x41
      dut.io.valid #= true

      dut.clockDomain.waitSampling()

      dut.io.valid #= false

      // Wait until transmission is complete
      while (dut.io.busy.toBoolean) {
        dut.clockDomain.waitSampling()
      }

      dut.clockDomain.waitSampling(5)

      println("UART TX simulation finished")
    }
}
```

---

# 19. UART TX에서 꼭 Waveform을 보라

이번 프로젝트에서는 다음 waveform을 직접 확인해야 합니다.

```text
tx

IDLE
───────────────────┐
                   │
START              │
                   └─────────
                    0

DATA

D0    D1    D2    D3    D4    D5    D6    D7

 1     0     0     0     0     0     1     0

STOP
────────────────────────────────────────────── 1
```

실제 UART는 다음과 같이 보입니다.

```text
Idle
───────
       \
        \____
             \________________
                              \
                               \________________
```

정확한 bit width가 가장 중요합니다.

---

# 20. 프로젝트 7 — UART RX

UART RX는 TX보다 어렵습니다.

TX는:

> "내가 언제 bit를 보낼지 결정"

하면 됩니다.

RX는:

> "상대방이 언제 bit를 보내기 시작했는지 알아내고 정확한 위치에서 샘플링"

해야 합니다.

그래서 UART RX에서 **sampling**이 핵심입니다.

공식 `UartCtrlRx` 역시 sampling tick과 bit timer를 사용하고, RXD의 sampling window와 majority vote를 사용합니다. 
[SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/Intermediates%20ones/uart.html)

---

# 21. UART RX 구조

```text
              UART RX
                 │
                 ▼
             Synchronizer
                 │
                 ▼
          Start Detection
                 │
                 ▼
          Bit Sampling
                 │
                 ▼
          Shift Register
                 │
                 ▼
             DATA OUT
```

---

# 22. 왜 1-bit 중앙에서 샘플링해야 하는가?

UART frame:

```text
Start       D0          D1
   │          │           │
   ▼          ▼           ▼
 ___          ___________
    |________|
         ^
       sample
```

bit의 경계에서 읽으면 timing error에 취약합니다.

따라서:

```text
START BIT
┌────────────────────┐
│                    │
└────────────────────┘
         ▲
         │
       middle
```

중앙에서 샘플링하는 것이 좋습니다.

---

# 23. UART RX 학습용 구현

이번에는 이해를 위해 **oversampling 없이 정확히 bit 중앙에서 샘플링하는 단순 버전**을 만들겠습니다.

이것은 실제 상용 UART IP와 동일한 수준의 robustness를 목표로 하는 것이 아니라:

> **UART RX의 핵심 원리를 학습하기 위한 RTL**

입니다.

실제 SpinalHDL `UartCtrlRx`는 sampling window와 majority vote 등을 사용하므로 훨씬 견고합니다. 
[SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Com/uart.html?utm_source=chatgpt.com)

---

# 24. UART RX 코드

## `UartRx.scala`

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

    // ----------------------------------------------
    // IDLE
    // ----------------------------------------------

    is(State.IDLE) {

      bitTimer := 0
      bitCounter := 0

      // UART start bit is LOW
      when(!io.rx) {

        bitTimer := 0
        state := State.START

      }
    }

    // ----------------------------------------------
    // START BIT
    // ----------------------------------------------

    is(State.START) {

      when(bitTimer === halfBitCount - 1) {

        bitTimer := 0

        // Confirm that start bit is still LOW
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

    // ----------------------------------------------
    // DATA BITS
    // ----------------------------------------------

    is(State.DATA) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        // Sample one data bit
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

    // ----------------------------------------------
    // STOP BIT
    // ----------------------------------------------

    is(State.STOP) {

      when(bitTimer === cyclesPerBit - 1) {

        bitTimer := 0

        // Stop bit should be HIGH
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

---

# 25. RX 코드에서 가장 중요한 부분

## Start detection

```scala
when(!io.rx) {
    state := State.START
}
```

UART의 idle은:

```text
1
```

입니다.

Start bit가:

```text
0
```

으로 떨어지는 순간을 찾습니다.

---

# 26. Start bit 검증

그런데 단순히 LOW가 되었다고 바로 데이터 수신을 시작하면 위험합니다.

그래서:

```scala
when(bitTimer === halfBitCount - 1) {
```

에서 다시 확인합니다.

```scala
when(!io.rx) {
    state := State.DATA
}
```

즉:

```text
RX

1 1 1 1 1 1 1
          \
           \ Start
            \
             0
             │
             │ half bit
             ▼
             ●  ← 다시 확인
```

여기서 LOW라면 진짜 start bit라고 판단합니다.

---

# 27. Data sampling

데이터 영역에서는:

```scala
shiftReg(bitCounter) := io.rx
```

를 사용합니다.

예를 들어:

```text
bitCounter = 0
```

이면:

```text
shiftReg(0) = rx
```

입니다.

다음에는:

```text
bitCounter = 1

shiftReg(1) = rx
```

가 됩니다.

결과:

```text
shiftReg

[7] [6] [5] [4] [3] [2] [1] [0]
 D7  D6  D5  D4  D3  D2  D1  D0
```

---

# 28. UART RX Simulation

## `UartRxSim.scala`

RX simulation은 TX simulation보다 조금 복잡합니다.

우리가 UART 신호 자체를 만들어 주어야 하기 때문입니다.

```scala
import spinal.core._
import spinal.core.sim._

object UartRxSim extends App {

  val clockFrequency = 10_000_000
  val baudRate = 1_000_000

  val cyclesPerBit =
    clockFrequency / baudRate

  SimConfig
    .withWave
    .doSim(
      UartRx(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      dut.io.rx #= true

      dut.clockDomain.forkStimulus(10)

      def waitBit(): Unit = {
        dut.clockDomain.waitSampling(cyclesPerBit)
      }

      // ------------------------------------------
      // Idle
      // ------------------------------------------

      dut.io.rx #= true

      dut.clockDomain.waitSampling(cyclesPerBit)

      // ------------------------------------------
      // Start bit
      // ------------------------------------------

      dut.io.rx #= false
      waitBit()

      // ------------------------------------------
      // Data = 0x41
      // 0x41 = 0100_0001
      // LSB first
      // ------------------------------------------

      dut.io.rx #= true   // D0
      waitBit()

      dut.io.rx #= false  // D1
      waitBit()

      dut.io.rx #= false  // D2
      waitBit()

      dut.io.rx #= false  // D3
      waitBit()

      dut.io.rx #= false  // D4
      waitBit()

      dut.io.rx #= false  // D5
      waitBit()

      dut.io.rx #= true   // D6
      waitBit()

      dut.io.rx #= false  // D7
      waitBit()

      // ------------------------------------------
      // Stop bit
      // ------------------------------------------

      dut.io.rx #= true
      waitBit()

      // Give simulation a few cycles
      dut.clockDomain.waitSampling(5)

      println(
        f"Received = 0x${dut.io.data.toBigInt}%02X"
      )
    }
}
```

---

# 29. 그런데 위 RX 예제에는 중요한 교육적 문제가 있다

위 코드는 UART RX의 기본 원리를 설명하기에는 좋지만, **실제 UART RX로 사용하기에는 부족합니다.**

왜냐하면 실제 통신에서는:

```text
TX clock
RX clock
```

가 정확히 동일하지 않을 수 있기 때문입니다.

예:

```text
TX baud = 115200
RX baud = 115000
```

처럼 차이가 있을 수 있습니다.

따라서 실제 UART RX에서는:

```text
Oversampling
     ↓
Sampling Window
     ↓
Majority Vote
     ↓
Bit Center Tracking
```

등을 사용하는 것이 훨씬 좋습니다.

공식 SpinalHDL `UartCtrl`도 RXD를 sampling window로 여러 번 샘플링하고 majority vote를 이용하여 필터링하는 구조를 사용합니다. 
[SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Com/uart.html?utm_source=chatgpt.com)

---

# 30. 실제 프로젝트에서는 `UartCtrl`도 알아야 한다

SpinalHDL에는 UART를 처음부터 직접 구현하지 않아도 사용할 수 있는:

```scala
UartCtrl
```

가 `spinal.lib`에 제공됩니다.

공식 문서의 구조는 다음과 같습니다.

```text
                  UartCtrl
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
 Clock Divider    TX Controller  RX Controller
                                │
                                ▼
                         Sampling / Vote
```

그리고 interface는:

```text
config
write : Stream[Bits]
read  : Flow[Bits]
uart  : Uart
```

형태입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Com/uart.html?utm_source=chatgpt.com)

---

# 31. `UartCtrl`을 이용한 TX/RX

실제 프로젝트에서는 다음처럼 훨씬 간단하게 사용할 수 있습니다.

```scala
import spinal.core._
import spinal.lib._

case class UartExample() extends Component {

  val io = new Bundle {
    val uart = master(Uart())
  }

  val uartCtrl = UartCtrl(
    config = UartCtrlInitConfig(
      baudrate = 115200,
      dataLength = 8,
      parity = UartParityType.NONE,
      stop = UartStopType.ONE
    )
  )

  io.uart <> uartCtrl.io.uart

  // No TX data for now
  uartCtrl.io.write.valid := False
  uartCtrl.io.write.payload := B(0)
}
```

공식 문서에서도 `UartCtrlInitConfig`를 이용해 baud rate, data length, parity, stop bit 등을 설정하는 방식을 제공합니다. 
[SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/Intermediates%20ones/uart.html?utm_source=chatgpt.com)

---

# 32. 직접 만든 UART TX/RX와 `UartCtrl`의 차이

학습에서는 **둘 다 알아야 합니다.**

| 학습용 직접 구현 | `UartCtrl` |
|---|---|
| FSM 이해 | 실제 reusable IP |
| Bit timing 이해 | Baud configuration |
| Shift register 이해 | Stream interface |
| UART protocol 이해 | Sampling |
| RX 중앙 sampling 이해 | Majority vote |
| RTL 설계 연습 | 실전 프로젝트 |
| 초보 학습에 좋음 | 실제 시스템에 좋음 |

즉:

> **처음에는 직접 구현하고, 그 다음 `UartCtrl`을 사용하는 것을 추천합니다.**

---

# 33. 세 프로젝트가 어떻게 연결되는가?

이번 세 프로젝트는 사실 서로 독립된 것이 아닙니다.

## Timer

먼저:

```text
Clock
 ↓
Counter
 ↓
Compare
 ↓
Tick
```

을 배웁니다.

↓

## UART TX

Timer 개념을:

```text
UART Bit Timer
```

로 발전시킵니다.

```text
Clock
 ↓
Bit Counter
 ↓
FSM
 ↓
TX
```

↓

## UART RX

TX에서 배운 bit timing을 역으로 사용합니다.

```text
RX
 ↓
Start Detection
 ↓
Half-bit Timer
 ↓
Bit Timer
 ↓
Sampling
 ↓
Shift Register
 ↓
Valid
```

즉 학습 구조가:

```text
Timer
  │
  ▼
Bit Timing
  │
  ▼
UART TX
  │
  ▼
UART RX
  │
  ▼
UART Controller
```

로 자연스럽게 연결됩니다.

---

# 34. 세 프로젝트에서 반드시 익혀야 할 SpinalHDL 문법

이번 세 가지 프로젝트를 공부할 때 다음 문법에 집중하십시오.

## Timer

```scala
Reg
init
when
otherwise
UInt
===
+
```

## UART TX

추가로:

```scala
SpinalEnum
switch
is
Bits
bit indexing
|>>
```

## UART RX

추가로:

```scala
shift register
FSM
sampling
counter
valid pulse
```

---

# 35. 특히 `SpinalEnum`을 잘 이해해야 한다

UART TX/RX에서:

```scala
object State extends SpinalEnum {
  val IDLE, START, DATA, STOP = newElement()
}
```

를 사용했습니다.

이것은 FSM 상태를 표현합니다.

즉:

```text
IDLE  = 00
START = 01
DATA  = 10
STOP  = 11
```

과 같은 상태 register가 하드웨어로 생성됩니다.

그리고:

```scala
switch(state) {

  is(State.IDLE) {
    ...
  }

  is(State.DATA) {
    ...
  }
}
```

는 해당 상태에 따라 다른 조합논리를 만드는 것입니다.


---

# 36. 세 프로젝트를 통해 얻는 RTL 설계 능력

이 세 가지를 제대로 구현하면 다음 개념을 상당히 확실하게 이해하게 됩니다.

```text
                 SpinalHDL
                     │
         ┌───────────┼───────────┐
         │           │           │
       Data        Control      Timing
         │           │           │
       UInt         FSM        Counter
       Bits         when       Reg
       Shift        switch      Compare
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
                   UART
```

이 정도가 되면 다음 프로젝트인 **FIFO**를 배우기가 훨씬 쉬워집니다.

---

# 37. 추천하는 실습 순서

제가 권장하는 순서는 다음입니다.

### STEP 1

먼저 Timer만 구현합니다.

```text
Timer.scala
TimerSim.scala
```

그리고 waveform을 확인합니다.

---

### STEP 2

Timer를 수정하여:

```text
enable
clear
limit
tick
```

을 직접 바꿔 봅니다.

---

### STEP 3

UART TX를 만듭니다.

```text
UartTx.scala
UartTxSim.scala
```

그리고 `0x41`, `0x55`, `0xAA`, `0xFF`를 각각 보내 보십시오.

특히:

```text
0x00
0xFF
0x55
0xAA
0x41
```

를 추천합니다.

`0x55 = 01010101`, `0xAA = 10101010`은 waveform에서 bit transition을 확인하기 매우 좋습니다.

---

### STEP 4

UART RX를 만듭니다.

먼저:

```text
0x41
```

하나만 제대로 받도록 합니다.

그 다음:

```text
0x55
0xAA
0xFF
0x00
```

을 테스트합니다.

---

### STEP 5

마지막으로 TX와 RX를 연결합니다.

```text
          ┌───────────┐
          │  UART TX  │
          └─────┬─────┘
                │
                │ tx
                ▼
          ┌───────────┐
          │  UART RX  │
          └─────┬─────┘
                │
                ▼
              data
```

즉:

```text
TX → RX loopback
```

을 구현합니다.

이것이 **UART 학습의 매우 중요한 단계**입니다.

---

# 38. 최종적으로 만들 것을 추천하는 UART Loopback

최종 실습에서는:

```scala
tx.io.tx <> rx.io.rx
```

와 같은 구조를 만들어:

```text
                ┌───────────┐
 data ─────────►│ UART TX   │
                └─────┬─────┘
                      │
                      │ serial
                      │
                ┌─────▼─────┐
                │ UART RX   │
                └─────┬─────┘
                      │
                      ▼
                   received
```

로 테스트합니다.

그러면:

```text
0x41
```

를 넣었을 때:

```text
TX
 ↓
serial waveform
 ↓
RX
 ↓
0x41
```

가 되는 것을 확인할 수 있습니다.

---

## 중요한 버전 관련 참고

이번 답변의 **직접 작성한 Timer/TX/RX 코드는 v1.15.0의 기본 SpinalHDL 문법과 API를 기준으로 작성**했습니다. 다만 공식 `UartCtrl`의 내부 구현은 직접 만든 학습용 UART와 목적이 다릅니다. 공식 구현은 `Stream`/`Flow`, `Uart`, sampling tick, RX sampling window와 majority vote 등을 사용하므로 실제 UART IP에 가까운 구조입니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Libraries/Com/uart.html?utm_source=chatgpt.com)

또한 공식 SpinalHDL 문서의 Timer 예제에서도 `Reg(UInt(...))`, `when`, `limit`, `full` 등을 이용하는 동일한 RTL 설계 패턴을 보여줍니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/v1.6.0/SpinalHDL/Examples/Advanced%20ones/timer.html?utm_source=chatgpt.com)

**다음 학습 단계에서는 이 세 예제를 단순히 복사해서 사용하는 것보다, 제가 각 코드를 한 줄씩 분해하여 `Scala 코드 → SpinalHDL 하드웨어 의미 → 생성되는 Verilog → 실제 waveform`의 4단계로 설명하는 방식이 가장 효과적입니다.** 특히 UART TX/RX는 FSM과 clock-cycle 계산을 정확히 이해하는 것이 핵심입니다.


---

---

**SpinalHDL을 처음 배우는 사람의 관점**에서, 단순히 문법을 나열하기보다 **“디지털 회로를 어떻게 생각하고 → SpinalHDL로 표현하고 → RTL로 생성하고 → 시뮬레이션하고 → 검증하는가”**라는 흐름으로 정리해 보겠습니다.

현재 공식 문서를 기준으로 보면 SpinalHDL은 **Scala 위에 구현된 하드웨어 기술 라이브러리/DSL**이며, 일반적인 VHDL/Verilog처럼 RTL을 기술하지만 Scala의 객체지향·함수형·제네릭 기능을 이용해 하드웨어를 재사용 가능하게 만드는 것이 핵심입니다. 생성 결과는 VHDL, Verilog, SystemVerilog이며 기존 EDA 도구와도 연동할 수 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/index.html?utm_source=chatgpt.com)

---

# 1. 먼저 SpinalHDL을 어떻게 이해해야 하는가?

SpinalHDL을 처음 보면 가장 혼란스러운 부분이 이것입니다.

> **“Scala 프로그램을 작성하는 것인가? 아니면 HDL을 작성하는 것인가?”**

정답은 **둘 다 맞지만, 목적은 하드웨어 설계**입니다.

개념적으로 다음과 같이 생각하면 쉽습니다.

```text
Scala 프로그램
      │
      │  SpinalHDL API 사용
      ▼
하드웨어 구조를 만드는 Scala 코드
      │
      │  Elaboration
      ▼
Hardware Netlist / 회로 구조
      │
      │  Code Generation
      ▼
Verilog / SystemVerilog / VHDL
      │
      ├── FPGA Tool
      ├── ASIC Tool
      ├── Verilator
      └── 기타 EDA Tool
```

즉, **Scala가 실행되면서 실제 디지털 회로의 구조를 만들어 내고**, 그 결과를 RTL로 출력합니다. 공식 문서에서도 SpinalHDL 코드를 실행하면 내부적으로 하드웨어 netlist를 나타내는 그래프를 만들고, elaboration 이후 이를 VHDL/Verilog로 출력한다고 설명합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Getting%20Started/Scala%20Guide/interaction.html?utm_source=chatgpt.com)

따라서 SpinalHDL을 배울 때 가장 중요한 사고방식은:

> **“Scala로 하드웨어를 생성한다.”**

입니다.

---

# 2. SpinalHDL은 HLS와 다르다

이 부분은 아주 중요합니다.

SpinalHDL은

> “C/C++ 같은 알고리즘을 입력하면 알아서 하드웨어를 만들어 주는 HLS”

가 아닙니다.

SpinalHDL은 기본적으로 **RTL 설계 방식**을 사용합니다.

예를 들어 AND 게이트를 만든다면:

```scala
io.c := io.a & io.b
```

이것은 단순히 Scala에서 `&` 연산을 수행하는 프로그램이 아닙니다.

실제로는 다음 회로를 의미합니다.

```text
a ─────┐
       AND ─── c
b ─────┘
```

공식 문서도 SpinalHDL의 목적을 추상적인 알고리즘을 자동으로 플립플롭/게이트로 변환하는 HLS가 아니라, **flip-flop, gate, if/case 등의 하드웨어 요소를 이용하여 재사용 가능한 추상화 계층을 만드는 것**으로 설명합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

---

# 3. SpinalHDL을 배우기 전에 알아야 할 디지털 논리

SpinalHDL 자체보다 먼저 다음 개념을 알고 있으면 학습 속도가 매우 빨라집니다.

## 3.1 조합논리

출력이 현재 입력에만 의존합니다.

```text
Inputs
  │
  ▼
Combinational Logic
  │
  ▼
Outputs
```

예:

- AND
- OR
- XOR
- MUX
- Decoder
- Encoder
- Adder
- Comparator

SpinalHDL에서는 이런 회로를 매우 직접적으로 표현할 수 있습니다.

```scala
io.y := io.a & io.b
```

또는

```scala
io.y := io.a | io.b
```

---

# 4. 순차논리

순차논리는 **상태(state)**를 갖습니다.

대표적으로:

- Register
- Counter
- FSM
- FIFO
- Pipeline
- CPU

등이 있습니다.

기본 구조는:

```text
             ┌─────────────┐
input ──────►│ Combinational│
             │    Logic     │
             └──────┬──────┘
                    │
                    ▼
                ┌───────┐
clock ────────► │  Reg  │
                └───┬───┘
                    │
                    ▼
                  output
```

SpinalHDL에서 중요한 것은 `Reg`입니다.

```scala
val counter = Reg(UInt(8 bits))
```

이 한 줄은 단순한 Scala 변수 선언과 다릅니다.

**하드웨어 레지스터를 선언하는 것**입니다.

---

# 5. SpinalHDL의 가장 중요한 개념: `Component`

SpinalHDL 회로 설계의 기본 단위는 `Component`입니다.

예를 들어 AND gate:

```scala
import spinal.core._

class AndGate extends Component {
  val io = new Bundle {
    val a = in Bool
    val b = in Bool
    val y = out Bool
  }

  io.y := io.a & io.b
}
```

이것을 읽는 방법은 다음과 같습니다.

```scala
class AndGate extends Component
```

→ `AndGate`라는 하드웨어 모듈을 만든다.

```scala
val io = new Bundle
```

→ 외부와 연결되는 인터페이스를 만든다.

```scala
val a = in Bool
```

→ 1비트 입력

```scala
val y = out Bool
```

→ 1비트 출력

```scala
io.y := io.a & io.b
```

→ AND 게이트를 연결

공식 getting-started 예제도 거의 동일한 구조의 AND gate부터 시작합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/v1.3.8/SpinalHDL/Getting%20Started/getting_started.html?utm_source=chatgpt.com)

---

# 6. `io`는 무엇인가?

초보자가 매우 자주 사용하는 것이:

```scala
val io = new Bundle {
    ...
}
```

입니다.

`io`는 해당 Component의 **외부 인터페이스**라고 생각하면 됩니다.

예를 들어:

```scala
val io = new Bundle {
  val a = in Bool
  val b = in Bool
  val y = out Bool
}
```

회로로 표현하면:

```text
             AndGate
        ┌───────────────┐
a ─────►│               │
b ─────►│     AND       ├────► y
        │               │
        └───────────────┘
```

---

# 7. 가장 먼저 익혀야 할 데이터 타입

SpinalHDL에서는 다음 타입들이 핵심입니다.

| 타입 | 의미 |
|---|---|
| `Bool` | 1-bit |
| `Bits` | unsigned bit vector |
| `UInt` | unsigned integer |
| `SInt` | signed integer |
| `Bundle` | 여러 신호를 하나의 구조체로 묶음 |
| `Vec` | 동일한 타입의 여러 요소 |

가장 중요한 것은:

```scala
Bool
Bits
UInt
SInt
```

입니다.

---

# 8. `Bool`

가장 간단합니다.

```scala
val a = in Bool
val b = in Bool
val y = out Bool

y := a & b
```

논리 회로의 0/1 신호를 표현합니다.

주요 연산:

```scala
a & b
a | b
a ^ b
!a
```

즉:

```text
& → AND
| → OR
^ → XOR
! → NOT
```

---

# 9. `Bits`

여러 개의 비트를 단순한 bit vector로 다룹니다.

```scala
val a = in Bits(8 bits)
```

8비트 신호입니다.

```text
a[7:0]
```

와 비슷하게 생각하면 됩니다.

---

# 10. `UInt`

정수 연산을 위한 unsigned 데이터입니다.

```scala
val a = in UInt(8 bits)
val b = in UInt(8 bits)
val y = out UInt(8 bits)

y := a + b
```

즉:

```text
8-bit a
     +
8-bit b
     │
     ▼
8-bit y
```

---

# 11. `SInt`

signed integer가 필요하면:

```scala
val a = in SInt(8 bits)
```

를 사용합니다.

따라서 초보자는 다음 구분을 확실히 기억하면 좋습니다.

```text
Bool
 └─ 1 bit

Bits
 └─ bit vector

UInt
 └─ unsigned integer

SInt
 └─ signed integer
```

---

# 12. SpinalHDL에서 `=`와 `:=`는 완전히 다르다

이것은 초보자가 반드시 익혀야 합니다.

Scala의 일반적인 변수 할당은:

```scala
=
```

하지만 SpinalHDL에서 하드웨어 신호를 연결할 때는:

```scala
:=
```

를 사용합니다.

예:

```scala
io.y := io.a & io.b
```

이것은

> "Scala 변수 y에 값을 대입한다"

가 아니라

> "`a AND b`의 하드웨어 결과를 `y`에 연결한다"

는 의미입니다.

이 차이를 이해하는 것이 SpinalHDL 학습의 핵심 중 하나입니다.

---

# 13. `Reg`는 반드시 이해해야 한다

다음 코드:

```scala
val reg = Reg(UInt(8 bits))
```

는 8비트 register입니다.

예를 들어:

```scala
val counter = Reg(UInt(8 bits))

counter := counter + 1
```

이면 개념적으로:

```text
           ┌──────────┐
           │          │
     ┌────►│   + 1    │
     │     │          │
     │     └────┬─────┘
     │          │
     │          ▼
     │       ┌─────┐
     └────────│ Reg │
             └─────┘
                 │
                 ▼
              counter
```

즉, 현재 값을 저장하고 다음 clock에서 갱신하는 회로입니다.

---

# 14. Clock과 Reset

순차논리를 설계하려면 clock domain 개념을 알아야 합니다.

SpinalHDL에서는 clock/reset을 추상화하여 다룹니다.

예를 들어:

```scala
val counter = Reg(UInt(8 bits)) init(0)
```

와 같이 초기값을 설정할 수 있습니다.

더 복잡한 설계에서는:

- `ClockDomain`
- reset
- synchronous reset
- asynchronous reset
- clock crossing

등을 이해해야 합니다.

공식 문서는 SpinalHDL의 중요한 장점 중 하나로 **clock-domain 관련 설계 오류를 조기에 검사할 수 있는 기능**도 언급합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

---

# 15. `when`은 조건부 하드웨어

SpinalHDL에서 조건문은 매우 중요합니다.

```scala
when(io.enable) {
  counter := counter + 1
}
```

이는 일반적인 소프트웨어 `if`와 비슷하게 보이지만 의미는:

> enable이 1일 때 register에 다음 값을 넣는 하드웨어

입니다.

즉:

```text
             enable
                │
                ▼
          ┌───────────┐
counter ─►│           │
           │   MUX     ├──► Reg
counter+1 ►│           │
          └───────────┘
```

라고 이해해야 합니다.

---

# 16. `otherwise`

예:

```scala
when(io.enable) {
  counter := counter + 1
} otherwise {
  counter := counter
}
```

다만 register의 경우 hold 동작이 기본적으로 자연스럽게 표현되므로 모든 경우에 `counter := counter`를 쓸 필요는 없습니다.

---

# 17. `switch` / `is`

FSM이나 opcode decoder 등을 만들 때 매우 중요합니다.

개념적으로:

```scala
switch(state) {
  is(State.IDLE) {
    ...
  }

  is(State.RUN) {
    ...
  }

  is(State.DONE) {
    ...
  }
}
```

이런 방식으로 상태 머신을 표현할 수 있습니다.

---

# 18. MUX를 이해하면 SpinalHDL이 쉬워진다

예를 들어:

```scala
when(sel) {
  y := a
} otherwise {
  y := b
}
```

이것은 사실상:

```text
       a ───────┐
                │
                ▼
              ┌─────┐
       b ────►│ MUX │────► y
              └─────┘
                ▲
                │
               sel
```

입니다.

즉 SpinalHDL의 조건문을 **소프트웨어 제어 흐름이 아니라 하드웨어 구조**로 읽는 습관이 중요합니다.

---

# 19. `Bundle`은 매우 중요하다

실제 프로젝트에서는 신호가 수십~수백 개가 됩니다.

예를 들어 CPU 버스:

```text
address
data
valid
ready
write
...
```

이것을 하나씩 전달하면 매우 복잡해집니다.

SpinalHDL에서는:

```scala
case class Bus() extends Bundle {
  val address = UInt(32 bits)
  val data    = Bits(32 bits)
  val valid   = Bool
  val ready   = Bool
}
```

처럼 묶을 수 있습니다.

그러면:

```scala
val bus = new Bus()
```

형태로 사용할 수 있습니다.

SpinalHDL이 VHDL/Verilog보다 특히 유리한 부분 중 하나가 **복잡한 bus와 interface를 추상화하여 연결하는 것**입니다. 공식 문서도 이러한 bus abstraction을 주요 장점으로 설명합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

---

# 20. `Vec`

동일한 신호가 여러 개 필요하면 `Vec`을 사용합니다.

예:

```scala
val regs = Vec(UInt(32 bits), 8)
```

개념적으로:

```text
regs[0]
regs[1]
regs[2]
...
regs[7]
```

와 같은 구조입니다.

이를 이용하면 register file, 여러 채널의 데이터, 병렬 처리 구조 등을 쉽게 만들 수 있습니다.

---

# 21. Component를 계층적으로 구성한다

실제 디지털 시스템에서는 모든 것을 하나의 Component에 작성하면 안 됩니다.

예를 들어 CPU를 만든다면:

```text
CPU
├── Fetch
├── Decode
├── Execute
├── RegisterFile
├── ALU
├── CSR
└── MemoryInterface
```

SpinalHDL에서는 이런 구조를 자연스럽게 만들 수 있습니다.

예:

```scala
class Alu extends Component {
  ...
}

class Cpu extends Component {
  val alu = new Alu
  ...
}
```

이것이 **hardware hierarchy**입니다.

---

# 22. SpinalHDL의 진짜 강점: Parameterization

SpinalHDL을 Verilog의 대체 문법 정도로만 사용하면 SpinalHDL의 장점을 충분히 활용하지 못합니다.

예를 들어 8-bit counter와 32-bit counter를 따로 작성하는 대신:

```scala
class Counter(width: Int) extends Component {
  val io = new Bundle {
    val enable = in Bool
    val value  = out UInt(width bits)
  }

  val counter = Reg(UInt(width bits)) init(0)

  when(io.enable) {
    counter := counter + 1
  }

  io.value := counter
}
```

그리고:

```scala
new Counter(8)
```

또는

```scala
new Counter(32)
```

로 사용할 수 있습니다.

이것이 **hardware generator**적인 사고방식입니다.

---

# 23. Scala의 반복문을 하드웨어 생성에 사용할 수 있다

예를 들어 8개의 동일한 구조를 생성한다고 해봅시다.

일반 HDL에서는 반복적인 코드를 많이 작성해야 할 수 있습니다.

SpinalHDL에서는 Scala의:

```scala
for (...)
```

등을 이용해 하드웨어 구조를 생성할 수 있습니다.

중요한 점:

> Scala의 `for`가 실행시간 하드웨어 loop가 되는 것이 아니다.

입니다.

**Elaboration 시점에 반복문이 실행되어 여러 개의 하드웨어를 생성하는 것**입니다.

이 구분은 SpinalHDL을 제대로 이해하기 위해 매우 중요합니다.

---

# 24. Elaboration이라는 개념

SpinalHDL 학습에서 반드시 등장하는 단어가:

> **Elaboration**

입니다.

쉽게 표현하면:

```text
Scala 코드
    │
    ▼
Component 생성
    │
    ▼
Parameter 결정
    │
    ▼
for / if / 함수 / 객체 등을 실행
    │
    ▼
하드웨어 구조 완성
    │
    ▼
RTL 생성
```

입니다.

따라서 SpinalHDL에는 크게 두 종류의 시간이 있다고 생각하면 편합니다.

### ① Elaboration time

Scala 프로그램이 실행되는 시간

### ② Hardware runtime

생성된 회로가 FPGA/ASIC에서 clock에 따라 동작하는 시간

예를 들어:

```scala
for(i <- 0 until 8) {
  ...
}
```

은 일반적으로 **elaboration time**에 실행됩니다.

반면:

```scala
counter := counter + 1
```

로 표현된 register 동작은 생성된 **하드웨어가 clock에 따라 실행**됩니다.

---

# 25. 이것이 SpinalHDL을 어렵게 만드는 핵심 이유

초보자가 가장 많이 혼동하는 부분입니다.

예를 들어:

```scala
val x = 10
```

은 Scala 값일 수 있습니다.

반면:

```scala
val x = UInt(8 bits)
```

은 하드웨어 신호입니다.

그리고:

```scala
val x = Reg(UInt(8 bits))
```

은 register입니다.

겉보기에는 모두 Scala 코드지만 **하드웨어 의미가 서로 다릅니다.**

따라서 항상 질문해야 합니다.

> "이 Scala 코드는 elaboration 때 실행되는가, 아니면 생성된 회로를 의미하는가?"

이 질문을 습관화하면 SpinalHDL 이해도가 급격히 올라갑니다.

---

# 26. Combinational과 Sequential을 구분하라

SpinalHDL 학습 초반에는 다음 표를 머릿속에 넣는 것이 좋습니다.

| 종류 | 특징 | 예 |
|---|---|---|
| Combinational | 상태 없음 | AND, MUX, Adder |
| Sequential | 상태 있음 | Register, Counter |
| FSM | 상태 + 조합논리 | CPU controller |
| Pipeline | 여러 단계 register | DSP/CPU |
| Memory | 저장공간 | RAM/ROM |

예:

```scala
io.y := io.a + io.b
```

→ 조합논리

반면:

```scala
val r = Reg(UInt(8 bits))
r := io.a
```

→ 순차논리

---

# 27. `Reg`를 기준으로 회로를 읽어라

실제 RTL 설계에서 아주 강력한 사고방식입니다.

코드를 봤을 때:

```scala
val r1 = Reg(...)
val r2 = Reg(...)
```

가 있다면:

```text
r1 → combinational logic → r2
```

와 같은 pipeline을 생각합니다.

예:

```scala
val r1 = Reg(UInt(8 bits))
val r2 = Reg(UInt(8 bits))

r1 := io.a + io.b
r2 := r1 + io.c
```

그러면:

```text
a ─┐
   ├─ ADD ──► r1 ──► ADD ──► r2
b ─┘                    ▲
                        │
                        c
```

즉 **2-stage pipeline**이 됩니다.

---

# 28. FSM 설계

SpinalHDL로 디지털 논리회로를 제대로 설계하려면 FSM을 반드시 익혀야 합니다.

대표적인 예:

```text
IDLE
 │
 │ start
 ▼
RUN
 │
 │ done
 ▼
DONE
 │
 ▼
IDLE
```

SpinalHDL에서는 `StateMachine` 관련 기능을 이용해 이를 표현할 수 있습니다.

개념적으로:

```scala
val fsm = new StateMachine {
  val idle = new State with EntryPoint
  val run  = new State
  val done = new State

  ...
}
```

정도의 형태로 발전합니다.

처음에는 FSM을 직접 상태 register + `switch`로 구현해 보고, 이후 SpinalHDL의 StateMachine abstraction을 배우는 것을 추천합니다.

---

# 29. SpinalHDL의 설계 철학

SpinalHDL을 배우면서 다음 철학을 잡는 것이 중요합니다.

### 기존 HDL

```text
회로를 표현하기 위해
많은 문법과 wiring을 직접 작성
```

### SpinalHDL

```text
회로의 구조를 표현하는 abstraction을 만들고
Scala의 기능을 이용해 반복/재사용
```

공식 문서에서 강조하는 주요 장점도 다음과 같습니다.

- 적은 wiring
- bus abstraction
- 코드 크기 감소
- 강력한 type system
- 조기 design checking
- clock-domain 관련 검사
- generic hardware 설계
- VHDL/Verilog와의 상호운용성 [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

---

# 30. Simulation은 반드시 같이 배워야 한다

SpinalHDL을 공부할 때:

> "코드 작성 → Verilog 생성"

까지만 하면 부족합니다.

반드시:

```text
Design
 ↓
Simulation
 ↓
Waveform
 ↓
Verification
```

까지 경험해야 합니다.

공식 환경에서는 Verilator와 GTKWave 같은 도구를 이용해 simulation과 waveform 확인을 할 수 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Getting%20Started/Install%20and%20setup.html?utm_source=chatgpt.com)

---

# 31. Simulation의 기본적인 사고방식

예를 들어 Counter를 만들었다면:

```text
reset
  ↓
counter = 0

clock
  ↓
counter = 1

clock
  ↓
counter = 2

clock
  ↓
counter = 3
```

이것을 testbench에서 확인합니다.

즉 단순히:

```text
"컴파일 성공"
```

이 아니라:

```text
"내가 의도한 clock-by-clock 동작을 하는가?"
```

를 검증해야 합니다.

---

# 32. SpinalHDL의 테스트 코드

SpinalHDL은 simulation API를 제공합니다.

개념적으로:

```scala
SimConfig
  .withWave
  .doSim(new Counter(8)) { dut =>
    ...
  }
```

와 같은 형태로 simulation을 구성할 수 있습니다.

초보자라면 처음부터 복잡한 verification framework를 공부할 필요는 없습니다.

먼저:

1. DUT 생성
2. 입력 설정
3. clock 생성
4. reset
5. 몇 cycle 진행
6. 출력 확인
7. waveform 확인

순서로 익히면 됩니다.

---

# 33. Waveform을 보는 능력이 매우 중요하다

디지털 설계에서는 코드를 보는 것보다 waveform을 보는 것이 더 중요할 때가 많습니다.

예:

```text
clk     ┌─┐ ┌─┐ ┌─┐ ┌─┐
        └─┘ └─┘ └─┘ └─┘

reset   ───────┐
               └────────

enable  ────┐       ┌────
            └───────┘

counter  0   1   2   3   4
```

이런 파형을 보고:

> "내 RTL이 의도한 하드웨어와 정확히 같은가?"

를 판단할 수 있어야 합니다.

---

# 34. SpinalHDL의 Design Error Checking

SpinalHDL의 중요한 장점 중 하나는 RTL 생성 과정에서 설계 오류를 발견하는 것입니다.

예를 들어:

- combinational loop
- latch 관련 문제
- signal 연결 문제
- width 문제
- clock-domain 문제

등을 가능한 한 조기에 발견할 수 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

이것은 SpinalHDL을 단순한 "Verilog 생성기"보다 훨씬 강력하게 만드는 요소입니다.

---

# 35. `Bits`, `UInt`, `SInt`의 폭(width)을 항상 의식하라

디지털 설계에서 매우 중요합니다.

예:

```scala
val a = UInt(8 bits)
val b = UInt(8 bits)
```

라고 해서 모든 연산 결과가 항상 원하는 폭을 자동으로 갖는다고 생각하면 안 됩니다.

특히:

```scala
a + b
```

같은 연산에서는 carry와 결과 폭을 의식해야 합니다.

예를 들어 8-bit + 8-bit는:

```text
255 + 1 = 256
```

이므로 결과를 8비트에 넣으면 overflow가 발생할 수 있습니다.

따라서 SpinalHDL을 배울 때 **bit width / resize / truncation / signedness**를 반드시 함께 공부해야 합니다.

---

# 36. Type system을 적극적으로 활용하라

SpinalHDL의 강점은 "타입이 많다"는 것이 아니라:

> **타입을 이용해서 하드웨어 실수를 일찍 발견할 수 있다**

는 것입니다.

예:

```scala
UInt
SInt
Bits
Bool
```

를 적절하게 사용하면 설계 의도를 코드에 반영할 수 있습니다.

따라서 모든 것을 `Bits`로 만들어놓고 나중에 해석하는 방식보다는:

```text
숫자 → UInt / SInt
bit field → Bits
조건 → Bool
interface → Bundle
```

처럼 사용하는 습관이 좋습니다.

---

# 37. `Bundle` + `IMasterSlave`

실제 프로젝트로 가면 interface abstraction이 중요해집니다.

예를 들어:

```text
valid
ready
payload
```

같은 handshake interface를 하나의 Bundle로 만들 수 있습니다.

그 위에 master/slave 방향성을 정의하면 복잡한 interface를 재사용하기 쉬워집니다.

이것이 나중에:

- AXI
- APB
- Avalon
- Stream
- FIFO interface
- CPU bus

등을 다룰 때 매우 중요해집니다.

---

# 38. `spinal.lib`를 반드시 공부하라

SpinalHDL은 크게:

```scala
import spinal.core._
import spinal.lib._
```

형태로 많이 사용합니다.

`spinal.core`는 기본적인 HDL 기능이고,

`spinal.lib`에는 다양한 reusable hardware component와 interface가 들어 있습니다.

공식 예제 역시 이 두 import를 기본으로 사용하는 경우가 많습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/index.html?utm_source=chatgpt.com)

---

# 39. 학습 순서는 이렇게 추천한다

처음부터 CPU나 AXI를 공부하지 않는 것이 좋습니다.

### Level 1 — 디지털 논리

먼저:

```text
AND
OR
XOR
NOT
MUX
Decoder
Adder
Comparator
```

---

### Level 2 — SpinalHDL 기본

다음:

```text
Component
Bundle
Bool
Bits
UInt
SInt
:=
when
otherwise
switch
```

---

### Level 3 — Sequential

그 다음:

```text
Reg
Clock
Reset
Counter
Register
Enable
```

---

### Level 4 — FSM

```text
State
StateMachine
Transition
EntryPoint
```

---

### Level 5 — 구조화

```text
Component hierarchy
Bundle
Vec
Parameter
Generic component
Scala function
Scala class
```

---

### Level 6 — Simulation

```text
SpinalSim
Testbench
Clock
Reset
assert
Waveform
GTKWave
Verilator
```

---

### Level 7 — 실제 hardware

```text
UART
FIFO
RAM
ROM
Timer
PWM
SPI
I2C
APB
AXI
```

---

### Level 8 — 고급 SpinalHDL

마지막으로:

```text
Hardware generator
Meta-hardware
Generic architecture
Pipeline abstraction
Stream
Flow
Bus abstraction
BlackBox
CDC
Formal verification
```

공식 예제도 대체로 **Simple → Intermediate → Advanced**로 나뉘며, simple 예제에는 carry adder, counter, ROM 등이 있고 intermediate 단계에는 UART, memory-mapped UART, timer 등이 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/index.html?utm_source=chatgpt.com)

---

# 40. 추천 실습 프로젝트

학습 효율을 높이려면 다음 순서로 직접 만들어 보세요.

## 프로젝트 1 — AND/OR/XOR

```text
입력 2개
출력 3개
```

SpinalHDL 기본 문법 학습.

---

## 프로젝트 2 — 4:1 MUX

```text
a
b
c
d
 │
 ▼
MUX
 │
 ▼
y
```

`when`, `switch` 연습.

---

## 프로젝트 3 — 8-bit Adder

```text
A ─┐
   ├── ADD ──► Y
B ─┘
```

`UInt`, width 학습.

---

## 프로젝트 4 — Counter

```text
enable
reset
clock
   │
   ▼
counter
```

`Reg` 학습.

---

## 프로젝트 5 — Timer

```text
clock
  ↓
counter
  ↓
compare
  ↓
interrupt
```

Sequential logic을 제대로 이해할 수 있습니다.

---

## 프로젝트 6 — UART TX

```text
CPU/Data
   │
   ▼
UART TX
   │
   ▼
serial output
```

FSM + counter + register + bit shifting을 모두 사용할 수 있습니다.

---

## 프로젝트 7 — UART RX

UART TX보다 어렵습니다.

```text
RX
 ↓
Sampling
 ↓
FSM
 ↓
Shift Register
 ↓
Data
```

---

## 프로젝트 8 — FIFO

여기서부터 실제 RTL 설계 느낌이 강해집니다.

```text
write
  ↓
 ┌──────────┐
 │   FIFO   │
 └──────────┘
            ↓
           read
```

---

## 프로젝트 9 — APB Peripheral

```text
CPU
 │
 │ APB
 ▼
┌──────────────┐
│ SpinalHDL IP │
│              │
│ Registers    │
│ Timer        │
│ Status       │
└──────────────┘
```

여기서부터 SoC 설계로 넘어갈 수 있습니다.

---

# 41. 효율적인 학습을 위해 반드시 해야 할 것

SpinalHDL을 단순히 책처럼 읽으면 효율이 떨어집니다.

추천 방식은:

```text
개념 학습
   ↓
10~30줄짜리 코드 작성
   ↓
Verilog 생성
   ↓
생성된 Verilog 확인
   ↓
Simulation
   ↓
Waveform 확인
   ↓
코드 수정
```

입니다.

특히 **SpinalHDL 코드와 생성된 Verilog를 비교하는 것**을 강력하게 추천합니다.

예:

### SpinalHDL

```scala
io.y := io.a & io.b
```

↓

### Generated RTL

```verilog
assign io_y = io_a & io_b;
```

이 과정을 반복하면:

> "내가 작성한 SpinalHDL 코드가 실제 어떤 하드웨어가 되는가?"

를 직관적으로 이해하게 됩니다.

---

# 42. SpinalHDL 프로젝트 구조

현재 공식 getting-started 문서는 SBT 기반 프로젝트를 제공하며 기본적인 프로젝트에는 `build.sbt`, `hw/`, `project/`, `README.md` 등이 포함됩니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Getting%20Started/Install%20and%20setup.html?utm_source=chatgpt.com)

대략:

```text
MySpinalProject/
│
├── build.sbt
├── project/
│
├── hw/
│   └── scala/
│       ├── AndGate.scala
│       ├── Counter.scala
│       └── Uart.scala
│
└── README.md
```

와 같은 구조로 시작하면 좋습니다.

---

# 43. 개발환경

현재 공식 문서에서는 Scala 환경과 함께 **JDK 17 LTS**를 권장하고 있으며, IDE로 IntelliJ IDEA 또는 VSCodium + Metals 등을 사용할 수 있습니다. Simulation에는 Verilator, waveform 확인에는 GTKWave를 사용할 수 있습니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Getting%20Started/Install%20and%20setup.html?utm_source=chatgpt.com)

초보자라면 다음 조합을 추천합니다.

```text
JDK 17
   +
SBT
   +
IntelliJ IDEA
   +
SpinalHDL
   +
Verilator
   +
GTKWave
   +
Git
```

다만 실제 프로젝트의 SpinalHDL/Scala 버전에 맞춰 공식 템플릿의 설정을 그대로 사용하는 것이 가장 안전합니다.

---

# 44. 처음에는 Scala를 얼마나 공부해야 하는가?

많이 공부할 필요 없습니다.

처음에는 다음 정도면 충분합니다.

### 반드시 필요한 Scala

```scala
class
object
val
var
def
if
for
foreach
case class
extends
new
```

그리고 조금 뒤:

```text
trait
generic/type parameter
higher-order function
map
filter
Option
```

정도를 배우면 충분합니다.

**Scala 전체를 먼저 공부한 다음 SpinalHDL을 시작할 필요는 없습니다.**

오히려:

```text
SpinalHDL
 ↓
필요한 Scala 문법 발견
 ↓
그 부분만 학습
 ↓
다시 SpinalHDL
```

방식이 효율적입니다.

---

# 45. SpinalHDL을 공부할 때 가장 흔한 실수

## 실수 1 — Scala 프로그램으로 생각한다

```scala
when(...)
```

를 software `if`처럼만 생각하면 안 됩니다.

→ 하드웨어 MUX/logic으로 생각해야 합니다.

---

## 실수 2 — `Reg`를 변수라고 생각한다

```scala
val counter = Reg(...)
```

→ 이것은 소프트웨어 변수와 다릅니다.

**flip-flop/register를 생성합니다.**

---

## 실수 3 — clock을 잊는다

Sequential logic은 clock 관점으로 생각해야 합니다.

---

## 실수 4 — width를 무시한다

```text
8-bit + 8-bit
```

에서 overflow를 반드시 고려해야 합니다.

---

## 실수 5 — 생성된 Verilog를 보지 않는다

처음에는 반드시 확인하는 것이 좋습니다.

---

## 실수 6 — 처음부터 CPU를 만든다

처음에는:

```text
AND
MUX
Adder
Counter
FSM
UART
FIFO
```

순으로 올라가는 것이 훨씬 좋습니다.

---

# 46. SpinalHDL의 핵심 개념을 한 장으로 정리하면

```text
                         SpinalHDL
                             │
             ┌───────────────┴───────────────┐
             │                               │
         Hardware                        Scala
         concepts                       language
             │                               │
      ┌──────┼───────┐                 ┌──────┼──────┐
      │      │       │                 │      │      │
    Bool   UInt    Reg                class   for   function
      │      │       │                 │      │      │
      └──────┴───────┴─────────────────┴──────┴──────┘
                             │
                             ▼
                       Elaboration
                             │
                             ▼
                       Hardware Graph
                             │
                             ▼
                  ┌──────────┼──────────┐
                  │          │          │
                Verilog    VHDL     SystemVerilog
                  │          │          │
                  └──────────┼──────────┘
                             ▼
                     Simulator / FPGA
                     / ASIC / EDA
```

이 그림을 머릿속에 가지고 있으면 좋습니다.

---

# 47. 결국 SpinalHDL에서 가장 중요한 10가지

처음에는 이것만 확실하게 익혀도 상당히 좋습니다.

### ① `Component`

```scala
class MyBlock extends Component
```

하드웨어 모듈.

### ② `Bundle`

```scala
val io = new Bundle { ... }
```

하드웨어 interface.

### ③ `Bool`

1-bit logic.

### ④ `Bits`

bit vector.

### ⑤ `UInt / SInt`

수치 연산.

### ⑥ `:=`

하드웨어 연결/대입.

### ⑦ `Reg`

clocked storage.

### ⑧ `when / switch`

조건부 하드웨어.

### ⑨ `Vec`

반복되는 hardware structure.

### ⑩ Elaboration

Scala가 실행되어 실제 하드웨어 구조를 만드는 과정.

---

# 48. 그리고 그 다음의 핵심은 "추상화"

SpinalHDL을 제대로 배우면 최종적으로는 다음 단계로 넘어갑니다.

처음에는:

```scala
class Counter extends Component
```

를 만드는 수준입니다.

그 다음:

```scala
class Counter(width: Int) extends Component
```

가 됩니다.

그 다음:

```scala
class Counter(
  width: Int,
  maxValue: Int,
  enableWidth: Int
) extends Component
```

처럼 parameterization합니다.

그리고 나중에는:

```text
Generic FIFO
Generic Pipeline
Generic Bus
Generic Cache
Generic CPU component
```

등을 만들게 됩니다.

즉,

> **SpinalHDL의 진짜 힘은 Verilog를 짧게 쓰는 것이 아니라 "재사용 가능한 하드웨어 생성기"를 만드는 데 있습니다.**

공식 문서에서도 Scala의 객체지향/함수형 기능과 generic design을 이용해 단순한 회로부터 복잡한 hardware generator까지 만들 수 있다는 점을 SpinalHDL의 중요한 특징으로 설명합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Introduction/SpinalHDL.html?utm_source=chatgpt.com)

---

# 49. 제가 추천하는 실제 학습 커리큘럼

처음 배우신다면 다음 순서가 가장 좋습니다.

| 단계 | 학습 내용 | 실습 |
|---|---|---|
| 1 | Digital Logic | AND/OR/XOR |
| 2 | `Component` / `Bundle` | AND Gate |
| 3 | `Bool/Bits` | MUX |
| 4 | `UInt/SInt` | Adder |
| 5 | `Reg` | Register |
| 6 | Clock/Reset | Counter |
| 7 | `when/switch` | Decoder |
| 8 | FSM | Traffic Light |
| 9 | `Vec` | Register File |
| 10 | Simulation | Counter Test |
| 11 | Waveform | GTKWave |
| 12 | Parameterization | Generic Counter |
| 13 | Bundle/Stream | FIFO |
| 14 | FSM + Counter | UART |
| 15 | Memory | RAM/ROM |
| 16 | Bus | APB Peripheral |
| 17 | Advanced | Pipeline |
| 18 | Generator | Generic CPU block |

**18단계까지 진행하면 SpinalHDL을 단순히 "조금 사용할 줄 아는 수준"을 넘어 실제 RTL 설계에 적용할 수 있는 기반이 생깁니다.**

---

# 50. 공식 학습 자료는 이렇게 활용하는 것이 좋다

현재 공식 SpinalHDL 문서는 **Getting Started → Data Types → Structuring → Semantic → Sequential Logic → Design Errors → Libraries → Simulation → Formal Verification → Examples** 순으로 구성되어 있어서, 초보자에게 상당히 좋은 학습 순서를 제공합니다. [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/index.html?utm_source=chatgpt.com)

특히 다음 자료를 추천합니다.

- [SpinalHDL 공식 Documentation](https://spinalhdl.github.io/SpinalDoc-RTD/?utm_source=chatgpt.com) — 전체 개념과 API를 배우는 기본 자료
- [SpinalHDL GitHub](https://github.com/SpinalHDL/SpinalHDL?utm_source=chatgpt.com) — 소스와 최신 개발 상황
- [SpinalTemplateSbt](https://github.com/SpinalHDL/SpinalTemplateSbt?utm_source=chatgpt.com) — 바로 시작할 수 있는 SBT 프로젝트 템플릿
- [SpinalWorkshop](https://github.com/SpinalHDL/SpinalWorkshop?utm_source=chatgpt.com) — 단계별 실습 자료
- [SpinalHDL Examples](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/index.html?utm_source=chatgpt.com) — 기본/중급/고급 예제 모음 [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/index.html?utm_source=chatgpt.com)

---

## 다음 단계

처음 학습하시는 목적이 **“SpinalHDL을 이용해서 디지털 논리회로를 효율적으로 설계하는 방법”**이라면, 위 내용을 한 번에 외우려고 하지 않는 것을 권합니다.

가장 좋은 방법은 제가 이어서 **실제 강의처럼 1단계부터 같이 진행하는 것**입니다.

추천 순서는:

```text
[Lesson 1]
SpinalHDL 개발환경 구축
       ↓
[Lesson 2]
첫 번째 AND Gate
       ↓
[Lesson 3]
Bool / Bits / UInt / SInt
       ↓
[Lesson 4]
MUX / Decoder / Adder
       ↓
[Lesson 5]
Reg / Clock / Reset
       ↓
[Lesson 6]
Counter
       ↓
[Lesson 7]
FSM
       ↓
[Lesson 8]
Simulation + Verilator + GTKWave
       ↓
[Lesson 9]
Bundle / Vec / Parameter
       ↓
[Lesson 10]
FIFO
       ↓
[Lesson 11]
UART
       ↓
[Lesson 12]
APB Peripheral
       ↓
[Lesson 13]
Pipeline
       ↓
[Lesson 14]
Hardware Generator
```

**특히 첫 단계에서는 Scala를 별도로 깊게 공부하지 않고, 디지털 회로 하나를 만들면서 필요한 Scala 문법을 동시에 익히는 방식**을 추천합니다.

원하시면 다음 답변부터는 **「SpinalHDL 초보자용 실전 강좌」 형식으로 Lesson 1부터 시작해서, 설치 → 프로젝트 생성 → 첫 AND Gate → 코드 한 줄 한 줄 해설 → 생성된 Verilog 분석 → Simulation까지 실제로 따라 할 수 있도록** 단계별로 진행할 수 있습니다.
