이제 UART Loopback 을 만들어서 시뮬레이션 해보고, 그 다음으로 FIFO 를 학습하자.

좋습니다. 이번에는 **UART TX → UART RX를 실제로 연결한 Loopback 구조**를 먼저 만들고, 
시뮬레이션으로 `0x41`이 TX를 거쳐 RX에서 정확히 복원되는 것까지 확인하겠습니다.

그리고 그 다음 단계로 **FIFO**를 학습하겠습니다.  
이번에는 특히 이전 문제를 반영해서 **테스트벤치의 종료 조건, clock 수, UART frame 길이까지 명확하게 계산해서** 작성하겠습니다. 

# 1. UART Loopback의 구조

우리가 이미 만든 두 블록을 연결합니다.

```text
             ┌─────────────┐
data ───────►│             │
valid ──────►│   UART TX   │────── tx ──────┐
             │             │                 │
             └─────────────┘                 │
                                             ▼
                                      ┌─────────────┐
                                      │   UART RX   │
                                      │             │
                                      └─────────────┘
                                             │
                                             ▼
                                           data
                                           valid
```

즉,

```text
TX.tx ───────────────► RX.rx
```

로 연결합니다.

이번 예제에서는 **실제 외부 핀 없이 내부에서 TX와 RX를 직접 연결**합니다.

---

# 2. UART Loopback Component

먼저 `UartTx`, `UartRx`를 하나의 Component 안에 넣겠습니다.

```scala
import spinal.core._

case class UartLoopback(
    clockFrequencyHz: Int,
    baudRate: Int
) extends Component {

  val io = new Bundle {
    val txData  = in Bits(8 bits)
    val txValid = in Bool()

    val rxData  = out Bits(8 bits)
    val rxValid = out Bool()

    val txBusy  = out Bool()
  }

  // ------------------------------------------------------------
  // UART TX
  // ------------------------------------------------------------

  val tx = UartTx(
    clockFrequencyHz = clockFrequencyHz,
    baudRate = baudRate
  )

  // ------------------------------------------------------------
  // UART RX
  // ------------------------------------------------------------

  val rx = UartRx(
    clockFrequencyHz = clockFrequencyHz,
    baudRate = baudRate
  )

  // ------------------------------------------------------------
  // TX 입력 연결
  // ------------------------------------------------------------

  tx.io.data  := io.txData
  tx.io.valid := io.txValid

  // ------------------------------------------------------------
  // TX → RX Loopback
  // ------------------------------------------------------------

  rx.io.rx := tx.io.tx

  // ------------------------------------------------------------
  // RX 출력
  // ------------------------------------------------------------

  io.rxData  := rx.io.data
  io.rxValid := rx.io.valid

  // TX busy
  io.txBusy := tx.io.busy
}
```

핵심은 딱 이것입니다.

```scala
rx.io.rx := tx.io.tx
```

하드웨어적으로는:

```text
        TX
         │
         │ tx
         ▼
        RX
```

라는 **wire 연결**이 만들어집니다.

---

# 3. 여기서 중요한 점

`UartLoopback` 안에 새로운 UART 로직을 만든 것이 아닙니다.

이미 만든

```scala
UartTx(...)
```

와

```scala
UartRx(...)
```

두 개의 Component를 **상위 Component에서 연결**한 것입니다.

이게 SpinalHDL에서 매우 중요한 구조입니다.

```text
Top
 │
 ├── UartTx
 │
 └── UartRx
```

실제 FPGA/ASIC 설계에서도 이런 식으로 작은 블록들을 조립해서 큰 시스템을 만듭니다.

---

# 4. Loopback Simulation

이번에는 `0x41` 하나를 보내겠습니다.

`0x41`은 ASCII `'A'`입니다.

UART frame은:

```text
START
  ↓
0
```

그 다음 LSB first:

```text
0x41 = 0100_0001

D0 = 1
D1 = 0
D2 = 0
D3 = 0
D4 = 0
D5 = 0
D6 = 1
D7 = 0
```

