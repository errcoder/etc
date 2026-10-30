좋습니다. **12-2 Register File까지 정상 통과**했습니다. ✅

현재까지:

```text
12-1 Simple Bus       ✓
12-2 Register File    ✓
```

이제 핵심인 **12-3 — Bus + Register File + ALU Datapath**로 갑니다.

# Project 12-3 — Mini CPU Datapath

이번에는 CPU의 실제 데이터 흐름을 하나로 연결합니다.

```text
             ┌──────────────┐
             │ RegisterFile │
             │              │
 readAddr ──→│ R0 R1 R2 R3  │
             └───┬──────┬───┘
                 │      │
                 ▼      ▼
                A        B
                 │      │
                 └──┬───┘
                    ▼
                  ┌────┐
 opcode ─────────→│ALU │
                  └──┬─┘
                     │
                     ▼
                  Result
                     │
                     ▼
               RegisterFile
```

## 핵심 아이디어

예를 들어 CPU가 다음 동작을 한다고 생각합니다.

```text
R2 = R0 + R1
```

그러면 하드웨어 데이터 흐름은:

```text
R0 ──→ ALU A
       │
R1 ──→ ALU B
       │
       ▼
      ADD
       │
       ▼
      R2
```

입니다.

### Datapath 코드

```scala id="6j5b7r"
import spinal.core._

class MiniCpuDatapath extends Component {
  val io = new Bundle {
    val readA      = in UInt(2 bits)
    val readB      = in UInt(2 bits)

    val write      = in Bool()
    val writeAddr  = in UInt(2 bits)
    val writeData  = in Bits(8 bits)

    val opcode     = in UInt(2 bits)

    val result     = out Bits(8 bits)
  }

  val regFile = new RegisterFile
  val alu      = new SimpleAlu

  // Register File read
  regFile.io.readAddress := io.readA

  // ALU A
  alu.io.a := regFile.io.readData.asUInt

  // 두 번째 operand를 위해 별도의 read가 필요하므로
  // 현재 4-register 구조에서는 간단히 두 번째 값을
  // 외부 write/read 경로와 분리하지 않고 구성할 수 없습니다.
}
```

여기서 잠깐 멈추는 것이 좋습니다.

현재 만든 `RegisterFile`은 **read port가 하나뿐**입니다.

그런데 일반적인 ALU 연산:

```text
R2 = R0 + R1
```

을 하려면 동시에:

```text
R0 → operand A
R1 → operand B
```

두 값을 읽어야 합니다.

즉, CPU datapath를 만들면서 자연스럽게 **Dual-Port Read Register File**이라는 중요한 개념이 등장합니다.

그래서 기존 `RegisterFile`을 억지로 확장하지 않고, 다음 단계에서:

```text
Register File
 ├── readAddressA → readDataA
 └── readAddressB → readDataB
```

형태로 **2-read-port Register File**을 만들겠습니다.

이게 실제 CPU 구조에 훨씬 가깝습니다.

---

---

좋습니다. 여기서 **Dual-Read Register File**로 확장하겠습니다.

# Project 12-3 — Dual-Read Register File

CPU의 일반적인 연산:

```text
R2 = R0 + R1
```

을 수행하려면 같은 시점에 R0와 R1을 읽어야 합니다.

```text
             ┌────────────────┐
readAddrA ──→│                │──→ readDataA
             │ Register File  │
readAddrB ──→│                │──→ readDataB
             │                │
writeAddr ──→│                │
writeData ──→│                │
             └────────────────┘
```

## 코드

```scala
import spinal.core._

class DualReadRegisterFile extends Component {
  val io = new Bundle {
    val writeEnable  = in Bool()
    val writeAddress = in UInt(2 bits)
    val writeData    = in Bits(8 bits)

    val readAddressA = in UInt(2 bits)
    val readDataA    = out Bits(8 bits)

    val readAddressB = in UInt(2 bits)
    val readDataB    = out Bits(8 bits)
  }

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  // Synchronous write
  when(io.writeEnable) {
    registers(io.writeAddress) := io.writeData
  }

  // Two independent combinational read ports
  io.readDataA := registers(io.readAddressA)
  io.readDataB := registers(io.readAddressB)
}
```

