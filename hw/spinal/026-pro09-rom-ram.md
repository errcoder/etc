좋습니다. **Project 8은 완료**하고, 바로 **Project 9 — ROM + RAM**으로 넘어가겠습니다.

이번 프로젝트의 목표는 메모리의 종류를 많이 배우는 것이 아니라, **RTL에서 메모리를 어떻게 표현하고 읽기/쓰기가 clock과 어떻게 관계되는지**를 확실히 이해하는 것입니다.

# Project 9 — ROM + RAM

## 1. 학습 목표

이번에는 다음 4가지만 잡겠습니다.

```text
ROM
 ├─ address → data
 └─ read-only

RAM
 ├─ address → data
 ├─ write
 └─ read

그리고
 └─ synchronous read / write timing
```

특히 **RAM의 read timing**을 simulation으로 확인하는 것이 중요합니다.

---

# 2. ROM

먼저 가장 간단한 ROM입니다.

```text
             address
                │
                ▼
          ┌───────────┐
          │    ROM    │
          ├───────────┤
          │ 0 : 0x10  │
          │ 1 : 0x22  │
          │ 2 : 0x35  │
          │ 3 : 0x47  │
          │ 4 : 0x59  │
          │ ...       │
          └─────┬─────┘
                │
                ▼
               data
```

SpinalHDL에서는 `Mem`을 사용합니다.

```scala
import spinal.core._

class SimpleRom extends Component {
  val io = new Bundle {
    val address = in UInt(3 bits)
    val data    = out Bits(8 bits)
  }

  val rom = Mem(Bits(8 bits), wordCount = 8)

  rom.init(
    Seq(
      B"8'x10",
      B"8'x22",
      B"8'x35",
      B"8'x47",
      B"8'x59",
      B"8'x6A",
      B"8'x7B",
      B"8'x8C"
    )
  )

  io.data := rom.readAsync(io.address)
}
```

여기서:

```scala
rom.readAsync(io.address)
```

는 **비동기 read**입니다.

즉 개념적으로:

```text
address 변경
     │
     ▼
    ROM
     │
     ▼
data 변경
```

이며 별도의 clock edge를 기다리지 않습니다.

이번 ROM에서는 이것만 사용하겠습니다.

---

# 3. ROM simulation

예를 들어:

```text
address   data
----------------
0         0x10
1         0x22
2         0x35
3         0x47
4         0x59
5         0x6A
6         0x7B
7         0x8C
```

를 확인하면 됩니다.

Testbench:

```scala
import spinal.core._
import spinal.core.sim._

object SimpleRomSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleRom)
    .doSim { dut =>

      for (address <- 0 until 8) {
        dut.io.address #= address
        sleep(1)

        println(
          f"address=$address%d data=0x${dut.io.data.toBigInt}%02X"
        )
      }
    }
}
```

여기서는 `waitSampling()`을 사용하지 않습니다.

왜냐하면 이번 ROM은:

```scala
readAsync()
```

이므로 **clocked operation이 아니기 때문**입니다.

이 차이를 기억해 두면 좋습니다.

---

# 4. RAM

이제 RAM으로 넘어갑니다.

구조는:

```text
             address
                │
                ▼
          ┌───────────┐
write ───>│           │
          │    RAM    │
data ────>│           │
          │           │
          └─────┬─────┘
                │
                ▼
             readData
```

간단한 RAM을 만들어 보겠습니다.

```scala
import spinal.core._

class SimpleRam extends Component {
  val io = new Bundle {
    val write    = in Bool()
    val address  = in UInt(2 bits)
    val writeData = in Bits(8 bits)
    val readData  = out Bits(8 bits)
  }

  val ram = Mem(Bits(8 bits), wordCount = 4)

  when(io.write) {
    ram.write(
      address = io.address,
      data = io.writeData
    )
  }

  io.readData := ram.readAsync(io.address)
}
```

여기서 RAM은:

```text
write = 1
```

일 때 해당 address에 데이터를 저장합니다.

예:

```text
address = 2
writeData = 0xAB
write = 1
```

이면:

```text
RAM[2] = 0xAB
```

가 됩니다.

그리고:

```text
address = 2
```

를 읽으면:

```text
readData = 0xAB
```

가 나옵니다.

---

# 5. 중요한 점 — RAM도 read 방식이 여러 가지다

