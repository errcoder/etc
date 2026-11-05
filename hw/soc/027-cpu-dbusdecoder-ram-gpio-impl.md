좋습니다. 이제 두 버전의 차이와 현재 SoC에서 유지해야 할 동작이 명확해졌습니다.

이번 GPIO 확장은 최신 DBusDecoder의 Unmapped Load Error 처리 기능을 기준으로 진행하겠습니다. 
이전 버전의 GPIO 디코딩 코드를 그대로 복원하지는 않겠습니다. 그 버전에는 GPIO 응답 경로가 없고, GPIO 주소도 Base Address 하나만 인식하기 때문입니다.

## 1. 두 버전의 차이와 유지할 기능

| 기능                  | 최신 버전 | 이전 버전 |
| ------------------- | ----- | ----- |
| RAM 주소 디코딩          | 있음    | 있음    |
| GPIO 주소 디코딩         | 없음    | 있음    |
| RAM Load 응답         | 있음    | 있음    |
| GPIO Load 응답        | 없음    | 없음    |
| Unmapped Load Error | 있음    | 없음    |
| Unmapped Store 차단   | 있음    | 있음    |

최신 버전에서 이미 검증한 다음 동작은 반드시 유지합니다.

- RAM `0x00000000 ~ 0x000003FF`
- Unmapped Load는 수락하고 Error Response 반환
- Unmapped Store는 수락하지 않음
- 기존 RAM의 동기식 Load 응답 타이밍 유지

## 2. 새 주소 맵 확정

GPIO는 16-byte 영역으로 디코딩하고, 세 개의 레지스터를 배치하겠습니다.

| 주소           | 레지스터     | CPU 접근       |
| ------------ | -------- | ------------ |
| `0x10000000` | GPIO_OUT | Load / Store |
| `0x10000004` | GPIO_IN  | Load         |
| `0x10000008` | GPIO_OE  | Load / Store |
| `0x1000000C` | Reserved | Error        |

이렇게 하면 GPIO 레지스터별 주소를 구분할 수 있고, 예약 주소 접근도 검증할 수 있습니다.

## 3. DBusDecoder에서 가장 중요한 변경

기존의 `errorPending`만으로는 세 장치의 응답을 구분할 수 없습니다.

따라서 Load 요청이 수락될 때 응답 소유자를 기록하겠습니다.
```
Load Response Owner

RAM(Owner 1)    GPIO(Owner 2)    Error(Owner 3)
    \               |               /
     \              |              /
      \             |             /
       v            v            v      
           CPU DBus Response     
          (ready / data / error)
```

여기서 `DBusSimpleRsp.ready`는 일반적인 Ready/Valid의 ready와 다릅니다. VexRiscv의 DBusSimple에서는 응답 데이터가 유효하다는 표시입니다.

따라서 RAM과 GPIO의 응답을 각각 추적하고, CPU에는 해당 요청의 응답만 전달해야 합니다.

또한 기존 RAM의 `readSync` 응답 타이밍과 GPIO의 응답 타이밍을 일치시키겠습니다. 
CPU가 한 번에 하나의 Load 응답을 기다리는 구조를 전제로 하되, 응답이 반환되는 사이클에 다음 Load가 수락되는 경우도 고려해 소유자 갱신 우선순위를 정하겠습니다.

## 4. 구현 순서

이제 다음 순서로 실제 코드를 만들겠습니다.

1. `GpioPeripheral.scala`: GPIO_OUT, GPIO_IN, GPIO_OE 및 Byte Mask
2. `DBusDecoder.scala`: 세 응답 소스의 응답 소유권 추적
3. `VexRiscvTop.scala`: GPIO 외부 신호 연결
4. `VexRiscvTopSim.scala`: CPU 프로그램을 통한 전체 검증

다음 작업은 `GpioPeripheral.scala`입니다. 외부 입력은 2단계 동기화하고, 입력 레지스터는 읽기 전용으로 구현하겠습니다. 
출력 및 방향 레지스터는 읽기와 쓰기를 모두 지원하도록 하겠습니다.

