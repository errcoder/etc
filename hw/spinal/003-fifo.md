# 🚀 좋습니다. FIFO 시작합니다.

UART에서 배운 내용을 그대로 이어서 가겠습니다.

오늘은 **FIFO의 개념 → 구조 → 동작 원리 → SpinalHDL 구현**까지 한 단계씩 가겠습니다.

---

# 1. FIFO가 왜 필요한가?

FIFO는 **First In, First Out**의 약자입니다.

말 그대로:

> **먼저 들어온 데이터가 먼저 나간다.**

입니다.

가장 쉬운 예는 줄서기입니다.

```text
들어오는 순서

A → B → C → D

       FIFO

A → B → C → D

나가는 순서
```

A가 먼저 들어갔으므로 A가 먼저 나옵니다.

---

# 2. UART에서 FIFO가 왜 필요한가?

지금까지 만든 UART RX를 생각해봅시다.

```text
UART RX
   │
   ▼
data
valid
```

예를 들어 UART가 다음 데이터를 계속 수신한다고 해봅시다.

```text
'A'
'B'
'C'
'D'
```

RX가 `valid`를 발생시킬 때마다 CPU가 **그 순간 바로 데이터를 읽어야 한다면** 문제가 생길 수 있습니다.

CPU가 바쁜 상황이라면:

```text
UART RX
   │
   ▼
'A' ──────► CPU ?

'B' ──────► CPU ?

'C' ──────► CPU ?
```

CPU가 `'A'`를 처리하는 동안 `'B'`, `'C'`가 들어올 수 있습니다.

그래서 중간에 저장 공간을 둡니다.

```text
UART RX
   │
   ▼
┌──────────────┐
│     FIFO     │
│              │
│ A            │
│ B            │
│ C            │
└──────────────┘
       │
       ▼
      CPU
```

UART RX가 빠르게 데이터를 넣고 CPU가 조금 늦게 데이터를 꺼내도 FIFO가 중간에서 버퍼 역할을 합니다.

---

# 3. FIFO의 핵심 인터페이스

가장 단순한 FIFO를 생각해봅시다.

```text
                 FIFO
          ┌────────────────┐
write ───►│                │
data  ───►│                │───► readData
          │                │
read  ───►│                │
          └────────────────┘
             │          │
             ▼          ▼
           full       empty
```

입력은 보통:

```text
write
writeData
```

출력은:

```text
readData
full
empty
```

그리고:

```text
read
```

가 있습니다.

---

# 4. FIFO의 내부 구조

FIFO를 구현하려면 결국 **메모리**가 필요합니다.

예를 들어:

> 깊이 4, 데이터 폭 8 bit

FIFO라면:

```text
Depth = 4
Data width = 8
```

이므로 내부에는:

```text
┌──────────┐
│ Address  │ Data
├──────────┤
│    0     │ A
│    1     │ B
│    2     │ C
│    3     │ D
└──────────┘
```

같은 저장 공간이 있습니다.

그리고 중요한 것이 두 개 있습니다.

```text
writePtr
readPtr
```

---

# 5. Write Pointer

`writePtr`는:

> **다음 데이터를 어디에 쓸 것인가?**

를 가리킵니다.

처음에는:

```text
writePtr = 0
```

입니다.

A를 write하면:

```text
memory[0] = A
writePtr = 1
```

B를 write하면:

```text
memory[1] = B
writePtr = 2
```

C:

```text
memory[2] = C
writePtr = 3
```

D:

```text
memory[3] = D
writePtr = 0
```

여기서 중요한 것이 있습니다.

**Pointer가 끝까지 가면 다시 처음으로 돌아갑니다.**

즉:

```text
0 → 1 → 2 → 3 → 0 → 1 → ...
```

이것을 **circular buffer**라고 합니다.

---

# 6. Read Pointer

`readPtr`는:

> **다음 데이터를 어디에서 읽을 것인가?**

를 가리킵니다.

처음에는:

```text
readPtr = 0
```

입니다.

A를 read하면:

```text
memory[0] → A

readPtr = 1
```

B를 read하면:

```text
memory[1] → B

readPtr = 2
```

이렇게 움직입니다.

---

# 7. FIFO의 진짜 핵심: Empty와 Full

여기서 아주 중요한 문제가 발생합니다.

Depth가 4라고 합시다.