그리고 STOP:

```text
1
```

따라서:

```text
0 1 0 0 0 0 0 1 0 1
↑                 ↑
START             STOP
```

총 **10 bit**입니다.

---

# 5. Simulation 코드

이번에는 이전의 `while` 방식 대신 **프레임에 필요한 clock 수를 직접 계산해서 기다립니다.**

```scala
import spinal.core._
import spinal.core.sim._

object UartLoopbackSim extends App {

  val clockFrequency = 10_000_000
  val baudRate       = 1_000_000

  val cyclesPerBit =
    clockFrequency / baudRate

  // UART frame:
  //
  // START + 8 DATA + STOP
  //
  val frameBits = 10

  // 여유를 조금 둔다.
  val simulationCycles =
    cyclesPerBit * (frameBits + 2)

  SimConfig
    .withWave
    .doSim(
      UartLoopback(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      // --------------------------------------------------------
      // 초기 입력
      // --------------------------------------------------------

      dut.io.txData  #= 0
      dut.io.txValid #= false

      // clock
      dut.clockDomain.forkStimulus(10)

      // --------------------------------------------------------
      // IDLE 상태 확인
      // --------------------------------------------------------

      dut.clockDomain.waitSampling(5)

      assert(
        !dut.io.txBusy.toBoolean,
        "TX should be idle before transmission"
      )

      // --------------------------------------------------------
      // 0x41 전송 요청
      // --------------------------------------------------------

      dut.io.txData  #= 0x41
      dut.io.txValid #= true

      // TX가 데이터를 받아들이는 clock
      dut.clockDomain.waitSampling()

      // valid는 pulse로 사용
      dut.io.txValid #= false

      // --------------------------------------------------------
      // UART frame 전송 완료까지 기다린다.
      //
      // START + DATA(8) + STOP
      // = 10 bit
      //
      // 1 bit = 10 clocks
      //
      // 총 100 clocks
      // --------------------------------------------------------

      dut.clockDomain.waitSampling(simulationCycles)

      // --------------------------------------------------------
      // RX 결과 확인
      // --------------------------------------------------------

      println(
        f"RX data  = 0x${dut.io.rxData.toBigInt}%02X"
      )

      println(
        s"RX valid = ${dut.io.rxValid.toBoolean}"
      )

      assert(
        dut.io.rxData.toBigInt == 0x41,
        "UART Loopback received wrong data"
      )

      // RX valid는 pulse이므로
      // 정확한 시점에 검사해야 한다.
      //
      // 여기서는 data가 정확히 복원되었는지 먼저 확인한다.

      println("UART Loopback simulation finished")
    }
}
```

다만 여기에는 **한 가지 더 중요한 부분**이 있습니다.

`rxValid`는 우리가 만든 `UartRx`에서:

```scala
io.valid := False
```

로 기본값을 주고,

```scala
when(io.rx) {
  io.valid := True
}
```

하는 구조입니다.

따라서 `rxValid`는 **STOP bit가 끝나는 특정 clock에서만 1이 되는 pulse**입니다.

즉 단순히 마지막에:

```scala
assert(dut.io.rxData.toBigInt == 0x41)
```

만 확인하는 것보다 **`rxValid`가 발생하는 순간을 잡아서 검사하는 것이 더 좋은 테스트벤치**입니다.

---

# 6. 더 제대로 된 Loopback 테스트

그래서 이번에는 simulation에서 `rxValid`를 기다리되, 무작정 `while`을 사용하지 않고 **명확한 최대 clock 수를 제한**하겠습니다.

