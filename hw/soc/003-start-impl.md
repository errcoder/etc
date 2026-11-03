좋습니다. 이제 **VexRiscv + Instruction ROM**부터 실제로 붙이겠습니다.

이번 단계에서는 너무 많은 것을 한꺼번에 넣지 않고,

> **VexRiscv → IBus → ROM → IBus 응답**

까지만 만들겠습니다.

VexRiscv 쪽은 지금 사용자가 직접 생성해서 확인한 **`GenSmallest`와 동일한 설정**을 기준으로 합니다. 현재 생성된 RTL이 SpinalHDL **1.13.0**으로 만들어졌다는 것도 확인되어 있으므로, 이 SoC 프로젝트에서는 일단 그 버전에 맞추겠습니다. VexRiscv 공식 예제도 `VexRiscv`의 plugin 구성과 `IBusSimplePlugin`/`DBusSimplePlugin` 구조를 동일하게 사용합니다. [GitHub](https://raw.githubusercontent.com/SpinalHDL/VexRiscv/master/src/main/scala/vexriscv/VexRiscv.scala)

## 1. 이번 단계의 구조

```text
                    +----------------------+
                    |      VexRiscv        |
                    |                      |
                    |  IBusSimplePlugin    |
                    +----------+-----------+
                               |
                         IBus command
                         PC = 0x8000_....
                               |
                               v
                    +----------------------+
                    |   Instruction ROM    |
                    |                      |
                    |  0x80000000 : ADDI   |
                    |  0x80000004 : ADDI   |
                    |  0x80000008 : ADD    |
                    |  0x8000000C : JAL    |
                    +----------------------+

DBus는 이번 단계에서는
CPU가 접근하지 않도록 기본 응답만 연결
```

특히 VexRiscv의 reset vector가 현재 `0x80000000`이므로 ROM도 그 주소에서 시작하도록 합니다.

---

# 2. 프로젝트 구조

기존 VexRiscv 저장소와 별도로 SoC 프로젝트를 하나 만드세요.

```text
MyVexSoc/
├── build.sbt
├── ext/
│   └── VexRiscv/
├── src/
│   └── main/
│       └── scala/
│           └── myvexsoc/
│               ├── CpuConfig.scala
│               ├── InstructionRom.scala
│               ├── VexRiscvTop.scala
│               └── VexRiscvTopSim.scala
└── generated/
```

`ext/VexRiscv`에는 지금 사용자가 clone한 VexRiscv를 그대로 넣습니다.

VexRiscv를 별도 프로젝트로 복사하는 것보다 source dependency 형태로 사용하는 방식은 공식 SpinalHDL의 `SpinalTemplateSbtDependencies` 예제에서도 사용하는 방법입니다. [GitHub](https://github.com/SpinalHDL/SpinalTemplateSbtDependencies?utm_source=chatgpt.com)

---

# 3. `build.sbt`

먼저 이것부터 만듭니다.

```scala
ThisBuild / scalaVersion := "2.12.18"

val spinalVersion = "1.13.0"

lazy val root = (project in file("."))
  .settings(
    name := "MyVexSoc",

    libraryDependencies ++= Seq(
      "com.github.spinalhdl" %% "spinalhdl-core" % spinalVersion,
      "com.github.spinalhdl" %% "spinalhdl-lib"  % spinalVersion
    ),

    Compile / unmanagedSourceDirectories +=
      baseDirectory.value / "ext" / "VexRiscv" / "src" / "main" / "scala"
  )
```

핵심은 이 부분입니다.

```scala
Compile / unmanagedSourceDirectories +=
  baseDirectory.value / "ext" / "VexRiscv" / "src" / "main" / "scala"
```

즉 우리가 작성하는 SoC 프로젝트에서

```scala
import vexriscv._
import vexriscv.plugin._
```

를 직접 사용할 수 있게 됩니다.

---

# 4. `CpuConfig.scala`

여기는 **사용자가 이미 성공적으로 실행한 `GenSmallest` 설정을 그대로 가져옵니다.**

```scala
package myvexsoc

import spinal.core._
import vexriscv._
import vexriscv.plugin._
import vexriscv.{plugin => vexPlugin}

object CpuConfig {

  def smallest(): VexRiscvConfig = {

    VexRiscvConfig(
      plugins = List(

        new IBusSimplePlugin(
          resetVector = 0x80000000l,
          cmdForkOnSecondStage = false,
          cmdForkPersistence = false,
          prediction = NONE,
          catchAccessFault = false,
          compressedGen = false
        ),

        new DBusSimplePlugin(
          catchAddressMisaligned = false,
          catchAccessFault = false
        ),

        new CsrPlugin(
          CsrPluginConfig.smallest
        ),

        new DecoderSimplePlugin(
          catchIllegalInstruction = false
        ),

        new RegFilePlugin(
          regFileReadyKind = vexPlugin.SYNC,
          zeroBoot = false
        ),

        new IntAluPlugin,

        new SrcPlugin(
          separatedAddSub = false,
          executeInsertion = false
        ),

        new LightShifterPlugin,

        new HazardSimplePlugin(
          bypassExecute           = false,
          bypassMemory            = false,
          bypassWriteBack         = false,
          bypassWriteBackBuffer   = false,
          pessimisticUseSrc       = false,
          pessimisticWriteRegFile = false,
          pessimisticAddressMatch = false
        ),

        new BranchPlugin(
          earlyBranch = false,
          catchAddressMisaligned = false
        ),

        new YamlPlugin("cpu0.yaml")
      )
    )
  }
}
```

여기서는 일부러 CPU 설정을 임의로 바꾸지 않았습니다.

즉 현재 사용자가 생성했던:

```text
GenSmallest
```

와 같은 CPU를 우리의 SoC 안에 넣는 것입니다.

---

# 5. Instruction ROM

이번 ROM은 우선 **simulation이 확실하게 동작하는 것**을 목표로 하겠습니다.

프로그램은 다음과 같습니다.

```text
0x80000000 : ADDI x1, x0, 1
0x80000004 : ADDI x2, x0, 2
0x80000008 : ADD  x3, x1, x2
0x8000000C : JAL  x0, 0
```

마지막 명령은 자기 자신으로 점프합니다.

따라서 CPU가 제대로 실행되면 대략:

```text
80000000
80000004
80000008
8000000C
8000000C
8000000C
...
```

형태의 fetch가 관찰되어야 합니다.

## `InstructionRom.scala`

```scala
package myvexsoc

import spinal.core._

class InstructionRom(
    baseAddress: BigInt = 0x80000000L,
    wordCount: Int = 16
) extends Component {

  val io = new Bundle {
    val cmdValid = in Bool ()
    val cmdReady = out Bool ()
    val cmdPc    = in UInt (32 bits)

    val rspValid = out Bool ()
    val rspError = out Bool ()
    val rspInst  = out Bits (32 bits)
  }

  val rom = Vec(Bits(32 bits), wordCount)

  // --------------------------------------------------------------------------
  // Test program
  //
  //   0x80000000 : ADDI x1, x0, 1
  //   0x80000004 : ADDI x2, x0, 2
  //   0x80000008 : ADD  x3, x1, x2
  //   0x8000000C : JAL  x0, 0
  //
  // --------------------------------------------------------------------------

  rom(0) := B(0x00100093L, 32 bits) // addi x1, x0, 1
  rom(1) := B(0x00200113L, 32 bits) // addi x2, x0, 2
  rom(2) := B(0x002081B3L, 32 bits) // add  x3, x1, x2
  rom(3) := B(0x0000006FL, 32 bits) // jal  x0, 0

  for (i <- 4 until wordCount) {
    rom(i) := B(0x00000013L, 32 bits) // nop
  }

  io.cmdReady := True

  val cmdFire = io.cmdValid && io.cmdReady

  val addressOffset =
    io.cmdPc - U(baseAddress, 32 bits)

  val wordIndex = UInt(log2Up(wordCount) bits)
  wordIndex := (addressOffset >> 2).resized

  val instruction = Bits(32 bits)
  instruction := B(0x00000013L, 32 bits)

  when(addressOffset < U(wordCount * 4, 32 bits)) {
    instruction := rom(wordIndex)
  }

  // One-cycle response.
  //
  // The command is sampled first.
  // The corresponding instruction is returned on the next cycle.
  io.rspValid := RegNext(cmdFire) init(False)
  io.rspInst  := RegNext(instruction) init(B(0x00000013L, 32 bits))
  io.rspError := False
}
```

### 여기서 중요한 점

ROM 응답을 **한 사이클 늦춰서** 줍니다.

```text
cycle N
    cmdValid = 1
    cmdPc    = 0x80000000

cycle N+1
    rspValid = 1
    rspInst  = ADDI x1,x0,1
```

이렇게 하는 이유는 CPU와 메모리 사이에 명확한 synchronous boundary를 만들기 위해서입니다.

그리고 사용자가 이전 UART/FIFO 학습에서 겪었던 것처럼 simulation에서 combinational signal을 보고 잘못된 시점의 값을 판단하는 문제도 줄어듭니다.

---

# 6. VexRiscvTop

이제 CPU와 ROM을 연결합니다.

여기서 중요한 VexRiscv 사용법이 하나 있습니다.

`VexRiscv`의 plugin이 만드는 `iBus`, `dBus`를 `rework`에서 외부 bus로 빼는 방식입니다. VexRiscv 공식 소스에서도 `plugin.iBus.setAsDirectionLess()` 후 외부 bus로 연결하는 패턴을 사용합니다. [GitHub](https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Linux.scala?utm_source=chatgpt.com)

```scala
package myvexsoc

import spinal.core._
import vexriscv._
import vexriscv.plugin._

class VexRiscvTop extends Component {

  val io = new Bundle {
    // Simulation / bring-up debug signals
    val debugIbusValid = out Bool ()
    val debugIbusReady = out Bool ()
    val debugIbusPc    = out UInt (32 bits)

    val debugRspValid  = out Bool ()
    val debugRspInst   = out Bits (32 bits)

    val debugDmemValid = out Bool ()
    val debugDmemWr    = out Bool ()
    val debugDmemAddr  = out UInt (32 bits)
    val debugDmemData  = out Bits (32 bits)
  }

  val cpu = new VexRiscv(
    CpuConfig.smallest()
  )

  var iBus: IBusSimpleBus = null
  var dBus: DBusSimpleBus = null

  cpu.rework {

    cpu.config.plugins.foreach {

      case plugin: IBusSimplePlugin =>
        plugin.iBus.setAsDirectionLess()
        iBus = master(plugin.iBus)
          .setName("iBus")

      case plugin: DBusSimplePlugin =>
        plugin.dBus.setAsDirectionLess()
        dBus = master(plugin.dBus)
          .setName("dBus")

      case _ =>
    }
  }

  // --------------------------------------------------------------------------
  // Instruction ROM
  // --------------------------------------------------------------------------

  val rom = new InstructionRom(
    baseAddress = 0x80000000L,
    wordCount = 16
  )

  rom.io.cmdValid := iBus.cmd.valid
  rom.io.cmdPc    := iBus.cmd.pc

  iBus.cmd.ready := rom.io.cmdReady

  iBus.rsp.valid := rom.io.rspValid
  iBus.rsp.error := rom.io.rspError
  iBus.rsp.inst  := rom.io.rspInst

  // --------------------------------------------------------------------------
  // Data bus
  //
  // Project 1에서는 실제 RAM이 없다.
  // 따라서 CPU가 data access를 하지 않는 한 이 부분은 사용되지 않는다.
  // --------------------------------------------------------------------------

  dBus.cmd.ready := True

  dBus.rsp.ready := True
  dBus.rsp.error := False
  dBus.rsp.data  := B(0, 32 bits)

  // --------------------------------------------------------------------------
  // Debug
  // --------------------------------------------------------------------------

  io.debugIbusValid := iBus.cmd.valid
  io.debugIbusReady := iBus.cmd.ready
  io.debugIbusPc    := iBus.cmd.pc

  io.debugRspValid  := iBus.rsp.valid
  io.debugRspInst   := iBus.rsp.inst

  io.debugDmemValid := dBus.cmd.valid
  io.debugDmemWr    := dBus.cmd.wr
  io.debugDmemAddr  := dBus.cmd.address
  io.debugDmemData  := dBus.cmd.data
}
```

여기서 한 가지 포인트가 있습니다.

```scala
cpu.rework {
```

안에서 VexRiscv의 plugin이 생성한 bus를 밖으로 끌어냅니다.

즉 우리가 이전에 만들었던 직접 만든 CPU처럼

```text
CPU → 우리가 정의한 Bus → RAM
```

구조를 만드는 것이 아니라,

```text
VexRiscv
   ↓
VexRiscv plugin이 생성한 IBus
   ↓
우리 SoC의 ROM
```

으로 연결하는 것입니다.

---

# 7. Simulation

이제 가장 중요한 부분입니다.

이번 simulation은 **무한 `while`을 사용하지 않겠습니다.**

사용자가 예전에 UART에서 겪었던 것처럼 simulation scheduling 때문에 잘못된 관찰이 발생할 가능성을 줄이기 위해 **고정 cycle 수**로 확인합니다.

```scala
package myvexsoc

import spinal.core.sim._

object VexRiscvTopSim extends App {

  SimConfig
    .withWave
    .allOptimisation
    .compile(new VexRiscvTop)
    .doSim { dut =>

      dut.clockDomain.forkStimulus(period = 10)

      // ------------------------------------------------------------
      // Reset
      // ------------------------------------------------------------

      dut.clockDomain.resetSim #= true

      dut.clockDomain.waitSampling(5)

      dut.clockDomain.resetSim #= false

      println()
      println("==============================================")
      println(" VexRiscv + Instruction ROM simulation")
      println("==============================================")
      println()

      // ------------------------------------------------------------
      // Observe a fixed number of cycles.
      //
      // No unbounded while-loop.
      // ------------------------------------------------------------

      for (cycle <- 0 until 40) {

        dut.clockDomain.waitSampling()

        val ibusValid = dut.debugIbusValid.toBoolean
        val ibusReady = dut.debugIbusReady.toBoolean
        val pc        = dut.debugIbusPc.toBigInt

        val rspValid  = dut.debugRspValid.toBoolean
        val rspInst   = dut.debugRspInst.toBigInt

        val dmemValid = dut.debugDmemValid.toBoolean

        if (ibusValid && ibusReady) {

          println(
            f"Cycle ${cycle + 1}%2d : " +
            f"IBUS PC=0x$pc%08X"
          )
        }

        if (rspValid) {

          println(
            f"             RSP INST=0x$rspInst%08X"
          )
        }

        if (dmemValid) {

          val wr   = dut.debugDmemWr.toBoolean
          val addr = dut.debugDmemAddr.toBigInt
          val data = dut.debugDmemData.toBigInt

          println(
            f"             DBUS ${if (wr) "WRITE" else "READ "} " +
            f"ADDR=0x$addr%08X DATA=0x$data%08X"
          )
        }
      }

      println()
      println("Simulation finished.")
      println()
    }
}
```

---

# 8. 실행

프로젝트 루트에서:

```bash
sbt "runMain myvexsoc.VexRiscvTopSim"
```

그리고 Verilog 생성은 나중에:

```bash
sbt "runMain myvexsoc.VexRiscvTop"
```

형태로 별도 generator를 만드는 것이 좋습니다.

---

## 9. 그런데 여기서 바로 한 가지 확인할 것

위 코드는 **현재 clone한 VexRiscv source와 실제 compile을 맞춰보는 단계**입니다.

특히 다음 두 부분은 VexRiscv 버전에 따라 API 차이가 생길 수 있습니다.

```scala
iBus = master(plugin.iBus)
```

와

```scala
dBus = master(plugin.dBus)
```

따라서 **처음부터 “이 코드는 무조건 컴파일된다”고 가정하지 않고**, 사용자가 현재 clone한 VexRiscv commit에 맞춰 실제 SBT compile을 확인하는 것이 맞습니다.

VexRiscv 자체도 공식 README에서 특정 시점에는 SpinalHDL의 unreleased master-head가 필요할 수 있다고 명시하고 있습니다. [GitHub](https://github.com/SpinalHDL/VexRiscv?utm_source=chatgpt.com)

---

# 10. 이번 단계에서 성공 판정

우리는 아직 RAM도, APB도, UART도 만들지 않습니다.

이번 Project 1-2의 성공 조건은 딱 이것입니다.

```text
Reset
  ↓
VexRiscv
  ↓
IBus request
  ↓
0x80000000
  ↓
ADDI x1,x0,1
  ↓
0x80000004
  ↓
ADDI x2,x0,2
  ↓
0x80000008
  ↓
ADD x3,x1,x2
  ↓
0x8000000C
  ↓
JAL x0,0
  ↓
0x8000000C
  ↓
0x8000000C
  ↓
...
```

특히 마지막의

```text
0x8000000C → 0x8000000C → 0x8000000C
```

가 중요합니다.

단순히 CPU가 instruction을 fetch하는 것뿐 아니라 **실제로 `JAL`까지 실행하고 있다는 증거**가 되기 때문입니다.

그리고 다음 단계에서는 이 ROM을 그대로 유지하면서 **RAM을 추가하고 `SW/LW`를 실제로 수행해서 `x3 = 3`이 RAM에 저장되는 것까지 검증**하겠습니다. 
그때부터 비로소 “CPU가 실제 RISC-V 프로그램을 실행한다”는 것을 확실하게 검증할 수 있습니다.

---

---

좋습니다. 이 오류는 **코드 문제가 아니라 VexRiscv의 `YamlPlugin.scala`가 필요로 하는 SnakeYAML dependency가 우리 `build.sbt`에 빠진 것**입니다.

현재 로그:

```text
object yaml is not a member of package org
import org.yaml.snakeyaml.{DumperOptions, Yaml}
```

즉 다음 dependency가 필요합니다.

```text
org.yaml:snakeyaml
```

그리고 우리가 `CpuConfig.scala`에서 넣었던:

```scala
new YamlPlugin("cpu0.yaml")
```

때문에 `YamlPlugin.scala`까지 컴파일 대상이 되면서 이 문제가 발생했습니다.

## 가장 깔끔한 수정

사실 **이번 단계에서는 `YamlPlugin`이 필요 없습니다.**

`YamlPlugin`은 CPU configuration YAML을 생성하기 위한 plugin이지, CPU 실행에 필요한 plugin이 아닙니다.

따라서 `CpuConfig.scala`에서 이것만 제거하는 것을 권장합니다.

```scala
new YamlPlugin("cpu0.yaml")
```

### 수정된 `CpuConfig.scala`

```scala
package myvexsoc

import spinal.core._
import vexriscv._
import vexriscv.plugin._
import vexriscv.{plugin => vexPlugin}

object CpuConfig {

  def smallest(): VexRiscvConfig = {

    VexRiscvConfig(
      plugins = List(

        new IBusSimplePlugin(
          resetVector = 0x80000000l,
          cmdForkOnSecondStage = false,
          cmdForkPersistence = false,
          prediction = NONE,
          catchAccessFault = false,
          compressedGen = false
        ),

        new DBusSimplePlugin(
          catchAddressMisaligned = false,
          catchAccessFault = false
        ),

        new CsrPlugin(
          CsrPluginConfig.smallest
        ),

        new DecoderSimplePlugin(
          catchIllegalInstruction = false
        ),

        new RegFilePlugin(
          regFileReadyKind = vexPlugin.SYNC,
          zeroBoot = false
        ),

        new IntAluPlugin,

        new SrcPlugin(
          separatedAddSub = false,
          executeInsertion = false
        ),

        new LightShifterPlugin,

        new HazardSimplePlugin(
          bypassExecute           = false,
          bypassMemory            = false,
          bypassWriteBack         = false,
          pessimisticUseSrc       = false,
          pessimisticWriteRegFile = false,
          pessimisticAddressMatch = false
        ),

        new BranchPlugin(
          earlyBranch = false,
          catchAddressMisaligned = false
        )
      )
    )
  }
}
```

그리고 **`YamlPlugin`을 제거했으므로 `YamlPlugin.scala` 자체를 compile하지 않아도 되는가?**

여기서 중요한 부분이 하나 있습니다.

현재 build가:

```scala
Compile / unmanagedSourceDirectories +=
  baseDirectory.value / "ext" / "VexRiscv" / "src" / "main" / "scala"
```

로 되어 있어서 VexRiscv의 **모든 Scala source**를 컴파일합니다.

따라서 `YamlPlugin.scala`는 여전히 컴파일되고 SnakeYAML 오류가 다시 날 가능성이 있습니다.

그러므로 이번에는 dependency를 추가하는 것이 더 안전합니다.

## `build.sbt`에 SnakeYAML 추가

```scala
libraryDependencies ++= Seq(
  "com.github.spinalhdl" %% "spinalhdl-core" % spinalVersion,
  "com.github.spinalhdl" %% "spinalhdl-lib"  % spinalVersion,
  "org.yaml" % "snakeyaml" % "2.2"
)
```

즉:

```scala
ThisBuild / scalaVersion := "2.13.12"

val spinalVersion = "1.13.0"

lazy val root = (project in file("."))
  .settings(
    name := "MyVexSoc",

    libraryDependencies ++= Seq(
      "com.github.spinalhdl" %% "spinalhdl-core" % spinalVersion,
      "com.github.spinalhdl" %% "spinalhdl-lib"  % spinalVersion,
      "org.yaml" % "snakeyaml" % "2.2"
    ),

    Compile / unmanagedSourceDirectories +=
      baseDirectory.value / "ext" / "VexRiscv" / "src" / "main" / "scala"
  )
```

### 그런데 `scalaVersion`도 확인해야 합니다

로그에:

```text
target/scala-2.13/classes
```

가 보입니다.

따라서 현재 프로젝트는 **Scala 2.13**을 사용하고 있습니다.

제가 앞에서 `2.12.18`을 제시한 것은 현재 사용자의 실제 환경과 맞지 않았습니다. 이 부분은 수정하는 것이 맞습니다.

---

## 지금은 이렇게 해주세요

1. `build.sbt`에 SnakeYAML 추가
2. `CpuConfig.scala`에서 `YamlPlugin` 제거
3. SBT에서:

```text
clean
compile
```

그리고:

```text
runMain myvexsoc.VexRiscvTopSim
```

을 실행하세요.

### 만약 다음 오류가 나오면

그 오류를 **그대로 붙여 주세요.**

이번에는 한 번에 여러 부분을 추측해서 고치지 않고,

```text
VexRiscv source
        ↓
compile
        ↓
VexRiscvTop elaboration
        ↓
simulation
```

순서대로 하나씩 실제 환경에 맞춰 맞추겠습니다.

특히 지금 단계에서는 **SnakeYAML 문제를 해결한 뒤 곧바로 `VexRiscvTop.scala`의 실제 bus API가 현재 clone과 맞는지 검증**하는 것이 다음 관문입니다.

---

---

맞습니다. 이건 제가 앞에서 `master(plugin.iBus)`라고 작성한 부분이 잘못됐습니다.

`master(...)`는 지금 사용 중인 SpinalHDL/VexRiscv 조합에서 사용할 수 있는 API가 아닙니다. **현재 VexRiscv의 `iBus`를 그대로 CPU 외부로 연결하는 방식으로 수정해야 합니다.**

그리고 여기서는 더 이상 API를 추측하지 않고, **현재 clone한 VexRiscv의 실제 `IBusSimplePlugin`/`IBusSimpleBus` 정의에 맞춰 수정하는 게 맞습니다.**

우선 `VexRiscvTop.scala`에서 이 부분:

```scala
iBus = master(plugin.iBus)
  .setName("iBus")
```

을 제거하세요.

다음으로 현재 VexRiscv source에서 `IBusSimplePlugin`이 어떻게 bus를 노출하는지 확인해야 합니다. 사용자가 현재 clone한 VexRiscv가 기준이므로, **`ext/VexRiscv/src/main/scala/vexriscv/plugin/IBusSimplePlugin.scala`의 내용**을 확인하면 정확히 맞출 수 있습니다.

터미널에서 다음을 실행해 주세요.

```bash
sed -n '1,260p' ext/VexRiscv/src/main/scala/vexriscv/plugin/IBusSimplePlugin.scala
```

그리고 결과를 그대로 보내주세요.

가능하면 이것도 같이 주세요.

```bash
grep -R "setAsDirectionLess" -n ext/VexRiscv/src/main/scala
```

이 두 결과를 보면 현재 버전의 정확한 연결 방식에 맞춰 `VexRiscvTop.scala`를 다시 작성할 수 있습니다.

**중요:** 지금은 `VexRiscvTop.scala`를 이것저것 임의로 수정하지 마세요. 앞 단계에서 제가 VexRiscv API를 잘못 가정한 것이므로, 이번에는 실제 source를 기준으로 맞추겠습니다.

---