처음:

```text
writePtr = 0
readPtr  = 0
```

아무 데이터도 없습니다.

```text
FIFO = EMPTY
```

그런데 데이터를 4개 넣으면:

```text
writePtr = 0
readPtr  = 0
```

가 다시 됩니다.

하지만 이번에는:

```text
FIFO = FULL
```

입니다.

즉:

```text
writePtr == readPtr
```

만 보고서는

```text
EMPTY
```

인지

```text
FULL
```

인지 구분할 수 없습니다.

---

# 8. 그래서 Occupancy Counter를 사용하자

교육용 FIFO에서는 이 방법이 아주 이해하기 쉽습니다.

```text
count
```

를 하나 둡니다.

`count`는:

> **현재 FIFO에 몇 개의 데이터가 들어 있는가?**

입니다.

Depth = 4라면:

```text
count = 0
```

→ EMPTY

```text
count = 1
```

→ 데이터 1개

```text
count = 2
```

→ 데이터 2개

```text
count = 3
```

→ 데이터 3개

```text
count = 4
```

→ FULL

따라서:

```text
empty = count == 0
full  = count == depth
```

가 됩니다.

이 방식은 처음 FIFO를 배우기에 매우 좋습니다.

---

# 9. Write가 발생하면

정상적인 write 조건을:

```text
write && !full
```

이라고 하겠습니다.

그러면:

```text
memory[writePtr] = writeData
writePtr         = writePtr + 1
count            = count + 1
```

입니다.

---

# 10. Read가 발생하면

정상적인 read 조건은:

```text
read && !empty
```

입니다.

그러면:

```text
readData = memory[readPtr]
readPtr  = readPtr + 1
count    = count - 1
```

입니다.

여기서 잠깐.

**SpinalHDL에서 Memory read 방식은 동기식/비동기식 등에 따라 동작이 달라질 수 있습니다.**

그래서 첫 FIFO에서는 복잡한 RAM inference를 바로 다루지 않고, **명확한 `Vec(Reg(...))` 기반 저장소**로 시작하겠습니다.

이렇게 하면 하드웨어 동작을 눈으로 따라가기 쉽습니다.

---

# 11. 첫 번째 FIFO

우선:

```text
Data width = 8
Depth      = 4
```

로 고정해서 배우겠습니다.

```scala id="fifobase1"
import spinal.core._

case class SimpleFifo(
    dataWidth: Int,
    depth: Int
) extends Component {

  require(depth >= 2)
  require(dataWidth >= 1)

  val io = new Bundle {
    val write     = in Bool()
    val writeData = in Bits(dataWidth bits)

    val read      = in Bool()
    val readData  = out Bits(dataWidth bits)

    val full      = out Bool()
    val empty     = out Bool()
  }

  // ------------------------------------------------------------
  // Memory
  // ------------------------------------------------------------

  val mem = Vec(
    Reg(Bits(dataWidth bits)) init(0),
    depth
  )

  // ------------------------------------------------------------
  // Pointer width
  // ------------------------------------------------------------

  val ptrWidth =
    log2Up(depth)

  val writePtr =
    Reg(UInt(ptrWidth bits)) init(0)

  val readPtr =
    Reg(UInt(ptrWidth bits)) init(0)

  // ------------------------------------------------------------
  // Number of stored elements
  // ------------------------------------------------------------

  val countWidth =
    log2Up(depth + 1)

  val count =
    Reg(UInt(countWidth bits)) init(0)

  // ------------------------------------------------------------
  // Status
  // ------------------------------------------------------------

  io.empty := count === 0
  io.full  := count === depth

  // ------------------------------------------------------------
  // Read data
  // ------------------------------------------------------------

  io.readData := mem(readPtr)

  // ------------------------------------------------------------
  // Write / Read
  // ------------------------------------------------------------

  val doWrite =
    io.write && !io.full

  val doRead =
    io.read && !io.empty

  when(doWrite) {

    mem(writePtr) := io.writeData

    when(writePtr === depth - 1) {
      writePtr := 0
    } otherwise {
      writePtr := writePtr + 1
    }
  }

  when(doRead) {

    when(readPtr === depth - 1) {
      readPtr := 0
    } otherwise {
      readPtr := readPtr + 1
    }
  }

  // ------------------------------------------------------------
  // Occupancy counter
  // ------------------------------------------------------------

  when(doWrite && !doRead) {

    count := count + 1

  } elsewhen(!doWrite && doRead) {

    count := count - 1
  }
}
```