여기서 너무 깊게 들어가지는 않겠습니다.

RAM read에는 대표적으로:

```text
Asynchronous Read
        vs
Synchronous Read
```

가 있습니다.

이번 Project 9에서는 우선 **둘의 차이를 정확하게 이해하는 것**만 중요합니다.

### Async read

```text
address ──> RAM ──> data
```

clock 불필요.

### Sync read

```text
address
   │
   ▼
  RAM
   │
 clock edge
   │
   ▼
 data
```

clock에 맞춰 data가 나옵니다.

---

# 6. 왜 이게 중요한가?

실제 FPGA에서는 RAM을 단순한 논리회로로 만드는 것이 아니라 **FPGA의 Block RAM 같은 전용 memory resource로 구현**하는 경우가 많습니다.

따라서 나중에는:

```text
SpinalHDL
    ↓
Mem
    ↓
FPGA memory inference
    ↓
Block RAM
```

이라는 관계를 이해하게 됩니다.

지금 단계에서는 이것만 기억하면 충분합니다.

> **`Mem`은 단순한 배열이 아니라 RTL memory를 표현하는 SpinalHDL의 기본 도구다.**

---

# 7. 이번 프로젝트의 핵심 비교

```text
┌────────────────────────────────────┐
│ ROM                                │
│                                    │
│ address ───────────────> data      │
│                                    │
│ read only                          │
└────────────────────────────────────┘


┌────────────────────────────────────┐
│ RAM                                │
│                                    │
│ address ──┐                         │
│           ├──────────────> data    │
│ write ────┤                         │
│ writeData ┘                         │
│                                    │
│ read + write                       │
└────────────────────────────────────┘
```

---

# 8. Project 9에서 simulation할 것

이번에는 딱 다음 정도면 충분합니다.

### ROM

```text
address 0 → expected data
address 1 → expected data
...
address 7 → expected data
```

### RAM

```text
초기 상태 확인
   ↓
address 0 ← 0xAA
address 1 ← 0x55
address 2 ← 0x12
   ↓
각 address read
   ↓
값 검증
```

그리고 중요한 것은:

```text
write
read
clock
```

의 timing 관계를 waveform에서 확인하는 것입니다.

---

## Project 9에서 굳이 하지 않을 것

이번 프로젝트의 목적을 벗어나므로 다음은 일단 넘어갑니다.

- Dual-port RAM
- true dual-port RAM
- byte enable
- RAM initialization의 FPGA vendor-specific 차이
- RAM inference 옵션
- 여러 clock domain RAM
- FIFO 내부 RAM 구조

이런 것은 나중에 필요해질 때 다루겠습니다.

---

### 다음 순서

우선 위 `SimpleRom`을 **컴파일하고 simulation**해 보겠습니다.

그 결과를 보내주시면 제가 결과를 확인한 뒤 바로 **RAM simulation**으로 넘어가겠습니다.

이렇게 하면 Project 9도 불필요하게 길어지지 않고 핵심만 빠르게 습득할 수 있습니다.

---

val ram = Mem(Bits(8 bits), wordCount = 4) 에서 wordCount 는 무슨 뜻인가?

---

`wordCount`는 **메모리에 저장할 수 있는 데이터의 개수(깊이, depth)**를 의미합니다.

```scala
val ram = Mem(Bits(8 bits), wordCount = 4)
```

를 두 부분으로 나누면:

```text
Bits(8 bits)  → 한 칸(word)에 저장할 데이터 크기
wordCount = 4 → 그런 칸이 몇 개 있는가
```

즉:

```text
             8 bits
          ┌──────────┐
address 0 │  0x??    │
          ├──────────┤
address 1 │  0x??    │
          ├──────────┤
address 2 │  0x??    │
          ├──────────┤
address 3 │  0x??    │
          └──────────┘
             ↑
          4 words
```

### 따라서 이 RAM은

- **Width:** 8 bits
- **Depth:** 4 words
- **총 용량:** 8 × 4 = **32 bits = 4 bytes**

입니다.

---

### 주소 폭은 왜 `2 bits`인가?

4개의 위치가 있으므로 주소는:

```text
00 → address 0
01 → address 1
10 → address 2
11 → address 3
```

따라서:

```scala
val address = in UInt(2 bits)
```