이제 동시에:

```text
readAddressA = 0
readAddressB = 1
```

이면:

```text
readDataA = R0
readDataB = R1
```

가 됩니다.

---

# Simulation

```scala
import spinal.core._
import spinal.core.sim._

object DualReadRegisterFileSim extends App {
  SimConfig
    .withWave
    .compile(new DualReadRegisterFile)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      dut.io.writeEnable  #= false
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0
      dut.io.readAddressA #= 0
      dut.io.readAddressB #= 0

      dut.clockDomain.waitSampling()

      // R0 = 0x10
      dut.io.writeEnable  #= true
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0x10
      dut.clockDomain.waitSampling()

      // R1 = 0x20
      dut.io.writeAddress #= 1
      dut.io.writeData    #= 0x20
      dut.clockDomain.waitSampling()

      // R2 = 0x30
      dut.io.writeAddress #= 2
      dut.io.writeData    #= 0x30
      dut.clockDomain.waitSampling()

      dut.io.writeEnable #= false

      // 동시에 R0, R1 읽기
      dut.io.readAddressA #= 0
      dut.io.readAddressB #= 1

      sleep(1)

      println(
        f"Read A=0x${dut.io.readDataA.toBigInt}%02X " +
        f"Read B=0x${dut.io.readDataB.toBigInt}%02X"
      )

      assert(dut.io.readDataA.toBigInt == 0x10)
      assert(dut.io.readDataB.toBigInt == 0x20)

      println("DualReadRegisterFile test passed.")
    }
}
```

예상:

```text
Read A=0x10 Read B=0x20
DualReadRegisterFile test passed.
```

이제 CPU datapath를 만들기 위한 준비가 끝났습니다.

```text
R0 ─────┐
        ├──→ ALU ──→ R2
R1 ─────┘
```

다음 단계에서 **DualReadRegisterFile + SimpleAlu를 실제로 연결해서 `R2 = R0 + R1`을 수행하는 Mini CPU Datapath**를 만들겠습니다.

---

---

좋습니다. **Dual-Read Register File까지 정상 통과**했습니다. ✅

현재 Project 12 진행 상황:

```text
12-1 Simple Bus              ✓
12-2 Register File           ✓
12-3 Dual-Read RegisterFile  ✓
```

이제 드디어 **ALU + Register File을 연결**합니다.

# Project 12-4 — Mini CPU Datapath

목표는 아주 간단합니다.

```text
R2 = R0 + R1
```

하드웨어는:

```text
       ┌───────────────┐
R0 ───→│               │
       │ Register File │──→ A
R1 ───→│               │──→ B
       └───────────────┘
                 │
                 ▼
              ┌─────┐
       ADD ──→│ ALU │
              └──┬──┘
                 │
                 ▼
                R2
```

## MiniCpuDatapath

```scala id="t7g3wq"
import spinal.core._

class MiniCpuDatapath extends Component {
  val io = new Bundle {
    val readAddressA = in UInt(2 bits)
    val readAddressB = in UInt(2 bits)

    val writeEnable  = in Bool()
    val writeAddress = in UInt(2 bits)
    val writeData    = in Bits(8 bits)

    val opcode       = in UInt(2 bits)

    val result       = out Bits(8 bits)
  }

  val regFile = new DualReadRegisterFile
  val alu      = new SimpleAlu

  // Register File
  regFile.io.readAddressA := io.readAddressA
  regFile.io.readAddressB := io.readAddressB

  regFile.io.writeEnable  := io.writeEnable
  regFile.io.writeAddress := io.writeAddress
  regFile.io.writeData    := io.writeData

  // ALU operands
  alu.io.a := regFile.io.readDataA.asUInt
  alu.io.b := regFile.io.readDataB.asUInt
  alu.io.opcode := io.opcode

  // ALU result
  io.result := alu.io.result.asBits
}
```

여기서 중요한 것은 **ALU 결과를 바로 register에 저장하지 않고 외부 `writeData` 경로로 되돌릴 수 있도록 만든 것**입니다.

즉 testbench에서:

```text
R0 = 0x10
R1 = 0x20
```