---

# 12. 여기서 정말 중요한 부분

이 부분을 잘 보세요.

```scala id="9a8x2p"
when(doWrite && !doRead) {
  count := count + 1

} elsewhen(!doWrite && doRead) {
  count := count - 1
}
```

그런데:

```text
write = 1
read  = 1
```

이면?

```text
doWrite = true
doRead  = true
```

입니다.

이 경우:

```text
데이터 하나 들어옴
데이터 하나 나감
```

이므로 FIFO에 들어 있는 데이터 개수는 **변하지 않아야 합니다.**

그래서:

```text
count = 그대로
```

입니다.

이것이 FIFO 설계에서 매우 중요한 부분입니다.

---

# 13. FIFO 상태 변화

초기:

```text
count = 0
empty = 1
full  = 0
```

A write:

```text
count = 1
empty = 0
full  = 0
```

B write:

```text
count = 2
```

C write:

```text
count = 3
```

D write:

```text
count = 4
empty = 0
full  = 1
```

이제 또 write를 하면:

```text
doWrite = write && !full
        = 1 && 0
        = 0
```

따라서 데이터가 들어가지 않습니다.

---

# 14. 그 다음 Read

현재:

```text
FIFO:

A
B
C
D
```

read:

```text
A ← read
```

그러면:

```text
FIFO:

B
C
D
```

그리고:

```text
count = 3
```

다시 read:

```text
B ← read
```

이제:

```text
C
D
```

입니다.

이것이 바로:

> **First In, First Out**

입니다.

---

# 15. 그런데 이 코드에서 한 가지 주의할 점

방금 구현한:

```scala
io.readData := mem(readPtr)
```

는 **read 요청이 발생한 순간에 데이터를 register에 저장하는 구조가 아닙니다.**

즉 `readData`는 현재 `readPtr`가 가리키는 메모리 내용을 보여주는 형태입니다.

따라서 첫 FIFO에서는 이것을:

> **"readData는 현재 FIFO의 head를 가리킨다."**

라고 이해하면 됩니다.

실제 FIFO IP나 FPGA RAM을 사용하면:

- synchronous read
- asynchronous read
- read-first
- write-first
- no-change

등의 메모리 동작 정책이 등장합니다.

이것은 나중에 제대로 다루겠습니다.

---

# 16. 다음은 Simulation

다음 단계에서는 바로 코드를 더 늘리지 않고,

```text
FIFO
 ↓
write A
write B
write C
 ↓
read A
read B
read C
```

를 simulation하고 waveform에서:

```text
writePtr
readPtr
count
full
empty
writeData
readData
```

를 직접 확인하겠습니다.

그리고 **동시에 `write`와 `read`가 들어오는 경우**, **FULL 상태에서 write**, **EMPTY 상태에서 read**까지 테스트하겠습니다.

이 테스트까지 통과시키면 그 다음에 **parameterized FIFO → UART RX FIFO 연결**로 넘어가면 됩니다.

---

---

# 🚀 좋습니다. FIFO Simulation으로 계속 갑시다.

이번에는 **코드를 그냥 실행하는 것보다 FIFO의 상태 변화를 직접 검증하는 것**에 집중하겠습니다.

그리고 한 가지 먼저 짚고 가겠습니다.

앞에서 만든 `SimpleFifo`에는 `depth`가 2의 거듭제곱이 아닐 때 pointer width와 비교식이 조금 더 신중해야 하는 부분이 있습니다. 이번 학습에서는 우선 **`depth = 4`**로 고정해서 동작 원리를 검증하고, 이후 parameterization을 제대로 정리하겠습니다.

---

# 1. 이번 Simulation에서 검증할 것

FIFO를 다음 순서로 테스트하겠습니다.

```text
초기
 ↓
A write
 ↓
B write
 ↓
C write
 ↓
A read
 ↓
B read
 ↓
동시 write/read
 ↓
FULL 테스트
 ↓
EMPTY 테스트
```

특히 다음 네 가지를 반드시 확인합니다.

| 상황 | 기대 결과 |
|---|---|
| EMPTY에서 read | 무시 |
| FULL에서 write | 무시 |
| write만 발생 | `count + 1` |
| read만 발생 | `count - 1` |
| write + read 동시 | `count` 유지 |