가 됩니다.

일반적으로:

```text
wordCount = N
address width = log2Up(N)
```

입니다.

예를 들어:

| wordCount | 주소 폭 | 주소 범위 |
|---:|---:|---|
| 2 | 1 bit | 0~1 |
| 4 | 2 bits | 0~3 |
| 8 | 3 bits | 0~7 |
| 16 | 4 bits | 0~15 |
| 256 | 8 bits | 0~255 |

즉 `wordCount`는 **데이터 폭(width)이 아니라 메모리의 개수(depth)**를 결정합니다.

그리고 여기서 `word`라는 표현은 꼭 CPU의 32-bit word 같은 의미가 아니라, **RAM에서 한 주소로 접근하는 하나의 데이터 항목**이라고 이해하면 됩니다.

---

좋습니다. **Project 9 — ROM + RAM**을 계속 진행하겠습니다.

ROM은 이미 시뮬레이션으로 확인했으므로, 이제 **RAM의 Write → Read**만 확실하게 확인하고 넘어가겠습니다.

# Project 9-2 — Simple RAM

먼저 우리가 사용했던 RAM은 다음과 같습니다.

```scala
import spinal.core._

class SimpleRam extends Component {
  val io = new Bundle {
    val write     = in Bool()
    val address   = in UInt(2 bits)
    val writeData = in Bits(8 bits)
    val readData  = out Bits(8 bits)
  }

  val ram = Mem(Bits(8 bits), wordCount = 4)

  when(io.write) {
    ram.write(
      address = io.address,
      data = io.writeData
    )
  }

  io.readData := ram.readAsync(io.address)
}
```

여기서 메모리 구조는:

```text
          8 bit
       ┌─────────┐
addr 0 │         │  0xAA
       ├─────────┤
addr 1 │         │  0xBB
       ├─────────┤
addr 2 │         │  0xCC
       ├─────────┤
addr 3 │         │  0xDD
       └─────────┘
          RAM
```

---

# 1. RAM Simulation

이번에는 단순하게

```text
Write 0xAA → address 0
Write 0xBB → address 1
Write 0xCC → address 2
Write 0xDD → address 3

그리고 다시 읽기
```

를 테스트하겠습니다.

```scala
import spinal.core._
import spinal.core.sim._

object SimpleRamSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleRam)
    .doSim { dut =>

      // --------------------------------
      // Write 0xAA -> address 0
      // --------------------------------
      dut.io.address #= 0
      dut.io.writeData #= 0xAA
      dut.io.write #= true

      sleep(1)

      dut.io.write #= false
      sleep(1)

      // --------------------------------
      // Write 0xBB -> address 1
      // --------------------------------
      dut.io.address #= 1
      dut.io.writeData #= 0xBB
      dut.io.write #= true

      sleep(1)

      dut.io.write #= false
      sleep(1)

      // --------------------------------
      // Write 0xCC -> address 2
      // --------------------------------
      dut.io.address #= 2
      dut.io.writeData #= 0xCC
      dut.io.write #= true

      sleep(1)

      dut.io.write #= false
      sleep(1)

      // --------------------------------
      // Write 0xDD -> address 3
      // --------------------------------
      dut.io.address #= 3
      dut.io.writeData #= 0xDD
      dut.io.write #= true

      sleep(1)

      dut.io.write #= false
      sleep(1)

      // --------------------------------
      // Read all addresses
      // --------------------------------
      for (address <- 0 until 4) {
        dut.io.address #= address
        sleep(1)

        println(
          f"address=$address%d readData=0x${dut.io.readData.toBigInt}%02X"
        )
      }
    }
}
```

## 예상 결과

정상이라면:

```text
[info] address=0 readData=0xAA
[info] address=1 readData=0xBB
[info] address=2 readData=0xCC
[info] address=3 readData=0xDD
```

가 나와야 합니다.

---

# 2. 이번 테스트에서 꼭 볼 것

이번에는 UART/FIFO 때처럼 clock scheduling을 복잡하게 생각할 필요가 없습니다.

현재 RAM의 read 방식이:

```scala
ram.readAsync(io.address)
```

이기 때문입니다.

즉:

```text
address = 2
      ↓
readAsync()
      ↓
readData = RAM[2]
```

입니다.

그리고 write는:

