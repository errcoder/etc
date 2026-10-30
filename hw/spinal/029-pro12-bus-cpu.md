좋습니다. 👍  
이번 DataProcessor simulation은 사용자가 직접 수정하는 것으로 하고 **Project 11은 여기서 통과 처리**하겠습니다.

앞으로는 같은 simulation scheduling 실수를 반복하지 않도록 더 신중하게 진행하겠습니다.

# Project 12 — Bus + Mini CPU

이번 프로젝트부터 한 단계 올라갑니다.

목표 구조:

```text
              ┌──────────────┐
              │    Mini CPU  │
              │              │
   Bus ──────→│   Register   │
              │      │       │
              │      ▼       │
              │     ALU      │
              │      │       │
              │      ▼       │
              │   Register   │
              └──────────────┘
```

이번에는 **실제 CPU 전체를 만드는 것이 아니라**, CPU의 핵심 개념만 압축해서 익힙니다.

### Project 12 학습 순서

```text
12-1  Simple Bus
       ↓
12-2  Register File
       ↓
12-3  ALU + Register + Bus
       ↓
12-4  아주 작은 Instruction
       ↓
12-5  Mini CPU
```

가장 먼저 **12-1 Simple Bus**부터 시작하겠습니다.

Bus에는 최소한 다음 신호가 필요합니다.

```text
address
write
writeData
readData
```

그리고 이번 프로젝트에서 중요한 것은 **Bus가 단순한 신호 묶음이 아니라, 여러 블록 사이의 데이터 통로 역할을 한다는 것**입니다.

다음 단계에서 `Bus`를 SpinalHDL의 `Bundle`로 정의하고, 간단한 Register 장치에 연결해 보겠습니다.

---

좋습니다. **Project 12-1 — Simple Bus**부터 시작하겠습니다.

이번에는 CPU를 서두르지 않고 **Bus 인터페이스 자체를 정확히 이해**합니다.

# 12-1. Simple Bus

먼저 아주 작은 메모리 맵을 만들겠습니다.

```text
address 0x00 → Register A
address 0x04 → Register B
address 0x08 → Register C
```

Bus는 다음과 같습니다.

```text
              SimpleBus
        ┌──────────────────┐
address│                  │
write  │                  │
writeData ────────────────→│
readData│←────────────────│
        └──────────────────┘
```

## 1. Bus Bundle

```scala
import spinal.core._

case class SimpleBus() extends Bundle {
  val address   = UInt(8 bits)
  val write     = Bool()
  val writeData = Bits(8 bits)
  val readData  = Bits(8 bits)
}
```

여기서 `Bundle`은 **하드웨어 블록 사이에서 함께 전달되는 신호들의 묶음**입니다.

그리고 이것을 `master/slave` 방향까지 포함하는 인터페이스로 만들 수도 있습니다.

```scala
case class SimpleBus() extends Bundle with IMasterSlave {
  val address   = UInt(8 bits)
  val write     = Bool()
  val writeData = Bits(8 bits)
  val readData  = Bits(8 bits)

  override def asMaster(): Unit = {
    out(address, write, writeData)
    in(readData)
  }
}
```

이번 프로젝트에서는 이 형태를 사용하겠습니다.

---

# 2. Bus Slave

Bus에 연결되는 아주 작은 장치를 만듭니다.

```scala
import spinal.core._

class BusRegisters extends Component {
  val io = new Bundle {
    val bus = slave(SimpleBus())
  }

  val regA = Reg(Bits(8 bits)) init(0)
  val regB = Reg(Bits(8 bits)) init(0)
  val regC = Reg(Bits(8 bits)) init(0)

  // Write
  when(io.bus.write) {
    switch(io.bus.address) {
      is(0x00) {
        regA := io.bus.writeData
      }

      is(0x04) {
        regB := io.bus.writeData
      }

      is(0x08) {
        regC := io.bus.writeData
      }
    }
  }

  // Read
  io.bus.readData := 0

  switch(io.bus.address) {
    is(0x00) {
      io.bus.readData := regA
    }

    is(0x04) {
      io.bus.readData := regB
    }

    is(0x08) {
      io.bus.readData := regC
    }
  }
}
```

전체 구조는:

```text
                 BusRegisters
              ┌───────────────┐
              │               │
address ─────→│ address decode│
write   ─────→│               │
writeData ───→│      │        │
              │      ▼        │
              │ ┌───┐ ┌───┐ ┌───┐
              │ │ A │ │ B │ │ C │
              │ └───┘ └───┘ └───┘
              │      │
readData  ←───│──────┘
              └───────────────┘
```