---

# 2. Simulation의 기본 원칙

이번에는 테스트벤치에서도 **매 clock마다 상태를 확인**하겠습니다.

예를 들어:

```scala
dut.clockDomain.waitSampling()
```

후에:

```scala
println(...)
```

을 실행합니다.

이렇게 해야 waveform의 clock edge와 우리가 출력한 상태를 대응시킬 수 있습니다.

---

# 3. FIFO Simulation

```scala
import spinal.core._
import spinal.core.sim._

object SimpleFifoSim extends App {

  SimConfig
    .withWave
    .doSim(
      SimpleFifo(
        dataWidth = 8,
        depth = 4
      )
    ) { dut =>

      // --------------------------------------------------------
      // Initial values
      // --------------------------------------------------------

      dut.io.write     #= false
      dut.io.writeData #= 0
      dut.io.read      #= false

      dut.clockDomain.forkStimulus(10)

      // --------------------------------------------------------
      // Helper
      // --------------------------------------------------------

      def printStatus(name: String): Unit = {

        println(
          f"$name%-20s " +
          f"empty=${dut.io.empty.toBoolean} " +
          f"full=${dut.io.full.toBoolean} " +
          f"readData=0x${dut.io.readData.toBigInt}%02X"
        )
      }

      // --------------------------------------------------------
      // Initial state
      // --------------------------------------------------------

      dut.clockDomain.waitSampling()

      printStatus("Initial")

      assert(
        dut.io.empty.toBoolean,
        "FIFO must be empty initially"
      )

      assert(
        !dut.io.full.toBoolean,
        "FIFO must not be full initially"
      )

      // --------------------------------------------------------
      // Write A
      // --------------------------------------------------------

      dut.io.writeData #= 0x41
      dut.io.write     #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false

      printStatus("After write A")

      assert(
        !dut.io.empty.toBoolean,
        "FIFO should not be empty after writing A"
      )

      // --------------------------------------------------------
      // Write B
      // --------------------------------------------------------

      dut.io.writeData #= 0x42
      dut.io.write     #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false

      printStatus("After write B")

      // --------------------------------------------------------
      // Write C
      // --------------------------------------------------------

      dut.io.writeData #= 0x43
      dut.io.write     #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false

      printStatus("After write C")

      // --------------------------------------------------------
      // Read A
      // --------------------------------------------------------

      assert(
        dut.io.readData.toBigInt == 0x41,
        "First FIFO data must be A"
      )

      dut.io.read #= true

      dut.clockDomain.waitSampling()

      dut.io.read #= false

      printStatus("After read A")

      // --------------------------------------------------------
      // Read B
      // --------------------------------------------------------

      assert(
        dut.io.readData.toBigInt == 0x42,
        "Second FIFO data must be B"
      )

      dut.io.read #= true

      dut.clockDomain.waitSampling()

      dut.io.read #= false

      printStatus("After read B")

      // --------------------------------------------------------
      // Read C
      // --------------------------------------------------------

      assert(
        dut.io.readData.toBigInt == 0x43,
        "Third FIFO data must be C"
      )

      dut.io.read #= true

      dut.clockDomain.waitSampling()

      dut.io.read #= false

      printStatus("After read C")

      // --------------------------------------------------------
      // FIFO should now be empty
      // --------------------------------------------------------

      assert(
        dut.io.empty.toBoolean,
        "FIFO must be empty after reading all data"
      )

      println()
      println("Basic FIFO test passed.")
    }
}
```

---

# 4. 그런데 여기서 아주 중요한 문제

방금 simulation을 보면 이런 코드가 있습니다.

```scala
dut.io.read #= true

dut.clockDomain.waitSampling()

dut.io.read #= false
```

그리고 그 전에:

```scala
assert(
  dut.io.readData.toBigInt == 0x41
)
```

를 검사했습니다.

이것은 우리가 설계한 FIFO의 **readData 의미** 때문입니다.

현재 FIFO는:

```scala
io.readData := mem(readPtr)
```

이기 때문에,

```text
read를 하기 직전
        │
        ▼
readPtr가 가리키는 값
        │
        ▼
readData
```

입니다.

즉 현재 구현은:

> **readData = FIFO의 현재 head**