```scala
when(io.write) {
  ram.write(
    address = io.address,
    data = io.writeData
  )
}
```

이므로 `write=true`인 동안 지정된 address에 데이터를 기록합니다.

---

## 중요한 차이

이번 RAM은 **async read RAM**입니다.

따라서 이번 Project 9에서는 다음 두 가지를 구분하는 것이 핵심입니다.

```text
ROM
 └─ readAsync()
       ↓
   address → data
```

```text
RAM
 ├─ write()
 │    ↓
 │   저장
 │
 └─ readAsync()
      ↓
   address → data
```

---

### 한 가지 주의

이 RAM은 **학습용 Simple RAM**입니다.

실제 FPGA에서 RAM을 설계할 때는 흔히 **synchronous read RAM** 형태가 사용되며, 이 경우에는:

```text
address
   ↓
clock
   ↓
readData
```

처럼 한 단계의 clock latency가 생길 수 있습니다.

하지만 지금은 그쪽으로 깊게 들어가지 않고, **`Mem`, wordCount, write, readAsync`의 기본 개념을 확실히 잡는 것**이 목적입니다.

---

우선 위 `SimpleRamSim`을 실행해 주세요.

**출력 결과를 그대로 보내주시면**, 그 결과를 확인한 뒤 Project 9의 마지막 핵심인 **Async Read RAM vs Sync Read RAM**을 아주 짧게 정리하고 **Project 10 — Ready/Valid + Block Connection**으로 넘어가겠습니다.

---

좋습니다. 이번 결과는 **RAM 코드 자체보다 Simulation 방식에서 문제가 발생한 것**입니다.

```text
address=0 readData=0xB0
address=1 readData=0xB0
address=2 readData=0xB0
address=3 readData=0xB0
```

정상적인 결과가 아닙니다. 우리가 기대한 것은 `AA / BB / CC / DD`였죠.

## 핵심 원인

앞에서 제가 제시한 Simulation 코드에서 이 부분이 잘못되었습니다.

```scala
dut.io.write #= true
sleep(1)
dut.io.write #= false
```

**`sleep()`은 clock edge를 발생시키지 않습니다.**

그리고 `ram.write()`는 메모리에 실제로 write하는 동작이므로 **clock에 동기화된 write**가 필요합니다.

즉,

```text
write=true
   │
   │  clock edge 필요
   ↓
RAM에 실제 저장
```

입니다.

반면 read는 우리가 사용한

```scala
ram.readAsync(io.address)
```

이므로 clock 없이도 읽을 수 있습니다.

---

# 수정된 RAM Simulation

이번에는 `waitSampling()`을 사용해서 정확하게 테스트하겠습니다.

```scala
import spinal.core._
import spinal.core.sim._

object SimpleRamSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleRam)
    .doSim { dut =>

      // --------------------------------
      // Clock 생성
      // --------------------------------
      dut.clockDomain.forkStimulus(period = 10)

      // --------------------------------
      // Write 0xAA -> address 0
      // --------------------------------
      dut.io.address #= 0
      dut.io.writeData #= 0xAA
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xBB -> address 1
      // --------------------------------
      dut.io.address #= 1
      dut.io.writeData #= 0xBB
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xCC -> address 2
      // --------------------------------
      dut.io.address #= 2
      dut.io.writeData #= 0xCC
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xDD -> address 3
      // --------------------------------
      dut.io.address #= 3
      dut.io.writeData #= 0xDD
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Read all addresses
      // --------------------------------
      for (address <- 0 until 4) {
        dut.io.address #= address

        // readAsync이므로 clock edge가 필요하지 않음
        sleep(1)

        println(
          f"address=$address%d readData=0x${dut.io.readData.toBigInt}%02X"
        )
      }
    }
}
```

### 예상 결과

```text
[info] address=0 readData=0xAA
[info] address=1 readData=0xBB
[info] address=2 readData=0xCC
[info] address=3 readData=0xDD
```

---

# 이번에 아주 중요한 개념 하나

이번 실수는 오히려 **SpinalHDL Simulation을 이해하는 데 좋은 예제**입니다.

RAM의 두 동작을 분리해서 생각하면 됩니다.

### Write

```text
address
writeData
write=true
     │
     ↓
 CLOCK EDGE
     │
     ↓
RAM에 저장
```

### Read

현재 코드는 `readAsync()`이므로:

```text
address
   │
   ↓