---

---

좋습니다. 다음 단계는 `GpioPeripheral.scala` 구현입니다.

이번에는 DBusDecoder를 먼저 바꾸지 않고, GPIO 주변장치 자체를 독립적으로 완성하겠습니다. 그래야 GPIO 레지스터 동작과 Load 응답 타이밍을 먼저 검증한 뒤, 기존 RAM 및 Trap/MRET 경로에 안전하게 통합할 수 있습니다.

이번 구현의 기준은 현재 VexRiscv 프로젝트의 SpinalHDL 1.13.0입니다.

## 1. GPIO 레지스터 맵

| 주소           | 레지스터     | 접근           | 기능        |
| ------------ | -------- | ------------ | --------- |
| `0x10000000` | GPIO_OUT | R/W          | 출력 데이터    |
| `0x10000004` | GPIO_IN  | R            | 외부 입력 데이터 |
| `0x10000008` | GPIO_OE  | R/W          | 출력 방향 설정  |
| `0x1000000C` | Reserved | Load 시 Error | 예약 영역     |

GPIO_OUT과 GPIO_OE는 byte mask를 적용해 부분 쓰기를 지원합니다. GPIO_IN은 외부 입력을 두 단계 레지스터로 동기화한 값을 읽습니다.

## 2. `GpioPeripheral.scala`

```
package myvexsoc

import spinal.core._
import vexriscv.plugin.DBUS

class GpioPeripheral extends Component {

  val io = new Bundle {
    val bus = slave(DBusSimpleBus())

    val gpioIn  = in Bits (32 bits)
    val gpioOut = out Bits (32 bits)
    val gpioOe  = out Bits (32 bits)
  }

  // ------------------------------------------------------------
  // GPIO registers
  // ------------------------------------------------------------

  val gpioOutReg = Reg(Bits(32 bits)) init (0)
  val gpioOeReg  = Reg(Bits(32 bits)) init (0)

  // ------------------------------------------------------------
  // External input synchronizer
  // ------------------------------------------------------------

  val gpioInSync0 = Reg(Bits(32 bits)) init (0)
  val gpioInSync1 = Reg(Bits(32 bits)) init (0)

  gpioInSync0 := io.gpioIn
  gpioInSync1 := gpioInSync0

  // ------------------------------------------------------------
  // Address decoding
  //
  // address[3:2]:
  //   0 : GPIO_OUT
  //   1 : GPIO_IN
  //   2 : GPIO_OE
  //   3 : Reserved
  //
  // address[1:0] are ignored. This allows byte/halfword
  // accesses within a 32-bit register.
  // ------------------------------------------------------------

  val wordOffset = io.bus.cmd.address(3 downto 2)

  // ------------------------------------------------------------
  // Bus command acceptance
  // ------------------------------------------------------------

  io.bus.cmd.ready := True

  val writeFire = io.bus.cmd.fire && io.bus.cmd.wr

  // ------------------------------------------------------------
  // GPIO_OUT / GPIO_OE write
  // ------------------------------------------------------------

  when(writeFire) {
    when(wordOffset === U(0, 2 bits)) {
      for (i <- 0 until 4) {
        when(io.bus.cmd.mask(i)) {
          gpioOutReg(i * 8 + 7 downto i * 8) :=
            io.bus.cmd.data(i * 8 + 7 downto i * 8)
        }
      }
    } elsewhen (wordOffset === U(2, 2 bits)) {
      for (i <- 0 until 4) {
        when(io.bus.cmd.mask(i)) {
          gpioOeReg(i * 8 + 7 downto i * 8) :=
            io.bus.cmd.data(i * 8 + 7 downto i * 8)
        }
      }
    }
  }

  // GPIO_IN and Reserved writes are accepted but ignored.
  // Store Access Fault is not implemented by this peripheral.

  // ------------------------------------------------------------
  // External GPIO outputs
  // ------------------------------------------------------------

  io.gpioOut := gpioOutReg
  io.gpioOe  := gpioOeReg

  // ------------------------------------------------------------
  // Load response
  // ------------------------------------------------------------

  val readEnable =
    io.bus.cmd.fire && !io.bus.cmd.wr

  val readData = Bits(32 bits)
  readData := B(0, 32 bits)

  when(wordOffset === U(0, 2 bits)) {
    readData := gpioOutReg
  } elsewhen (wordOffset === U(1, 2 bits)) {
    readData := gpioInSync1
  } elsewhen (wordOffset === U(2, 2 bits)) {
    readData := gpioOeReg
  }

  val readDataReg =
    RegNextWhen(readData, readEnable) init (B(0, 32 bits))

  val readErrorReg =
    RegNext(readEnable && (wordOffset === U(3, 2 bits))) init (False)

  io.bus.rsp.ready := RegNext(readEnable) init (False)
  io.bus.rsp.data  := readDataReg
  io.bus.rsp.error := readErrorReg
}
```