### 핵심

예를 들어:

```text
address = 0x04
write   = 1
writeData = 0x55
```

이면:

```text
regB ← 0x55
```

가 됩니다.

반대로:

```text
address = 0x04
write = 0
```

이면:

```text
readData = regB
```

입니다.

즉 **address decoder**가 CPU의 주소를 보고 어느 register를 선택할지 결정합니다.

---

## 3. 여기서 중요한 구조

이번 Bus는 사실상 다음 두 경로를 갖습니다.

### Write path

```text
CPU
 │
 ├─ address
 ├─ write
 └─ writeData
       │
       ▼
 Address Decoder
       │
       ▼
 Register
```

### Read path

```text
Register
    │
    ▼
Address Decoder
    │
    ▼
readData
    │
    ▼
CPU
```

그리고 **write는 clocked**, **read는 현재 코드에서는 combinational**입니다.

이 차이도 Project 9에서 했던 RAM과 연결되는 중요한 개념입니다.

다음은 이 BusRegisters를 실제로 **write/read simulation**해서 주소 `0x00`, `0x04`, `0x08`이 정확히 동작하는지 검증하겠습니다.

---

좋습니다. **Project 12-1의 BusRegisters simulation**으로 바로 검증하겠습니다.

이번에는 특히 **write는 clock edge가 필요하고, read는 combinational**이라는 점을 확인합니다.

# 12-1 Simulation — Bus Read / Write

```scala
import spinal.core._
import spinal.core.sim._

object BusRegistersSim extends App {
  SimConfig
    .withWave
    .compile(new BusRegisters)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // --------------------------------------------------
      // Initial
      // --------------------------------------------------

      dut.io.bus.address   #= 0
      dut.io.bus.write     #= false
      dut.io.bus.writeData #= 0

      dut.clockDomain.waitSampling()

      // --------------------------------------------------
      // Write 0x11 -> Register A (0x00)
      // --------------------------------------------------

      dut.io.bus.address   #= 0x00
      dut.io.bus.write     #= true
      dut.io.bus.writeData #= 0x11

      dut.clockDomain.waitSampling()

      // Write transaction 종료
      dut.io.bus.write #= false

      // --------------------------------------------------
      // Write 0x22 -> Register B (0x04)
      // --------------------------------------------------

      dut.io.bus.address   #= 0x04
      dut.io.bus.write     #= true
      dut.io.bus.writeData #= 0x22

      dut.clockDomain.waitSampling()

      dut.io.bus.write #= false

      // --------------------------------------------------
      // Write 0x33 -> Register C (0x08)
      // --------------------------------------------------

      dut.io.bus.address   #= 0x08
      dut.io.bus.write     #= true
      dut.io.bus.writeData #= 0x33

      dut.clockDomain.waitSampling()

      dut.io.bus.write #= false

      // --------------------------------------------------
      // Read Register A
      // --------------------------------------------------

      dut.io.bus.address #= 0x00

      sleep(1)

      println(
        f"Read A: address=0x00 readData=0x${dut.io.bus.readData.toBigInt}%02X"
      )

      assert(
        dut.io.bus.readData.toBigInt == 0x11,
        "Register A must contain 0x11"
      )

      // --------------------------------------------------
      // Read Register B
      // --------------------------------------------------

      dut.io.bus.address #= 0x04

      sleep(1)

      println(
        f"Read B: address=0x04 readData=0x${dut.io.bus.readData.toBigInt}%02X"
      )

      assert(
        dut.io.bus.readData.toBigInt == 0x22,
        "Register B must contain 0x22"
      )

      // --------------------------------------------------
      // Read Register C
      // --------------------------------------------------

      dut.io.bus.address #= 0x08

      sleep(1)

      println(
        f"Read C: address=0x08 readData=0x${dut.io.bus.readData.toBigInt}%02X"
      )

      assert(
        dut.io.bus.readData.toBigInt == 0x33,
        "Register C must contain 0x33"
      )

      println("BusRegisters test passed.")
    }
}
```

예상 결과:

```text
Read A: address=0x00 readData=0x11
Read B: address=0x04 readData=0x22
Read C: address=0x08 readData=0x33
BusRegisters test passed.
```

## 이번 simulation에서 확인하는 것

### Write

```text
address + write + writeData
             │
             ▼
        clock edge
             │
             ▼
          Register
```

따라서 write에는:

```scala
dut.clockDomain.waitSampling()
```

이 필요합니다.

### Read

현재 `BusRegisters`에서는:

```scala
io.bus.readData := ...
```

가 combinational logic이므로:

```text
address 변경
    ↓
address decoder
    ↓
readData
```