readAsync()
   │
   ↓
readData
```

**clock이 필요 없습니다.**

즉:

| 동작 | 현재 RAM |
|---|---|
| Write | Clock 필요 |
| Read | Clock 불필요 |
| Read 방식 | Async |
| Memory depth | 4 words |
| Data width | 8 bits |

이것이 이번 Project 9에서 반드시 가져가야 할 핵심입니다.

그리고 이 부분은 제가 앞서 `sleep(1)`만 사용한 RAM 테스트 코드를 제시한 것이 잘못이었습니다. **SpinalHDL 1.15.0 기준으로 Simulation scheduling까지 고려하면 `write` 테스트에는 clock edge를 명시해야 합니다.**

수정된 코드를 실행해서 결과를 확인해 주세요. 정상적으로 `AA, BB, CC, DD`가 나오면 **Project 9의 ROM/RAM 핵심 개념은 완료**하고 바로 Project 10으로 넘어가겠습니다.

---

```
[info] [Progress] Start SimpleRam test simulation with seed 1668218169
[info] address=0 readData=0xAA
[info] address=1 readData=0xBB
[info] address=2 readData=0xCC
[info] address=3 readData=0xDD
[info] [Done] Simulation done in 19.718 ms
```
Sync Read RAM 은 중요한 부분이니 시뮬레이션 코드까지 진행하고, 그 후에
Async Read RAM vs Sync Read RAM**을 아주 짧게 정리하자.

---

좋습니다. 이번에는 **Sync Read RAM을 확실히 이해하고 시뮬레이션까지 확인한 다음 Project 10으로 넘어가겠습니다.**

그리고 방금 Async Read RAM 테스트도 정상적으로 통과했습니다.

```text
address=0 → 0xAA
address=1 → 0xBB
address=2 → 0xCC
address=3 → 0xDD
```

---

# Project 9-3 — Synchronous Read RAM

지금까지의 RAM은:

```scala
ram.readAsync(io.address)
```

였습니다.

이번에는 이것을:

```scala
ram.readSync(io.address)
```

로 바꿉니다.

가장 중요한 차이는 이것입니다.

### Async Read

```text
address ───────────────→ readData
             즉시
```

### Sync Read

```text
address ──→ [CLOCK] ──→ readData
```

즉, **Sync Read에서는 read address가 clock edge를 통해 RAM에 전달되고, 그 결과가 register를 거쳐 출력됩니다.**

---

# 1. SimpleSyncRam

SpinalHDL 1.15.0 기준으로 다음과 같이 작성하겠습니다.

```scala
import spinal.core._

class SimpleSyncRam extends Component {
  val io = new Bundle {
    val write     = in Bool()
    val address   = in UInt(2 bits)
    val writeData = in Bits(8 bits)
    val readData  = out Bits(8 bits)
  }

  val ram = Mem(Bits(8 bits), wordCount = 4)

  // Synchronous write
  when(io.write) {
    ram.write(
      address = io.address,
      data = io.writeData
    )
  }

  // Synchronous read
  io.readData := ram.readSync(io.address)
}
```

여기서 핵심적으로 바뀐 것은 딱 한 줄입니다.

```scala
// Async
io.readData := ram.readAsync(io.address)
```

↓

```scala
// Sync
io.readData := ram.readSync(io.address)
```

---

# 2. Simulation

이번에는 **clock cycle을 명확하게 관찰**하겠습니다.

먼저 RAM에:

```text
0 → AA
1 → BB
2 → CC
3 → DD
```

를 저장합니다.

그 후 주소를 하나씩 넣으면서 Sync Read의 동작을 확인합니다.

```scala
import spinal.core._
import spinal.core.sim._

object SimpleSyncRamSim extends App {
  SimConfig
    .withWave
    .compile(new SimpleSyncRam)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // --------------------------------
      // Write 0xAA -> address 0
      // --------------------------------
      dut.io.address #= 0
      dut.io.writeData #= 0xAA
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xBB -> address 1
      // --------------------------------
      dut.io.address #= 1
      dut.io.writeData #= 0xBB
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xCC -> address 2
      // --------------------------------
      dut.io.address #= 2
      dut.io.writeData #= 0xCC
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Write 0xDD -> address 3
      // --------------------------------
      dut.io.address #= 3
      dut.io.writeData #= 0xDD
      dut.io.write #= true

