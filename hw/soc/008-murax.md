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

실제 소스의 구성은 이와 같으며, 각 Plugin의 설정은 CPU의 기능과 파이프라인 동작을 결정합니다.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



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

참고로 이번 조사에서 기준으로 삼은 것은 사용자의 로컬 VexRiscv와 동일하다고 확인된 커밋 `baf7dc82f855eddaf5b0a20a6802c526f127e38`입니다.&#x20;