> 주의: 위 코드의 `import vexriscv.plugin.DBUS`는 사용하지 않습니다. 이 프로젝트에서 `DBusSimpleBus`를 정의한 패키지에 맞춰 import해야 하므로, 기존 `SimpleDataRam.scala`에서 사용하는 import를 그대로 적용하세요. 불필요한 `DBUS` import는 제거하면 됩니다.

## 3. 동작에서 중요한 점

### 입력 동기화

```
gpioInSync0 := io.gpioIn
gpioInSync1 := gpioInSync0
```

외부 입력은 두 단계 레지스터를 거칩니다. 따라서 `gpioIn`을 변경한 직후 Load하면 이전 값이 읽힐 수 있습니다. 입력을 변경한 뒤 최소 두 번의 클럭 상승 에지를 지난 다음 읽도록 시뮬레이션해야 합니다.

### 부분 쓰기

예를 들어 다음 명령을 생각해 보겠습니다.

- 기존 GPIO_OUT: `0x11223344`
- 쓰기 데이터: `0xAABBCCDD`
- mask: `0b0101`

결과는 다음과 같습니다.

```
GPIO_OUT = 0x11BB33DD
```

mask의 각 비트가 해당 바이트 레인에만 쓰기를 허용합니다.

### Load 응답

`io.bus.rsp.ready`는 응답 유효 신호입니다. Load 명령이 수락된 다음 클럭에 한 사이클 동안 올라가며, `data`와 `error`도 같은 응답 사이클에 맞춰집니다.

Reserved 영역을 Load하면 `data = 0`, `error = true`가 됩니다.

## 4. 이번 단계의 검증 항목

아직 시뮬레이션을 통과했다고 가정하지 않겠습니다. 다음 테스트벤치에서 아래 항목을 각각 확인해야 합니다.

| 테스트                | 기대 결과               |
| ------------------ | ------------------- |
| 초기 상태              | OUT=0, OE=0         |
| GPIO_OUT 전체 쓰기/읽기  | 쓴 값과 읽은 값 일치        |
| GPIO_OE 전체 쓰기/읽기   | 쓴 값과 읽은 값 일치        |
| GPIO_OUT byte mask | 지정된 바이트만 변경         |
| GPIO_OE byte mask  | 지정된 바이트만 변경         |
| GPIO_IN 동기화        | 2단계 동기화 후 입력값 읽기    |
| GPIO_IN 쓰기         | 레지스터 값 변경 없음        |
| Reserved Load      | 응답 valid 및 error 확인 |
| 응답 타이밍             | Load 수락 후 한 사이클 응답  |

다음은 `GpioPeripheralSim.scala`입니다. 먼저 이 컴포넌트를 프로젝트에 추가하고 import를 기존 코드와 일치시킨 뒤 컴파일해 주세요. 컴파일 결과를 확인하면 그 환경에 맞춰 응답 타이밍까지 검증하는 테스트벤치를 진행하겠습니다.

---

---