      dut.clockDomain.waitSampling()

      dut.io.write #= false


      // --------------------------------
      // Sync Read
      // --------------------------------
      for (address <- 0 until 4) {

        dut.io.address #= address

        println(
          f"Before clock: address=$address%d readData=0x${dut.io.readData.toBigInt}%02X"
        )

        dut.clockDomain.waitSampling()

        println(
          f"After  clock: address=$address%d readData=0x${dut.io.readData.toBigInt}%02X"
        )
      }
    }
}
```

---

# 3. 여기서 관찰해야 하는 것

예를 들어 address `2`를 읽는다면:

```text
address = 2
     │
     │
     │  아직 clock 전
     ↓
readData는 이전 값
     
     │
     │ CLOCK
     ↓
     
RAM[2] = 0xCC
     │
     ↓
readData = 0xCC
```

따라서 Async RAM과 달리:

```scala
dut.io.address #= 2
sleep(1)
```

만 해서는 **Sync Read 결과를 기대하면 안 됩니다.**

반드시:

```scala
dut.io.address #= 2
dut.clockDomain.waitSampling()
```

처럼 clock edge를 기다려야 합니다.

---

# 4. Async vs Sync

이제 두 RAM을 비교하면 아주 명확합니다.

| | Async Read RAM | Sync Read RAM |
|---|---|---|
| Read API | `readAsync()` | `readSync()` |
| Address 변화 | 즉시 반영 | Clock 필요 |
| Read clock | 필요 없음 | 필요 |
| 출력 | 조합적으로 변화 | Clock에 맞춰 변화 |
| FPGA RAM 구현과의 관계 | 제한적일 수 있음 | 실제 Block RAM에 적합한 경우가 많음 |

개념적으로 보면:

```text
Async
────────────────────

address ────────→ RAM ────────→ data
                  즉시
```

반면:

```text
Sync
────────────────────

address ───→ RAM ───→ Register ───→ data
              ↑
            clock
```

입니다.

---

# 5. 특히 중요한 부분 — "1 Cycle Latency"

Sync Read에서 앞으로 매우 중요하게 사용하게 될 개념이 **latency**입니다.

예를 들어:

```text
Cycle 10
address = 2

        ↓ clock

Cycle 11
readData = RAM[2]
```

처럼 생각할 수 있습니다.

즉:

> **주소를 넣고 clock을 통과한 후 데이터가 나온다.**

이 개념은 앞으로 배울 **Ready/Valid, Block Connection, Pipeline**에서 계속 등장합니다.

특히 나중에:

```text
CPU
 ↓
RAM
 ↓
Data
```

같은 구조를 만들면 이 **1-cycle memory latency**를 반드시 고려해야 합니다.

---

## 이번 테스트에서 중요한 점

이번에는 RAM을 초기화하지 않았기 때문에 **읽기 전에 반드시 write를 먼저 수행**하도록 테스트했습니다.

그리고 우리가 확인하려는 것은:

```text
Write
  ↓
RAM 저장
  ↓
Address 입력
  ↓
Clock
  ↓
Sync Read Data
```

전체 과정입니다.

---

### 이제 실행해 주세요.

특히 출력에서

```text
Before clock
After clock
```

의 값이 어떻게 변하는지 확인해 봅시다.

그 결과를 보내주시면 **Sync Read의 clock/latency 동작까지 검증한 것으로 Project 9를 완전히 마무리**하고,

# Project 10 — Ready/Valid + Block Connection

으로 넘어가겠습니다.

---

```
[info] [Progress] Start SimpleSyncRam test simulation with seed 221408667
[info] Before clock: address=0 readData=0x6C
[info] After  clock: address=0 readData=0x6C
[info] Before clock: address=1 readData=0x6C
[info] After  clock: address=1 readData=0xAA
[info] Before clock: address=2 readData=0xAA
[info] After  clock: address=2 readData=0xBB
[info] Before clock: address=3 readData=0xBB
[info] After  clock: address=3 readData=0xCC
[info] [Done] Simulation done in 31.977 ms
```

---

네, **결과를 보면 Sync Read RAM의 핵심 동작이 정확히 확인되었습니다.** 다만 여기서 한 가지 중요한 점을 짚고 Project 9를 마무리하는 것이 좋겠습니다.

## 1. 결과 해석

출력:

```text
Before clock: address=0 readData=0x6C
After  clock: address=0 readData=0x6C