라고 보는 것이 맞습니다.

---

# 5. Waveform에서는 이렇게 보입니다

A, B, C를 차례로 넣으면:

```text
clock       0   1   2   3
            │   │   │   │

write       0   1   1   1
writeData       A   B   C

count       0   1   2   3

readData    ?   A   A   A
```

개념적으로 FIFO 내부는:

```text
writePtr
   │
   ▼

┌─────┬─────┬─────┬─────┐
│  A  │  B  │  C  │     │
└─────┴─────┴─────┴─────┘
  ▲
  │
readPtr
```

입니다.

---

# 6. 이번에는 FULL도 테스트합시다

Depth가 4이므로 정확히 네 개를 넣으면:

```text
count = 4
```

가 됩니다.

즉:

```text
full = true
```

가 되어야 합니다.

테스트를 추가해봅시다.

```scala
// --------------------------------------------------------
// Fill FIFO completely
// --------------------------------------------------------

dut.io.writeData #= 0x44
dut.io.write     #= true

dut.clockDomain.waitSampling()

dut.io.write #= false

printStatus("After write D")

assert(
  dut.io.full.toBoolean,
  "FIFO must be full after four writes"
)
```

그러면:

```text
FIFO

A
B
C
D

count = 4
full  = true
empty = false
```

입니다.

---

# 7. FULL 상태에서 다시 Write

이제 중요한 테스트입니다.

```scala
dut.io.writeData #= 0x55
dut.io.write     #= true

dut.clockDomain.waitSampling()

dut.io.write #= false

printStatus("Write while full")
```

우리는 내부에서:

```scala
val doWrite =
  io.write && !io.full
```

라고 했습니다.

현재:

```text
write = 1
full  = 1
```

이므로:

```text
doWrite = 1 && 0
        = 0
```

입니다.

따라서 `0x55`는 FIFO에 들어가면 안 됩니다.

이것도 검증할 수 있습니다.

```text
A
B
C
D
```

상태가 그대로여야 합니다.

---

# 8. EMPTY 상태에서 Read

반대로 모든 데이터를 읽은 뒤:

```text
count = 0
empty = true
```

상태에서:

```scala
dut.io.read #= true
```

를 해도 아무 일도 일어나면 안 됩니다.

왜냐하면:

```scala
val doRead =
  io.read && !io.empty
```

이므로:

```text
read  = 1
empty = 1

doRead = 0
```

이기 때문입니다.

이것 역시 FIFO의 매우 중요한 안전장치입니다.

---

# 9. 가장 중요한 테스트: 동시 Read + Write

이제 FIFO의 핵심으로 들어갑니다.

현재 FIFO가:

```text
A
B
C
```

라고 합시다.

동시에:

```text
write = 1
read  = 1
```

그리고:

```text
writeData = D
```

를 넣습니다.

그러면 한 clock에서:

```text
A → read
D → write
```

가 동시에 발생합니다.

결과는:

```text
B
C
D
```

입니다.

데이터 개수는:

```text
3
```

에서:

```text
3
```

으로 그대로입니다.

그래서 우리가 작성한:

```scala
when(doWrite && !doRead) {
  count := count + 1

} elsewhen(!doWrite && doRead) {
  count := count - 1
}
```

가 중요한 것입니다.

둘 다 true이면 두 조건 모두 false이므로:

```text
count 변경 없음
```

입니다.

---

# 10. 이것을 반드시 기억하세요

FIFO의 `count` 변화는 세 가지뿐입니다.

```text
WRITE only
    ↓
count + 1


READ only
    ↓
count - 1


WRITE + READ
    ↓
count unchanged
```

그리고:

```text
NO WRITE + NO READ
    ↓
count unchanged
```

입니다.

표로 보면:

| write | read | count 변화 |
|---:|---:|---:|
| 0 | 0 | 0 |
| 1 | 0 | +1 |
| 0 | 1 | -1 |
| 1 | 1 | 0 |

이 표는 앞으로 FIFO 설계할 때 계속 머릿속에 가지고 있어야 합니다.

---

# 11. 그런데 FIFO 구현에서 한 단계 더 깊은 문제가 있습니다

현재 구현을 자세히 보면:

```scala
val doWrite =
  io.write && !io.full

val doRead =
  io.read && !io.empty
```

입니다.