입니다.

그래서 read에서는 clock을 기다리지 않고:

```scala
dut.io.bus.address #= 0x04
sleep(1)
```

후 `readData`를 확인합니다.

---

이 테스트가 통과하면 **Bus + Address Decode**까지 완료입니다.

다음 **12-2에서는 Register File**을 만들어서 `Bus → Register File → ALU`로 연결하겠습니다.

---

---

좋습니다. **Project 12-1 — Simple Bus + Address Decode 완료**입니다. ✅

검증된 내용:

```text
0x00 → Register A → 0x11
0x04 → Register B → 0x22
0x08 → Register C → 0x33
```

그리고 중요한 동작도 확인했습니다.

```text
Write → clock 필요
Read  → 현재 구조에서는 combinational
```

이제 바로 **Project 12-2 — Register File**로 넘어가겠습니다.

---

# Project 12-2 — Register File

CPU에서는 여러 개의 register를 사용합니다.

이번에는 아주 작은 **4 × 8-bit Register File**을 만듭니다.

```text
        writeAddress
             │
        writeData
             │
             ▼
       ┌───────────┐
       │ Register  │
       │   File    │
       │           │
       │ R0 = 00   │
       │ R1 = 00   │
       │ R2 = 00   │
       │ R3 = 00   │
       └─────┬─────┘
             │
       readAddress
             │
             ▼
         readData
```

인터페이스는:

```scala id="3j5x7s"
import spinal.core._

class RegisterFile extends Component {
  val io = new Bundle {
    val writeEnable  = in Bool()
    val writeAddress = in UInt(2 bits)
    val writeData    = in Bits(8 bits)

    val readAddress  = in UInt(2 bits)
    val readData     = out Bits(8 bits)
  }

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  when(io.writeEnable) {
    registers(io.writeAddress) := io.writeData
  }

  io.readData := registers(io.readAddress)
}
```

### 동작

예를 들어:

```text
writeEnable  = 1
writeAddress = 2
writeData    = 0x55
```

에서 clock edge가 발생하면:

```text
R2 = 0x55
```

가 됩니다.

그 후:

```text
readAddress = 2
```

로 설정하면:

```text
readData = 0x55
```

입니다.

즉 현재 Register File은:

```text
Write → synchronous
Read  → combinational
```

구조입니다.

이번에는 코드를 먼저 simulation으로 검증하겠습니다. 특히 **R0~R3 각각에 값을 쓰고 다시 읽는 것**까지 확인한 뒤 다음 단계로 넘어가겠습니다.

---

---

좋습니다. **12-2 Register File simulation**으로 바로 검증합니다.

```scala id="8a5v4k"
import spinal.core._
import spinal.core.sim._

object RegisterFileSim extends App {
  SimConfig
    .withWave
    .compile(new RegisterFile)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // Initial
      dut.io.writeEnable  #= false
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0
      dut.io.readAddress  #= 0

      dut.clockDomain.waitSampling()

      // --------------------------------------------------
      // Write R0 ~ R3
      // --------------------------------------------------

      for (address <- 0 until 4) {
        dut.io.writeEnable  #= true
        dut.io.writeAddress #= address
        dut.io.writeData    #= (0x10 + address)

        dut.clockDomain.waitSampling()
      }

      dut.io.writeEnable #= false

      // --------------------------------------------------
      // Read R0 ~ R3
      // --------------------------------------------------

      for (address <- 0 until 4) {
        dut.io.readAddress #= address

        // Combinational read propagation
        sleep(1)

        val actual = dut.io.readData.toBigInt.intValue
        val expected = 0x10 + address

        println(
          f"Read R$address%d: data=0x$actual%02X"
        )

        assert(
          actual == expected,
          f"R$address%d: expected=0x$expected%02X, actual=0x$actual%02X"
        )
      }

      println("RegisterFile test passed.")
    }
}
```

예상 결과:

```text
Read R0: data=0x10
Read R1: data=0x11
Read R2: data=0x12
Read R3: data=0x13
RegisterFile test passed.
```

이번 단계의 핵심은 간단합니다.

```text
writeAddress + writeData
        │
        ▼
    clock edge
        │
        ▼
    Register
        │
        ▼
   readAddress
        │
        ▼
     readData
```

이 테스트가 통과하면 **12-2 Register File 완료**입니다.

다음은 드디어 이것들을 연결해서:

```text
Bus
 │
 ▼
Register File
 │
 ▼
 ALU
 │
 ▼
Register File
```

형태의 **작은 CPU datapath**를 만들겠습니다.
 
goto chapter30