을 만들어 놓고,

```text
readAddressA = 0
readAddressB = 1
opcode = ADD
```

를 설정하면:

```text
ALU result = 0x30
```

이 됩니다.

그리고 이 값을:

```text
writeAddress = 2
writeData = ALU result
writeEnable = true
```

로 다음 clock에 저장하면:

```text
R2 = 0x30
```

이 됩니다.

### 전체 데이터 흐름

```text
R0 ──┐
     │
     ▼
    ALU ─────────→ writeData
     ▲                  │
     │                  ▼
R1 ──┘                R2
```

이제 다음 단계에서 이 구조를 **한 번의 ADD instruction처럼 동작시키는 simulation**을 작성해서

```text
R0 = 0x10
R1 = 0x20
R2 = 0x30
```

을 실제로 검증하겠습니다.

---

---

좋습니다. 여기서 **Dual-Read Register File**로 확장하겠습니다.

# Project 12-3 — Dual-Read Register File

CPU의 일반적인 연산:

```text
R2 = R0 + R1
```

을 수행하려면 같은 시점에 R0와 R1을 읽어야 합니다.

```text
             ┌────────────────┐
readAddrA ──→│                │──→ readDataA
             │ Register File  │
readAddrB ──→│                │──→ readDataB
             │                │
writeAddr ──→│                │
writeData ──→│                │
             └────────────────┘
```

## 코드

```scala
import spinal.core._

class DualReadRegisterFile extends Component {
  val io = new Bundle {
    val writeEnable  = in Bool()
    val writeAddress = in UInt(2 bits)
    val writeData    = in Bits(8 bits)

    val readAddressA = in UInt(2 bits)
    val readDataA    = out Bits(8 bits)

    val readAddressB = in UInt(2 bits)
    val readDataB    = out Bits(8 bits)
  }

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  // Synchronous write
  when(io.writeEnable) {
    registers(io.writeAddress) := io.writeData
  }

  // Two independent combinational read ports
  io.readDataA := registers(io.readAddressA)
  io.readDataB := registers(io.readAddressB)
}
```

이제 동시에:

```text
readAddressA = 0
readAddressB = 1
```

이면:

```text
readDataA = R0
readDataB = R1
```

가 됩니다.

---

# Simulation

```scala
import spinal.core._
import spinal.core.sim._

object DualReadRegisterFileSim extends App {
  SimConfig
    .withWave
    .compile(new DualReadRegisterFile)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      dut.io.writeEnable  #= false
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0
      dut.io.readAddressA #= 0
      dut.io.readAddressB #= 0

      dut.clockDomain.waitSampling()

      // R0 = 0x10
      dut.io.writeEnable  #= true
      dut.io.writeAddress #= 0
      dut.io.writeData    #= 0x10
      dut.clockDomain.waitSampling()

      // R1 = 0x20
      dut.io.writeAddress #= 1
      dut.io.writeData    #= 0x20
      dut.clockDomain.waitSampling()

      // R2 = 0x30
      dut.io.writeAddress #= 2
      dut.io.writeData    #= 0x30
      dut.clockDomain.waitSampling()

      dut.io.writeEnable #= false

      // 동시에 R0, R1 읽기
      dut.io.readAddressA #= 0
      dut.io.readAddressB #= 1

      sleep(1)

      println(
        f"Read A=0x${dut.io.readDataA.toBigInt}%02X " +
        f"Read B=0x${dut.io.readDataB.toBigInt}%02X"
      )

      assert(dut.io.readDataA.toBigInt == 0x10)
      assert(dut.io.readDataB.toBigInt == 0x20)

      println("DualReadRegisterFile test passed.")
    }
}
```

예상:

```text
Read A=0x10 Read B=0x20
DualReadRegisterFile test passed.
```

이제 CPU datapath를 만들기 위한 준비가 끝났습니다.

```text
R0 ─────┐
        ├──→ ALU ──→ R2
R1 ─────┘
```

다음 단계에서 **DualReadRegisterFile + SimpleAlu를 실제로 연결해서 `R2 = R0 + R1`을 수행하는 Mini CPU Datapath**를 만들겠습니다.

---

---


