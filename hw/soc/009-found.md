여전히 동일한 오류다. 

```
[Progress] at 0.000 : Elaborate components
ERROR !
A null pointer access has been detected in the JVM.
This could happen when in your SpinalHDL description, you access an signal which is only defined further.
For instance :
  val result = Bool()
  result := a ^ b  //a and b can't be accessed there because they are only defined one line below (Software rule of execution order)
  val a,b = Bool()
   
[error] java.lang.NullPointerException: Cannot invoke "vexriscv.plugin.IBusSimpleBus.cmd()" because 
the return value of "vexriscv.plugin.IBusSimplePlugin.iBus()" is null
```

---

원인을 찾았습니다. 제가 앞서 제시한 `VexRiscvTop.scala`가 Murax의 중요한 구조를 빠뜨렸습니다. CPU를 최상위 `Component`에 직접 생성한 뒤 곧바로 `cpu.plugins`의 버스에 접근하도록 했는데, 이 코드에서는 `IBusSimplePlugin.iBus`가 아직 생성되지 않은 상태였습니다.

실제 Murax 소스를 다시 확인했습니다. Murax는 CPU 생성과 Plugin 연결을 모두 다음 영역 안에 배치합니다.

```
val system = new ClockingArea(systemClockDomain) {
  val cpu = new VexRiscv(...)

  for (plugin <- cpu.plugins) {
    // Plugin 연결
  }
}
```

실제 소스의 208\~243행이 이 구조입니다.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



이번에는 이 구조를 그대로 따르겠습니다. `rework`, `addPrePopTask`, 사용자 정의 메모리 Plugin은 사용하지 않습니다.

## 1. 수정된 `VexRiscvTop.scala`

```

package myvexsoc

import spinal.core._
import vexriscv._
import vexriscv.plugin._

class VexRiscvTop extends Component {

  // CPU configuration
  val cpuConfig = CpuConfig.smallest()

  // Follow the same structure as Murax:
  // instantiate the CPU and connect its plugins inside a ClockingArea.
  val system = new ClockingArea(ClockDomain.current) {

    val cpu = new VexRiscv(cpuConfig)

    // SoC interrupt sources
    val externalInterrupt = False
    val timerInterrupt = False

    // Connect CPU plugins
    for (plugin <- cpu.plugins) {
      plugin match {

        // Instruction bus
        case p: IBusSimplePlugin =>
          p.iBus.cmd.ready := False

          p.iBus.rsp.valid := False
          p.iBus.rsp.error := False
          p.iBus.rsp.inst := B(0, 32 bits)

        // Data bus
        case p: DBusSimplePlugin =>
          p.dBus.cmd.ready := False

          p.dBus.rsp.ready := False
          p.dBus.rsp.error := False
          p.dBus.rsp.data := B(0, 32 bits)

        // CSR interrupt inputs
        case p: CsrPlugin =>
          p.externalInterrupt := externalInterrupt
          p.timerInterrupt := timerInterrupt

        case _ =>
      }
    }
  }
}

```

## 2. `VexRiscvTopSim.scala`

시뮬레이션 코드는 이전 것을 그대로 사용해도 됩니다.

```

package myvexsoc

import spinal.core._
import spinal.core.sim._

object VexRiscvTopSim extends App {

  SimConfig
    .withWave
    .compile(new VexRiscvTop)
    .doSim { dut =>

      println("========================================")
      println(" VexRiscvTop Simulation Start")
      println("========================================")

      dut.clockDomain.forkStimulus(period = 10)

      dut.clockDomain.waitSampling(5)
      println("[PASS] Initial reset and clock startup")

      dut.clockDomain.waitSampling(20)
      println("[PASS] CPU clocked for 20 additional cycles")

      println("========================================")
      println(" VexRiscvTop Simulation Finished")
      println("========================================")
    }
}

```

## 3. 실행

```
sbt "runMain myvexsoc.VexRiscvTopSim"
```

이번 수정에서 바뀐 핵심은 CPU와 Plugin 연결을 `ClockingArea` 내부로 옮긴 것입니다. Murax의 실제 구조와 일치시켰습니다.

아직 검증 전

제가 여기서 실제 로컬 프로젝트를 실행한 것은 아니므로 성공을 보장하지는 않겠습니다. 실행 결과를 보내 주세요. 특히 이번에는 `iBus() is null` 오류가 사라지는지 먼저 확인하겠습니다.