Before clock: address=1 readData=0x6C
After  clock: address=1 readData=0xAA

Before clock: address=2 readData=0xAA
After  clock: address=2 readData=0xBB

Before clock: address=3 readData=0xBB
After  clock: address=3 readData=0xCC
```

이것은 정확히 **1-cycle read latency**가 발생하고 있다는 뜻입니다.

예를 들어:

```text
address = 1
     │
     │ clock
     ▼
readData = RAM[0] = AA
```

그 다음:

```text
address = 2
     │
     │ clock
     ▼
readData = RAM[1] = BB
```

즉, 우리가 현재 작성한 코드에서 `readSync()`는 **주소를 넣은 바로 그 sampling 시점 이후에 기대 주소의 데이터가 보이는 형태가 아니라, 관찰 코드의 scheduling 기준으로 이전 주소의 데이터가 출력되는 것**을 확인했습니다.

---

# 2. 왜 address=0에서 AA가 안 나왔을까?

이 부분이 중요합니다.

첫 번째:

```text
Before clock: address=0 readData=0x6C
After  clock: address=0 readData=0x6C
```

그리고 다음:

```text
Before clock: address=1 readData=0x6C
After  clock: address=1 readData=0xAA
```

즉 `0xAA`는 **address=0을 넣은 뒤 다음 read 단계에서 관찰**되었습니다.

따라서 이번 테스트 결과를 단순히:

> "`readSync()`는 항상 정확히 1 clock 뒤에 나온다."

라고만 외우는 것보다,

> **Sync Read는 clocked memory read이며, read address와 read data 사이에 clock-domain latency가 존재한다.**

라고 이해하는 것이 좋습니다.

이것이 앞으로 매우 중요합니다.

---

# 3. Async와 Sync를 최종 정리

### Async Read

```text
address ─────────→ RAM ─────────→ data
                     즉시
```

우리가 테스트했던 결과:

```text
address=0 → AA
address=1 → BB
address=2 → CC
address=3 → DD
```

---

### Sync Read

```text
address ──→ RAM
             │
           CLOCK
             │
             ▼
           readData
```

따라서 read address를 바꾼 뒤 **clock을 기준으로 data가 업데이트**됩니다.

이번 시뮬레이션에서 이것을 실제 waveform/출력으로 확인했습니다.

---

# 4. Project 9 완료

이제 Project 9에서 배워야 할 핵심은 충분히 다뤘습니다.

```text
Project 9 — ROM + RAM
│
├─ Mem
│
├─ wordCount
│
├─ Data Width
│
├─ Address Width
│
├─ ROM
│   └─ init()
│
├─ RAM
│   └─ write()
│
├─ Async Read
│   └─ readAsync()
│
└─ Sync Read
    └─ readSync()
        └─ clock / latency
```

특히 이번 Project 9에서는 단순히 API를 외운 것이 아니라,

**Simulation에서 clock scheduling이 실제 memory behavior에 어떤 영향을 주는지까지 확인했습니다.**

이 부분은 앞으로 상당히 도움이 됩니다.

---

# 다음 단계 — Project 10

이제 예정대로:

# **Project 10 — Ready / Valid + Block Connection**

으로 넘어가겠습니다.

이번 프로젝트의 핵심은 지금까지 배운 개별 블록들을 **실제로 연결하는 방법**입니다.

구조는 간단하게:

```text
Producer
   │
   │ data + valid
   ▼
┌─────────────┐
│   Process   │
└─────────────┘
   │
   │ data + ready
   ▼
Consumer
```

그리고 가장 중요한 개념:

```text
valid = "나는 지금 데이터를 가지고 있다"
ready = "나는 지금 데이터를 받을 수 있다"

transfer = valid && ready
```

를 배웁니다.

이것은 이후 **FIFO → Block Connection → ALU → CPU → Pipeline**으로 이어지는 매우 중요한 기반입니다.

**Project 10에서는 먼저 `valid/ready`의 가장 작은 예제를 만들고, 바로 Simulation으로 handshake가 실제로 언제 발생하는지 확인하는 방식으로 진행하겠습니다.**