```scala
import spinal.core._
import spinal.core.sim._

object UartLoopbackSim extends App {

  val clockFrequency = 10_000_000
  val baudRate       = 1_000_000

  val cyclesPerBit =
    clockFrequency / baudRate

  val frameBits = 10

  SimConfig
    .withWave
    .doSim(
      UartLoopback(
        clockFrequencyHz = clockFrequency,
        baudRate = baudRate
      )
    ) { dut =>

      dut.io.txData  #= 0
      dut.io.txValid #= false

      dut.clockDomain.forkStimulus(10)

      // --------------------------------------------------------
      // IDLE
      // --------------------------------------------------------

      dut.clockDomain.waitSampling(5)

      assert(
        !dut.io.txBusy.toBoolean,
        "TX should be idle"
      )

      // --------------------------------------------------------
      // TX request
      // --------------------------------------------------------

      dut.io.txData  #= 0x41
      dut.io.txValid #= true

      dut.clockDomain.waitSampling()

      dut.io.txValid #= false

      // --------------------------------------------------------
      // RX valid를 기다린다.
      //
      // 단, 무한 loop는 사용하지 않는다.
      // UART frame의 최대 길이를 알고 있기 때문에
      // timeout을 둔다.
      // --------------------------------------------------------

      var received = false

      for (_ <- 0 until cyclesPerBit * (frameBits + 2)) {

        dut.clockDomain.waitSampling()

        if (dut.io.rxValid.toBoolean) {

          val receivedData =
            dut.io.rxData.toBigInt

          println(
            f"RX valid! data = 0x$receivedData%02X"
          )

          assert(
            receivedData == 0x41,
            f"Expected 0x41, but received 0x$receivedData%02X"
          )

          received = true
        }
      }

      // --------------------------------------------------------
      // 반드시 RX valid가 발생해야 한다.
      // --------------------------------------------------------

      assert(
        received,
        "RX valid was not detected before timeout"
      )

      println(
        "UART Loopback simulation finished successfully"
      )
    }
}
```

이 방식이 교육용으로 훨씬 좋습니다.

왜냐하면:

```scala
for (_ <- 0 until ...)
```

때문에 **simulation이 무한히 기다릴 수 없기 때문입니다.**

예를 들어 UART RX가 고장 나서 `rxValid`가 영원히 올라오지 않아도:

```text
TIMEOUT
 ↓
simulation 종료
 ↓
assert 실패
```

가 됩니다.

반면 무제한 `while`은 잘못 작성하면:

```text
simulation
   ↓
무한 대기
   ↓
simulation 종료 안 됨
```

이라는 문제가 발생할 수 있습니다.

---

# 7. 우리가 이번 Loopback에서 검증하는 것

이번 테스트는 단순히 "코드가 돌아간다"를 보는 게 아닙니다.

다음 경로 전체를 검증합니다.

```text
0x41
 │
 ▼
TX input
 │
 ▼
UART TX
 │
 ▼
START
 │
 ▼
D0 ~ D7
 │
 ▼
STOP
 │
 ▼
TX wire
 │
 ▼
UART RX
 │
 ▼
START detection
 │
 ▼
DATA sampling
 │
 ▼
STOP check
 │
 ▼
RX data
 │
 ▼
0x41
```

즉:

```text
0x41 ──► TX ──► serial wire ──► RX ──► 0x41
```

가 되는지를 확인합니다.

---

# 8. Waveform에서 확인할 것

GTKWave에서는 최소한 다음 신호를 봅니다.

```text
io_txData
io_txValid
io_txBusy

tx.io.tx

rx.io.rx
rx.io.data
rx.io.valid
```

특히:

```text
tx.io.tx
rx.io.rx
```

가 동일해야 합니다.

Loopback이므로 당연히:

```text
TX ───────── RX
```

입니다.

UART frame도 다음과 같이 보여야 합니다.

```text
       START    DATA                  STOP
TX       0     1 0 0 0 0 0 1 0          1
         │     │                    │
         └─────┴────────────────────┘
```

각 bit가 정확히 `10 clocks` 동안 유지되는지도 확인하면 좋습니다.

---

# 9. 여기까지 하면 UART에서 배울 수 있는 것이 정리됩니다

지금까지 만든 구조를 보면:

```text
               UART
                │
       ┌────────┴────────┐
       │                 │
     UART TX           UART RX
       │                 │
       └──── Loopback ────┘
```

여기서 중요한 디지털 설계 개념을 꽤 많이 배웠습니다.