그러면 FIFO가 **FULL인데 동시에 read와 write가 들어오면** 어떻게 될까요?

현재 설계에서는:

```text
full = true

write = 1
read  = 1
```

이면:

```text
doWrite = false
doRead  = true
```

입니다.

즉:

```text
read만 수행
write는 거부
```

됩니다.

이것은 한 가지 유효한 FIFO 정책입니다.

하지만 다른 FIFO 설계에서는:

> "FULL 상태에서 read와 write가 동시에 발생하면 write도 허용한다."

라는 정책을 사용할 수도 있습니다.

그 경우에는:

```text
FULL
  │
  ├── READ
  └── WRITE
       ↓
둘 다 허용
```

할 수 있습니다.

이 차이는 나중에 **FIFO throughput**을 이해할 때 중요합니다.

---

# 12. 지금 단계에서 중요한 것은 "정답 하나"가 아니라 정책

FIFO 설계에서 다음을 명확히 정의해야 합니다.

```text
FULL + WRITE ?
FULL + WRITE + READ ?

EMPTY + READ ?
EMPTY + WRITE + READ ?
```

즉 FIFO는 단순히 메모리를 만드는 문제가 아니라 **인터페이스의 동작 정책을 정의하는 문제**이기도 합니다.

---

# 13. 이번 단계에서 배운 구조

현재 FIFO는:

```text
                 ┌──────────────────┐
                 │                  │
writeData ──────►│      Memory      │
                 │                  │
writePtr ───────►│                  │
                 │                  │
readPtr ────────►│                  │
                 │                  │
                 └──────────────────┘
                     ▲          │
                     │          ▼
                 writePtr    readData
                     │
                     │
                 ┌───┴───┐
                 │ count │
                 └───┬───┘
                     │
                ┌────┴────┐
                ▼         ▼
              empty      full
```

입니다.

---

# 14. 다음 단계에서 할 중요한 개선

현재 `SimpleFifo`는 학습용으로 아주 좋지만, 실제 FPGA/ASIC 설계 관점에서는 개선할 부분이 있습니다.

다음 단계에서는 이것을 하나씩 고치겠습니다.

### ① `depth`가 4가 아닌 경우

현재:

```scala
log2Up(depth)
```

와 pointer wrap을 더 정확하게 처리해야 합니다.

예:

```text
depth = 5
```

이면 pointer가:

```text
0 → 1 → 2 → 3 → 4 → 0
```

이어야 하지:

```text
0 → 1 → 2 → 3 → 4 → 5 → ...
```

가 되면 안 됩니다.

---

### ② Memory를 제대로 이해하기

현재:

```scala
Vec(Reg(...))
```

로 만들었습니다.

다음에는 SpinalHDL의 `Mem`을 사용해서:

```text
Register 기반 저장소
        ↓
Memory 기반 저장소
```

차이를 살펴보겠습니다.

---

### ③ Read semantics

특히:

```scala
io.readData := mem(readPtr)
```

와

```text
read 요청 → 다음 clock에 data
```

의 차이를 명확히 이해해야 합니다.

이것은 FPGA Block RAM을 사용할 때 굉장히 중요합니다.

---

### ④ 마지막으로 UART + FIFO

최종적으로는:

```text
                  ┌─────────┐
Serial ──► UART RX ─► FIFO  │
                  └────┬────┘
                       │
                       ▼
                      CPU
```

그리고 반대로:

```text
CPU
 │
 ▼
 FIFO
 │
 ▼
UART TX
 │
 ▼
Serial
```

구조를 만들게 됩니다.

---

# 🎯 지금 여기서 잠깐 멈추는 게 좋습니다

다음 단계는 바로 더 복잡한 FIFO를 만드는 것보다 **현재 `SimpleFifo`의 simulation waveform을 직접 읽는 것**이 좋습니다.

특히 다음을 GTKWave에서 확인해 보세요.

```text
write
writeData
read
readData
writePtr
readPtr
count
empty
full
```

그리고 머릿속으로:

```text
write A
write B
write C
read A
read B
read C
```

할 때 각각의 clock에서

```text
writePtr
readPtr
count
```

가 어떻게 변해야 하는지 예측해보는 겁니다.

**그 예측과 waveform이 일치하면 FIFO의 핵심 원리를 제대로 잡은 것입니다.**

---

---


