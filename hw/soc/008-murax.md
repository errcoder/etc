vexriscv cpu 를 사용하는 soc 예제를 찾아보고 정확히 어떻게 플러그인 을 사용하는 지 알아봐

---

조사해 보니, VexRiscv를 SoC에 통합할 때 따라야 할 가장 좋은 기준 예제는 같은 VexRiscv 저장소에 있는 `Murax.scala`입니다. 그리고 이 예제를 보면 앞서 제가 제안했던 `SoCMemoryPlugin` 방식은 수정해야 합니다.

핵심은 다음과 같습니다.

> CPU 기능을 확장할 때는 VexRiscv Plugin을 사용하고, CPU의 메모리 버스와 SoC의 RAM·ROM·주변장치를 연결할 때는 SoC 최상위 컴포넌트에서 CPU의 Plugin 객체를 찾아 연결합니다.

즉, 메모리와 주변장치를 연결하기 위해 별도의 CPU Plugin을 만들 필요가 없습니다. 앞서 제가 `SoCMemoryPlugin` 안에서 버스를 연결하도록 안내한 것은 Murax의 실제 설계 방식과 다릅니다.

## 1. 가장 중요한 예제: Murax

원본 소스: [VexRiscv / Murax.scala](https://github.com/SpinalHDL/VexRiscv/blob/baf7dc82f855eddaf5b0a20a6802c526f127e38/src/main/scala/vexriscv/demo/Murax.scala?utm_source=chatgpt.com)

Murax는 VexRiscv CPU를 중심으로 다음을 구성한 실제 SoC입니다.

- 명령어 버스와 데이터 버스
- 8KB 온칩 RAM
- 메모리 버스 Arbiter 및 Decoder
- APB3 Bridge
- GPIO, UART, Timer
- CSR 인터럽트 연결

### 1-1. CPU Plugin 목록 구성

Murax의 `MuraxConfig.default`에는 다음과 같이 Plugin들이 들어갑니다.

```
cpuPlugins = ArrayBuffer(
  new IBusSimplePlugin(
    resetVector = 0x80000000l,
    cmdForkOnSecondStage = true,
    cmdForkPersistence = false,
    prediction = NONE,
    catchAccessFault = false,
    compressedGen = false
  ),
  new DBusSimplePlugin(
    catchAddressMisaligned = false,
    catchAccessFault = false,
    earlyInjection = false
  ),
  new CsrPlugin(CsrPluginConfig.smallest(
    mtvecInit = 0x80000020l
  )),
  new DecoderSimplePlugin(
    catchIllegalInstruction = false
  ),
  new RegFilePlugin(
    regFileReadyKind = plugin.SYNC,
    zeroBoot = false
  ),
  new IntAluPlugin,
  new SrcPlugin(
    separatedAddSub = false,
    executeInsertion = false
  ),
  new LightShifterPlugin,
  new HazardSimplePlugin(
    bypassExecute = false,
    bypassMemory = false,
    bypassWriteBack = false,
    bypassWriteBackBuffer = false
  ),
  new BranchPlugin(
    earlyBranch = false,
    catchAddressMisaligned = false
  ),
  new YamlPlugin("cpu0.yaml")
)
```

실제 소스의 구성은 이와 같으며, 각 Plugin의 설정은 CPU의 기능과 파이프라인 동작을 결정합니다.

### 1-2. CPU 생성 후 Plugin 객체를 순회하며 연결

가장 중요한 부분입니다. Murax는 CPU를 먼저 생성합니다.

```
val cpu = new VexRiscv(
  config = VexRiscvConfig(
    plugins = cpuPlugins +=
      new DebugPlugin(debugClockDomain, hardwareBreakpointCount)
  )
)
```

그 직후 `cpu.plugins`를 순회합니다.

```
for(plugin <- cpu.plugins) plugin match {

  case plugin: IBusSimplePlugin =>
    mainBusArbiter.io.iBus.cmd <> plugin.iBus.cmd
    mainBusArbiter.io.iBus.rsp <> plugin.iBus.rsp

  case plugin: DBusSimplePlugin =>
    mainBusArbiter.io.dBus <> plugin.dBus

  case plugin: CsrPlugin =>
    plugin.externalInterrupt := externalInterrupt
    plugin.timerInterrupt := timerInterrupt

  case _ =>
}
```

이것이 실제 SoC 통합의 핵심 패턴입니다.

- `IBusSimplePlugin`: CPU 명령어 버스를 SoC의 메모리 시스템에 연결합니다.
- `DBusSimplePlugin`: CPU 데이터 버스를 SoC의 메모리 시스템에 연결합니다.
- `CsrPlugin`: SoC의 인터럽트 신호를 CPU에 전달합니다.
- `DebugPlugin`: 디버그 인터페이스를 SoC 외부 또는 디버그 시스템에 연결합니다.

여기서 중요한 점은 Plugin을 생성하는 것과 Plugin의 하드웨어 인터페이스를 SoC에 연결하는 것은 서로 다른 단계라는 것입니다. Murax는 CPU 생성 이후 최상위 SoC 영역에서 이 연결을 수행합니다.

### 1-3. 메모리와 주변장치 연결

Murax는 CPU의 IBus와 DBus를 곧바로 RAM에 각각 연결하지 않습니다.

```
             VexRiscv CPU
              │       │
           IBus      DBus
              │       │
              └───┬───┘
                  │
             Main Bus Arbiter
                  │
             Main Bus Decoder
              ┌───┴────┐
              │        │
             RAM    APB Bridge
                       │
                   APB Decoder
                  ┌────┼────┐
                  │    │    │
                 GPIO UART Timer
```

실제 Murax 소스에서도 다음과 같이 구성합니다.

- `MuraxMasterArbiter`: IBus와 DBus를 하나의 Main Bus로 통합
- `MuraxPipelinedMemoryBusRam`: 온칩 RAM
- `PipelinedMemoryBusToApbBridge`: APB 변환
- `Apb3Decoder`: APB 주변장치 주소 디코딩
- `MuraxPipelinedMemoryBusDecoder`: RAM과 APB Bridge의 주소 영역 분리

주소 매핑은 다음과 같습니다.

| 장치     | 주소            |
| ------ | ------------- |
| RAM    | `0x80000000`  |
| APB 영역 | `0xF0000000`  |
| GPIO   | APB `0x00000` |
| UART   | APB `0x10000` |
| Timer  | APB `0x20000` |

이 부분이 우리가 만들 SoC의 버스 구조를 설계할 때 참고할 핵심입니다.

## 2. 두 번째 예제: Briey

원본 소스: [VexRiscv / Briey.scala](https://github.com/SpinalHDL/VexRiscv/blob/baf7dc82f855eddaf5b0a20a6802c526f127e38/src/main/scala/vexriscv/demo/Briey.scala?utm_source=chatgpt.com)

Briey는 Murax보다 복잡한 SoC입니다.

| 구분      | Murax              | Briey                  |
| ------- | ------------------ | ---------------------- |
| 명령어 버스  | IBusSimplePlugin   | IBusCachedPlugin       |
| 데이터 버스  | DBusSimplePlugin   | DBusCachedPlugin       |
| 메모리 시스템 | PipelinedMemoryBus | AXI4                   |
| 캐시      | 기본적으로 없음           | Instruction/Data Cache |
| 주변장치    | APB3               | APB3                   |
| 주요 용도   | 소형 FPGA SoC        | 외부 메모리를 포함한 확장 SoC     |

Briey도 `cpuPlugins`에 Plugin을 구성하고, CPU를 생성하는 방식입니다. 차이는 캐시가 포함된 버스 Plugin을 사용한다는 것입니다.

## 3. 우리 SoC에 적용할 올바른 구조

현재 프로젝트에서는 Murax의 구조를 참고하되, 처음부터 Murax 전체를 복제하지 않는 것이 좋겠습니다.

우선 다음과 같이 단순화하겠습니다.

## MyVexSoc 초기 구조

VexRiscv CPU

IBusSimplePlugin · DBusSimplePlugin · CsrPlugin

IBus

Instruction Fetch

DBus

Load / Store

SoC Interconnect

Instruction ROM

CPU 명령어 공급

Data RAM

Load / Store

APB3 Bridge

주변장치 연결

Interrupt

CSR 연결

### 실제 코드 작성 원칙

앞서 제안했던 다음 방식은 사용하지 않겠습니다.

```
cpuConfig.plugins += new SoCMemoryPlugin(...)
```

`SoCMemoryPlugin`을 CPU의 Plugin 목록에 넣어 외부 메모리 시스템을 구성하는 방식은 Murax의 설계 패턴이 아닙니다.

대신 다음 구조를 사용합니다.

```
class VexRiscvTop extends Component {

  val cpuConfig = CpuConfig.smallest()
  val cpu = new VexRiscv(cpuConfig)

  // CPU 생성 이후 Plugin 객체를 검색
  val iBusPlugin = cpu.plugins.collectFirst {
    case p: IBusSimplePlugin => p
  }.get

  val dBusPlugin = cpu.plugins.collectFirst {
    case p: DBusSimplePlugin => p
  }.get

  val csrPlugin = cpu.plugins.collectFirst {
    case p: CsrPlugin => p
  }.get

  // SoC의 메모리와 인터럽트 연결은
  // 이 Component의 영역에서 수행
}
```

이 코드는 Murax의 실제 사용 패턴을 간략화한 구조 예시입니다. 아직 ROM/RAM 연결을 포함한 완성 코드나 컴파일 성공 코드라고 주장하는 것은 아닙니다.

## 4. 앞서 발생한 오류에 대한 중요한 구분

이번 조사로 두 가지 문제를 분리해서 봐야 한다는 점도 확인했습니다.

1. Plugin 인터페이스 접근 문제: Murax처럼 CPU 생성 이후 최상위 SoC에서 `cpu.plugins`를 순회하는 구조로 변경해야 합니다. 앞서 제안한 사용자 정의 메모리 Plugin 방식은 철회하겠습니다.
2. `CsrPlugin.scala`의 `StreamFifoLowLatency` 초기값 오류: Murax의 정상적인 Plugin 연결 방식을 확인했다고 해서 이 오류까지 해결된 것은 아닙니다. 이 오류는 별도로 실제 로컬 소스의 해당 코드와 SpinalHDL 라이브러리 버전을 대조해야 합니다.

## 다음 진행 방향

저는 다음 순서가 가장 안전하다고 생각합니다.

1. `SoCMemoryPlugin` 제거
2. Murax 방식으로 `VexRiscvTop`에서 Plugin 객체 검색
3. IBus를 Instruction ROM에 직접 연결
4. DBus를 Data RAM에 연결
5. ROM에서 명령어를 가져와 CPU가 실행하는 시뮬레이션 확인
6. 이후 Main Bus Arbiter와 APB3를 추가

다음 단계에서는 우선 1\~2번을 반영한 `VexRiscvTop.scala`를 정확히 작성하고, CPU 생성 후 Plugin 인터페이스가 실제로 정상 접근되는지부터 확인하겠습니다. 이렇게 해야 또 다른 추측성 lifecycle 수정으로 시간을 낭비하지 않습니다.

참고로 이번 조사에서 기준으로 삼은 것은 사용자의 로컬 VexRiscv와 동일하다고 확인된 커밋 `baf7dc82f855eddaf5b0a20a6802c526f127e38`입니다.


---

---

좋습니다. 이번 단계에서는 실제 Murax의 구조를 그대로 따라 사용자 정의 `SoCMemoryPlugin`을 제거하고, SoC 최상위 컴포넌트에서 CPU Plugin을 찾아 연결하는 방식으로 바꾸겠습니다.

이번에는 ROM이나 RAM까지 한꺼번에 넣지 않겠습니다. 먼저 CPU를 생성하고 IBus, DBus, CSR 인터럽트 인터페이스에 접근해서 정상적으로 elaboration되는지 확인합니다.

Murax의 실제 구현은 `new VexRiscv(...)` 직후 `cpu.plugins`를 순회하며 `IBusSimplePlugin`, `DBusSimplePlugin`, `CsrPlugin`을 처리합니다.

## 1. `SoCMemoryPlugin` 제거

`VexRiscvTop.scala`에서 다음 코드를 제거합니다.

```
val socMemoryPlugin = new SoCMemoryPlugin(...)
cpuConfig.plugins += socMemoryPlugin
```

그리고 이번 단계에서는 `SoCMemoryPlugin.scala`를 사용하지 않습니다.

## 2. `VexRiscvTop.scala` 교체

아래 코드는 Murax의 Plugin 접근 방식을 따르면서, 아직 메모리가 없으므로 CPU의 메모리 요청을 받아들이지 않도록 구성한 단계별 검증용 코드입니다.

```

package myvexsoc

import spinal.core._
import vexriscv._
import vexriscv.plugin._

class VexRiscvTop extends Component {

  // ------------------------------------------------------------
  // CPU configuration
  // ------------------------------------------------------------

  val cpuConfig = CpuConfig.smallest()

  // ------------------------------------------------------------
  // CPU instance
  // ------------------------------------------------------------

  val cpu = new VexRiscv(cpuConfig)

  // ------------------------------------------------------------
  // SoC interrupt sources
  // ------------------------------------------------------------

  val externalInterrupt = False
  val timerInterrupt = False

  // ------------------------------------------------------------
  // Connect CPU plugins to the SoC
  // ------------------------------------------------------------

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

```

현재 코드의 동작

| 인터페이스              | 연결 상태           | 의미                 |
| ------------------ | --------------- | ------------------ |
| IBus CMD           | `ready = False` | 명령어 요청을 수락하지 않음    |
| IBus RSP           | `valid = False` | 명령어 응답을 전달하지 않음    |
| DBus CMD           | `ready = False` | 데이터 접근 요청을 수락하지 않음 |
| DBus RSP           | `ready = False` | 데이터 응답을 전달하지 않음    |
| External interrupt | `False`         | 외부 인터럽트 없음         |
| Timer interrupt    | `False`         | 타이머 인터럽트 없음        |

따라서 이 단계에서는 CPU가 명령어를 실행하지 않습니다. 목적은 CPU 생성, Plugin 탐색, 버스 방향 및 CSR 입력 연결을 확인하는 것입니다.

## 3. `build.sbt` 확인

앞서 보여 준 설정에는 SpinalHDL compiler plugin이 없었습니다. 아직 추가하지 않았다면 다음 항목을 반드시 추가해 주세요.

```
compilerPlugin(
  "com.github.spinalhdl" %% "spinalhdl-idsl-plugin" % spinalVersion
)
```

의존성 부분은 다음과 같아야 합니다.

```
libraryDependencies ++= Seq(
  "com.github.spinalhdl" %% "spinalhdl-core" % spinalVersion,
  "com.github.spinalhdl" %% "spinalhdl-lib" % spinalVersion,
  compilerPlugin(
    "com.github.spinalhdl" %% "spinalhdl-idsl-plugin" % spinalVersion
  ),
  "org.yaml" % "snakeyaml" % "1.8"
)
```

## 4. 실행

기존 `VexRiscvTopSim.scala`를 그대로 사용하되, 이전에 추가했던 `dut.socMemoryPlugin.debug...` 참조는 제거해야 합니다.

```
sbt clean
sbt update
sbt "runMain myvexsoc.VexRiscvTopSim"
```

이번 단계의 성공 기준

- `PhaseCreateComponent` 오류 없이 elaboration 완료
- `StreamFifoLowLatency` 초기값 오류가 재발하지 않음
- IBus / DBus / CSR 연결에서 null 참조가 발생하지 않음

참고: 이번 코드가 해당 오류까지 해결한다고 미리 단정하지는 않겠습니다. 만약 동일한 `StreamFifoLowLatency` 오류가 발생하면 그 결과를 그대로 보내 주세요. 그때는 `CsrPlugin`과 `IBusSimplePlugin`의 실제 생성 경로를 다시 추적하겠습니다.

## 다음 단계 예고

이번 검증이 통과하면 다음에는 `InstructionRom`을 추가합니다.

```
VexRiscv
   |
   | IBus
   v
Instruction ROM
   |
   v
ADDI x1, x0, 1
ADDI x2, x0, 2
ADD  x3, x1, x2
JAL  x0, 0
```

그때부터는 CPU가 실제로 명령어를 가져와 실행하는 것을 시뮬레이션으로 확인하겠습니다.

---

---