| 개념 | UART에서 배운 것 |
|---|---|
| Register | `Reg(...)` |
| Counter | `bitTimer`, `bitCounter` |
| FSM | `IDLE`, `START`, `DATA`, `STOP` |
| Shift Register | `shiftReg` |
| Timing | `cyclesPerBit` |
| Serial protocol | UART frame |
| Component | TX/RX 분리 |
| Interconnection | TX → RX |
| Simulation | `waitSampling()` |
| Verification | `assert()` |
| Waveform | VCD / GTKWave |

이제 **FIFO로 넘어가기 아주 좋은 시점**입니다.

---

# 10. 다음: FIFO

FIFO는 UART와 연결해서 생각하면 훨씬 쉽습니다.

현재는:

```text
CPU
 │
 ▼
UART TX
 │
 ▼
Serial
```

인데, 실제 시스템에서는 보통:

```text
CPU
 │
 ▼
┌─────────┐
│  FIFO   │
└─────────┘
 │
 ▼
UART TX
```

처럼 만듭니다.

RX도:

```text
UART RX
   │
   ▼
┌─────────┐
│  FIFO   │
└─────────┘
   │
   ▼
CPU
```

가 됩니다.

그래서 FIFO를 배우면 **UART가 실제 시스템에서 어떻게 사용되는지**까지 자연스럽게 연결됩니다.

다음 단계에서는 먼저 가장 단순한 **Single-Clock FIFO**부터 만들겠습니다.

```text
             WRITE
               │
               ▼
        ┌───────────────┐
        │               │
        │     FIFO      │
        │               │
        └───────────────┘
               │
               ▼
             READ
```

그리고 순서대로:

1. FIFO가 왜 필요한가
2. `write`, `read`, `full`, `empty`
3. FIFO 내부 Memory
4. Write Pointer
5. Read Pointer
6. Occupancy Counter
7. Full / Empty 판정
8. SpinalHDL로 구현
9. Simulation
10. Waveform 검증
11. UART TX FIFO에 연결

순서로 가면 좋습니다.

---

---

---

"rxValid는 STOP bit가 끝나는 특정 clock에서만 1이 되는 pulse입니다." --> UartRx 모듈에서 io.valid 는 초기값으로 False 주어진다. 
데이터 수신이 완료되는 State.STOP 상태에서 io.valid := True 이고, 그 이후로 False 로 되돌리는 어떠한 로직도 코드 상에는 보이지 않는다.  
그런데 UartRx 모듈의 io.valid 에 직접 연결된 UartLoopback 의 io.rxValid 가 어떻게 1 clock pulse 가 될수 있나?

맞습니다. **질문하신 지점이 정확합니다.**  
그리고 제가 앞에서 `"rxValid는 STOP bit가 끝나는 특정 clock에서만 1이 되는 pulse"`라고 말하면서 그 이유를 충분히 설명하지 않은 것이 문제였습니다.

핵심은 **`io.valid`가 `Reg`가 아니라 `Bool` 출력이고, 우리가 `:=`로 조합 논리를 만들었다는 것**입니다.

---

# 1. 먼저 `UartRx` 코드를 정확히 다시 보겠습니다

우리가 작성했던 부분은:

```scala
io.data  := shiftReg
io.valid := False

switch(state) {

  // ...

  is(State.STOP) {
    when(bitTimer === cyclesPerBit - 1) {
      bitTimer := 0

      when(io.rx) {
        io.valid := True
      }

      state := State.IDLE
    }
  }

  // ...
}
```

여기서 중요한 것은:

```scala
io.valid := False
```

입니다.

이것은

> "처음에 False로 저장해 놓는다"

라는 의미가 **아닙니다.**

---

# 2. `io.valid`는 Register가 아닙니다

다음과 비교해 보면 명확합니다.

### Register

```scala
val validReg = Reg(Bool()) init(False)

validReg := True
```

이것은 실제로 **flip-flop**이 만들어집니다.

즉:

