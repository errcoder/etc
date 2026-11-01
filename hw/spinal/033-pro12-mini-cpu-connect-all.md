좋습니다. 이제 **12-9 — Mini CPU 전체 연결**로 갑니다.

이번 단계가 Project 12의 핵심입니다.

지금까지 검증한 블록들을 연결합니다.

```text
        ┌────────┐
        │   PC   │
        └───┬────┘
            │
            ▼
     ┌─────────────┐
     │ Instruction │
     │     ROM     │
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │   Decoder   │
     └──────┬──────┘
            │
       ┌────┴─────┐
       ▼          ▼
    rs1/rs2       rd
       │
       ▼
 ┌──────────────┐
 │ RegisterFile │
 └──────┬───────┘
        │
        ▼
     ┌─────┐
     │ ALU │
     └──┬──┘
        │
        └──────> Write Back
```

## 12-9 MiniCpu

기존 블록들을 그대로 재사용합니다.

```scala
import spinal.core._

class MiniCpu extends Component {
  val io = new Bundle {
    // 외부에서 register 초기화
    val loadEnable  = in Bool()
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    val pc          = out UInt(4 bits)
    val instruction = out Bits(16 bits)
    val result      = out Bits(8 bits)
  }

  val fetch   = new FetchUnit
  val decoder = new InstructionDecoder
  val regFile = new DualReadRegisterFile
  val alu     = new SimpleAlu

  // --------------------------------------------------
  // Fetch
  // --------------------------------------------------

  fetch.io.enable := True

  io.pc          := fetch.io.pc
  io.instruction := fetch.io.instruction

  // --------------------------------------------------
  // Decode
  // --------------------------------------------------

  decoder.io.instruction := fetch.io.instruction

  // --------------------------------------------------
  // Register File Read
  // --------------------------------------------------

  regFile.io.readAddressA := decoder.io.rs1
  regFile.io.readAddressB := decoder.io.rs2

  // --------------------------------------------------
  // ALU
  // --------------------------------------------------

  alu.io.a      := regFile.io.readDataA.asUInt
  alu.io.b      := regFile.io.readDataB.asUInt
  alu.io.opcode := decoder.io.opcode

  io.result := alu.io.result.asBits

  // --------------------------------------------------
  // Register Write
  // --------------------------------------------------

  regFile.io.writeEnable  := io.loadEnable
  regFile.io.writeAddress := io.loadAddress
  regFile.io.writeData    := io.loadData

  when(decoder.io.valid) {
    regFile.io.writeEnable  := True
    regFile.io.writeAddress := decoder.io.rd
    regFile.io.writeData    := alu.io.result.asBits
  }
}
```

### 여기서 중요한 점

이제 CPU 외부에서 instruction을 직접 넣지 않습니다.

```text
❌ 이전

외부 → instruction → CPU
```

가 아니라,

```text
✅ 현재

PC → ROM → instruction → CPU
```

입니다.

---

# Simulation

우선 R0와 R1을 초기화합니다.

그 다음 ROM에 들어 있는 명령어를 CPU가 실제로 가져오는지 확인합니다.

```scala
import spinal.core.sim._

object MiniCpuSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave.compile(
      new MiniCpu
    ).doSim { dut =>

      dut.clockDomain.forkStimulus(10)

      // ----------------------------------------------
      // R0 = 0x10
      // ----------------------------------------------

      dut.io.loadEnable  #= true
      dut.io.loadAddress #= 0
      dut.io.loadData    #= 0x10

      dut.clockDomain.waitSampling()

      // ----------------------------------------------
      // R1 = 0x20
      // ----------------------------------------------

      dut.io.loadAddress #= 1
      dut.io.loadData    #= 0x20

      dut.clockDomain.waitSampling()

      dut.io.loadEnable #= false

      // ----------------------------------------------
      // PC = 2
      //
      // FetchUnit은 매 clock마다 PC를 증가시키므로
      // 현재 PC 위치는 초기화 clock 수만큼 진행됨.
      //
      // 여기서는 현재 fetch 상태를 직접 확인한다.
      // ----------------------------------------------

      sleep(1)

      println(
        f"PC=${dut.io.pc.toInt} " +
        f"instruction=0x${dut.io.instruction.toBigInt}%04X " +
        f"result=0x${dut.io.result.toBigInt}%02X"
      )

      println("Mini CPU integration test completed.")
    }
  }
}
```

그런데 여기서 **중요한 문제가 하나 있습니다.**

현재 `MiniCpu`에서:

```scala
fetch.io.enable := True
```

이므로 **CPU가 매 clock마다 계속 실행됩니다.**

따라서 단순히 `R0/R1`을 load하는 동안에도 PC가 진행됩니다.

그리고 더 중요한 문제가 있습니다.

현재 ROM의 명령어는:

```text
0x0810 = ADD R2, R0, R1
```

이므로 R0/R1 초기화 직후 바로 실행될 수 있지만, 그 다음 clock에는 계속 다른 명령어가 실행됩니다.

즉 지금 구조는 **“프로그램 실행”과 “외부 register 초기화”가 동시에 존재하는 과도기 구조**입니다.

이 상태에서 무리하게 전체 CPU simulation을 작성하면 이전처럼 **clock scheduling 때문에 결과를 잘못 해석할 가능성**이 있습니다.

따라서 이번 단계에서는 먼저 **MiniCpu의 wiring이 맞는지 확인하고**, 다음 단계에서 `load`와 CPU 실행을 명확히 분리하는 **Reset/Run 구조**를 넣는 것이 올바릅니다.

### 그래서 이번 12-9의 목표

현재는 코드 연결 자체를 이해하는 것이 목적이고, **아직 “test passed”를 선언하지 않습니다.**

다음 **12-10**에서:

```text
RESET
  ↓
Register 초기화
  ↓
RUN
  ↓
PC → ROM → Decode → Execute → Write Back
```

구조를 정리하고 **실제로 ADD → SUB → AND → OR 명령어를 순차 실행하는 simulation**을 만들겠습니다.

이렇게 해야 Project 12를 깔끔하게 마무리할 수 있습니다.

---