```text
clock
  │
  ▼
┌──────────┐
│ validReg │
│  FF      │
└──────────┘
```

`True`가 저장되면 다음 clock 이후에도 계속 `True`입니다.

따라서 다시:

```scala
validReg := False
```

라고 하지 않는 한 계속 유지됩니다.

---

# 3. 그런데 `io.valid`는 다릅니다

우리는:

```scala
val io = new Bundle {
  val valid = out Bool()
}
```

라고 했습니다.

그리고:

```scala
io.valid := False
```

를 했습니다.

이것은 **조합 논리의 기본값(default assignment)**입니다.

개념적으로 보면 SpinalHDL은 다음과 같은 하드웨어를 만들게 됩니다.

```text
                  ┌────────────────────┐
state ───────────►│                    │
bitTimer ────────►│  combinational     │────► io.valid
rx ──────────────►│  logic             │
                  │                    │
                  └────────────────────┘
```

즉 `io.valid` 자체에는 저장 기능이 없습니다.

---

# 4. 실제로 어떤 논리가 만들어지는가?

코드를 조금 단순화해 보겠습니다.

```scala
io.valid := False

when(condition) {
  io.valid := True
}
```

이것은 개념적으로:

```text
if (condition)
    valid = 1
else
    valid = 0
```

와 같습니다.

즉:

```text
condition = false
       │
       ▼
valid = 0


condition = true
       │
       ▼
valid = 1
```

입니다.

`valid`에 `1`을 **저장하는 것이 아닙니다.**

---

# 5. 그러면 UartRx에서는 언제 `valid = True`인가?

정확하게 보면:

```scala
io.valid := False
```

가 기본값이고,

```scala
is(State.STOP) {
  when(bitTimer === cyclesPerBit - 1) {

    when(io.rx) {
      io.valid := True
    }

    state := State.IDLE
  }
}
```

에서만 `True`로 override됩니다.

따라서 조건은 사실상:

```text
state == STOP
AND
bitTimer == cyclesPerBit - 1
AND
rx == 1
```

입니다.

즉 개념적으로:

```scala
io.valid :=
  (state === State.STOP) &&
  (bitTimer === cyclesPerBit - 1) &&
  io.rx
```

와 같은 조합 논리입니다.

---

# 6. 그래서 왜 정확히 1 clock인가?

여기가 핵심입니다.

예를 들어:

```text
cyclesPerBit = 10
```

이라고 합시다.

STOP 상태에서 `bitTimer`가:

```text
0
1
2
3
4
5
6
7
8
9
```

로 증가합니다.

그리고:

```scala
when(bitTimer === 9) {
    ...
    state := State.IDLE
}
```

가 실행됩니다.

따라서 마지막 STOP sampling 시점에서:

```text
state     = STOP
bitTimer  = 9
rx        = 1
```

이면:

```text
io.valid = 1
```

입니다.

동시에 clock edge에서:

```scala
state := State.IDLE
```

가 예약됩니다.

다음 clock부터는:

```text
state = IDLE
```

이 됩니다.

그러면 더 이상:

```scala
is(State.STOP)
```

조건이 성립하지 않습니다.

따라서 기본값:

```scala
io.valid := False
```

가 적용됩니다.

결과적으로:

```text
clock       N       N+1
             │        │
state       STOP     IDLE
             │        │
bitTimer      9        0
             │        │
io.valid      1        0
```

이 됩니다.

그래서 **1-clock pulse**가 됩니다.

---

# 7. 그림으로 보면 더 명확합니다

```text
             STOP 상태
                │
                │ bitTimer == 9
                │ rx == 1
                ▼
clock       ────┐       ┌────
                │       │
                ▼       ▼

state       STOP       IDLE
             ────────────
             
bitTimer       9         0
               

io.valid       1         0
             ────┐
                 │
                 └────────────
```

`io.valid`가 `1`이 된 것을 다음 clock까지 저장하는 Register가 존재하는 게 아닙니다.

**STOP + 마지막 timer 조건이 성립하는 동안만 조합적으로 1이 되는 것**입니다.

---

# 8. 중요한 SpinalHDL 개념

이 부분은 앞으로 FIFO를 배울 때도 굉장히 중요합니다.

## `:=`가 Register를 만드는 것은 아닙니다

예를 들어:

```scala
io.valid := False
```

라고 해서 `valid`에 `False`가 저장되는 것이 아닙니다.

**왼쪽이 무엇인지**가 중요합니다.

### Register에 대입

```scala
val counter = Reg(UInt(8 bits))

counter := counter + 1
```

→ `counter`는 저장소입니다.

### Output에 대입

```scala
io.valid := False
```

→ 조합 논리를 정의합니다.

---

# 9. 그럼 `io.valid := True`를 계속 유지하려면?

질문하신 것처럼 만약 다음과 같은 코드였다면:

```scala
val valid = Reg(Bool()) init(False)

when(condition) {
  valid := True
}
```

이 경우에는 이야기가 완전히 달라집니다.

```text
condition 발생
      │
      ▼
valid Register
      │
      ▼
     True
```

그리고 이후 `condition`이 사라져도:

```text
valid = True
```

가 계속 유지됩니다.

다시 False로 만들려면:

```scala
when(clear) {
  valid := False
}
```

같은 로직이 필요합니다.

---

# 10. 그런데 현재 UartRx의 `io.valid`는 Register가 아닙니다

현재 구조는:

```text
                ┌─────────────────────┐
state ─────────►│                     │
bitTimer ──────►│  Combinational      │────► io.valid
rx ────────────►│  Logic              │
                │                     │
                └─────────────────────┘
```

이고,

```text
state
  │
  ▼
┌─────────┐
│   FF    │
└─────────┘
```

만 Register입니다.

즉:

```text
state       → Register
bitTimer    → Register
bitCounter  → Register
shiftReg    → Register

io.valid    → Combinational output
```

입니다.

---

# 11. 따라서 제가 앞에서 한 표현을 조금 수정해야 합니다

제가 이전에:

> "`rxValid`는 STOP bit가 끝나는 특정 clock에서만 1이 되는 pulse"

라고 했는데, 더 정확하게 표현하면:

> **`io.valid`는 Register가 아니라 조합 출력이며, `State.STOP`에서 마지막 bit timer 조건과 `rx == 1` 조건이 
만족되는 동안 `True`가 된다. 그리고 같은 clock edge에서 `state`가 `IDLE`로 변경되므로 다음 clock에는 기본값 `False`가 되어 결과적으로 1-clock pulse처럼 동작한다.**

입니다.

이 차이가 상당히 중요합니다.

---

# 12. 그리고 여기서 한 가지 더 중요한 문제

사실 현재 `UartRx` 코드에는 **교육용으로는 괜찮지만 개선할 만한 부분**이 있습니다.

현재:

```scala
io.valid := False

switch(state) {
    ...
    is(State.STOP) {
        when(bitTimer === cyclesPerBit - 1) {
            when(io.rx) {
                io.valid := True
            }
            state := State.IDLE
        }
    }
}
```

이 구조에서는 `valid`가 **순수 조합 신호**입니다.

따라서 우리가 시스템을 좀 더 제대로 설계한다면 보통:

```text
RX 완료
   │
   ▼
data register에 저장
   │
   ▼
valid pulse
```

라는 명확한 handshake 구조를 설계하게 됩니다.

그리고 이것이 다음에 FIFO를 배우면:

```text
UART RX
   │
   │ data + valid
   ▼
┌──────────┐
│   FIFO   │
└──────────┘
```

로 자연스럽게 연결됩니다.

---

그리고 **제가 앞서 Loopback simulation에서 `rxValid`를 다루는 설명도 이 점을 정확하게 반영해서 다시 작성해야 합니다.
** 단순히 "STOP 끝나면 valid가 1이 된다"고 설명하는 것은 부족했습니다.

좋은 질문입니다. 이 부분을 확실히 이해하고 넘어가는 것이 맞습니다.
