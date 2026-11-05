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

```scala
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

좋은 지적입니다. 제가 구현 코드를 먼저 제시하고, 정작 GPIO가 무엇이고 CPU와 어떤 역할을 하는 장치인지 설명하지 않았습니다.

이 부분을 먼저 이해해야 합니다. GPIO는 CPU가 외부 세계와 데이터를 주고받는 가장 기본적인 주변장치 중 하나입니다. 이번에는 코드보다 개념과 설계 이유를 먼저 설명하겠습니다.

# 1. GPIO란 무엇인가?

GPIO는 General Purpose Input/Output의 약자입니다.

한국어로는 범용 입출력 장치라고 합니다.

쉽게 말하면 CPU가 외부의 전기 신호를 읽거나, 외부로 전기 신호를 출력할 수 있게 해 주는 장치입니다.

예를 들어 다음과 같은 장치들을 생각해 볼 수 있습니다.

![2. Button Input in ESP32. Hi, let’s move on to our next project… | by 18223051 Muhammad Aymar Barkhaya | Medium](https://images.openai.com/static-rsc-4/3WfvM2cRcFVVQBUpM3PJ6XOqSN_oVGkPkA5zKAwuXx5gRBRKeIBvGa9tADZ33gfsN2h7gLdN55-PJcwCH8kAMEwWt3eitcMRg5YDj8xwfdaQ8B_Jt6YkhOwVaCk47Gl5yUoPnw3f1Oa5AHyWHXXmN9-jLXSbH6ucBDu38MJ57TA)

LED 제어 (Output)

CPU가 GPIO 출력에 1을 기록하면 LED를 켜고, 0을 기록하면 끕니다.

![How use Raspberry Pi Pico GPIO with MicroPython code ?](https://images.openai.com/static-rsc-4/wBMKNOHxe84DMHDNCstRDZ2AIOqTtcQmKCGtfnXI6YSIJ9nELhO04jzKPdx47-smwekPHG45Dy56tl3TuI6TrcEpGMw5rw5nrrNnNYLr8Kkg_ghrov9NslVduyhFpniN5H13trQXHqhswQhBC9LhwYS8_aY8ET11gceWr9BC5SU)

버튼 읽기 (Input)

버튼을 누르면 외부 회로의 전압 상태가 바뀝니다. CPU는 GPIO 입력을 읽어서 버튼이 눌렸는지 판단합니다.

![STM32F103C8T6 Development Board  – Melodio](https://images.openai.com/static-rsc-4/mGpuWnib6_k48BTes4gkiGZjAuEDK3Duhb9ZN15Rgp3lbL0b_zAP6PK-S0vpWoMyK1YZfGFc4QdPe6KuPfmzhLZp96zSZzpvyYp3Gig63K2hKc_TZImMsxBFDGpInzaagsL-2DUojmXIhwDtxJhsyqC-pb3F8SccaCGAlLV5Cwg)

외부 핀 제어

GPIO 핀을 통해 센서, 스위치, LED, 다른 디지털 회로와 연결할 수 있습니다.

즉, 우리가 만드는 VexRiscv SoC에 GPIO를 넣는다는 것은 CPU가 메모리 Load/Store 명령을 이용해 외부 디지털 신호를 제어하고 읽을 수 있도록 한다는 뜻입니다.

# 2. GPIO는 왜 레지스터가 필요한가?

CPU는 외부 핀을 직접 제어하지 않습니다.

CPU와 GPIO 사이에는 DBus라는 데이터 통로가 있습니다.

CPU는 일반적인 메모리 접근과 똑같이 Load/Store 명령을 사용합니다.

예를 들어 다음과 같은 C 코드를 생각해 보겠습니다.

```c
#define GPIO_BASE  0x10000000

#define GPIO_OUT   (*(volatile unsigned int *)(GPIO_BASE + 0x00))
#define GPIO_IN    (*(volatile unsigned int *)(GPIO_BASE + 0x04))
#define GPIO_OE    (*(volatile unsigned int *)(GPIO_BASE + 0x08))

GPIO_OUT = 0x00000001;   // 출력 0번 핀을 HIGH로 설정
GPIO_OE  = 0x00000001;   // 0번 핀을 출력으로 설정

unsigned int value = GPIO_IN;  // 외부 입력 읽기
```

여기서 중요한 점은 `GPIO_OUT`, `GPIO_IN`, `GPIO_OE`가 일반 RAM에 저장된 데이터가 아니라는 것입니다.

이 주소들은 GPIO 하드웨어 내부의 레지스터에 연결되어 있습니다.

이런 방식을 Memory-Mapped I/O (MMIO) 라고 합니다.

![Memory-Mapped I/O vs Port-Mapped I/O in Embedded Systems | MURALI KRISHNA posted on the topic | LinkedIn](https://images.openai.com/static-rsc-4/OuGC4mqPrP6jH8U74bd8qlcRjI8jiFAMJ3ys2kPJBrejMOHcGNm2YDgSv1tnZmEU8EXYyY8b4FfEC4FFb9MowMzBTLK35EiM2Y0LSj5qXk2VOTyQaJlL1BEtuyV1zA-AZP76d8QSqIgnPSZPV6xSQeA1UgEBWv2deEpJUEfsOis)

[linkedin.com](https://www.linkedin.com/posts/murali-krishna-a9b883ba_define-embeddedsystems-embeddedc-activity-7451997028781363201-9Nss)

CPU 입장에서는 RAM에 접근하는 것과 동일하게 Load/Store 명령을 사용하지만, 실제로는 주소에 따라 RAM 또는 GPIO가 동작합니다.

예를 들면 다음과 같습니다.

| CPU 접근 주소    | 실제 동작            |
| ------------ | ---------------- |
| `0x00000000` | RAM 레지스터리 접근     |
| `0x00000004` | RAM 레지스터리 접근     |
| `0x10000000` | GPIO_OUT 레지스터 접근 |
| `0x10000004` | GPIO_IN 레지스터 접근  |
| `0x10000008` | GPIO_OE 레지스터 접근  |

이것이 우리가 DBusDecoder를 사용하는 이유입니다. 주소를 보고 어느 장치에 명령을 전달할지 결정합니다.

# 3. GPIO 레지스터 맵은 왜 이렇게 구성하는가?

우리가 정한 레지스터는 세 가지입니다.

## GPIO 레지스터 구성
```
GPIO_OUT
Read / Write
# 0x10000000
```
CPU가 외부로 출력할 데이터를 저장합니다.
```
GPIO_IN
Read Only
# 0x10000004
```
외부 핀에서 들어오는 입력 데이터를 읽습니다.
```
GPIO_OE
Read / Write
# 0x10000008
```
각 핀을 입력으로 사용할지 출력으로 사용할지 설정합니다.

각각 왜 필요한지 살펴보겠습니다.

## 3-1. GPIO_OUT: 출력 데이터

GPIO_OUT은 CPU가 외부로 내보낼 데이터를 저장합니다.

32비트 레지스터이므로 최대 32개의 GPIO 출력을 제어할 수 있습니다.

예를 들어 다음과 같이 기록한다고 가정하겠습니다.

```
GPIO_OUT = 0x00000005
```

2진수로 표현하면 다음과 같습니다.

```
0000 0000 0000 0000
0000 0000 0000 0101
```

따라서 다음과 같은 상태가 됩니다.

| GPIO 번호  | 출력 값 |
| -------- | ---- |
| GPIO[0]  | 1    |
| GPIO[1]  | 0    |
| GPIO[2]  | 1    |
| GPIO[3]  | 0    |
| ...      | ...  |
| GPIO[31] | 0    |

여기서 GPIO_OUT은 출력할 데이터를 저장하는 레지스터입니다.

실제 핀에서 출력할 수 있는지는 GPIO_OE 설정에 따라 달라집니다.

## 3-2. GPIO_IN: 외부 입력 데이터

GPIO_IN은 외부에서 들어오는 신호를 CPU가 읽을 수 있도록 하는 레지스터입니다.

예를 들어 버튼 0번과 버튼 2번이 눌렸다고 가정하겠습니다.

```
gpioIn = 0x00000005
```

CPU가 GPIO_IN을 Load하면 `0x00000005`를 읽을 수 있습니다.

중요한 차이가 있습니다.

- GPIO_OUT: CPU가 값을 기록하는 곳
- GPIO_IN: 외부 회로가 값을 제공하는 곳

따라서 GPIO_IN에는 CPU가 데이터를 기록할 필요가 없습니다.

## 3-3. GPIO_OE: 입력과 출력 방향 설정

여기가 GPIO에서 중요한 개념입니다.

GPIO 핀은 상황에 따라 입력으로도, 출력으로도 사용할 수 있습니다.

예를 들어 GPIO[0]은 LED에 연결하고 GPIO[1]은 버튼에 연결할 수 있습니다.

그렇다면 각 핀을 입력으로 사용할지 출력으로 사용할지 구분해야 합니다.

GPIO_OE는 Output Enable의 약자입니다.

이번 설계에서는 다음 규칙을 사용합니다.

- `1`: 출력으로 사용
- `0`: 입력으로 사용

예를 들어 다음과 같습니다.

```
GPIO_OE = 0x00000001
```

| 핀       | GPIO_OE | 방향 |
| ------- | ------- | -- |
| GPIO[0] | 1       | 출력 |
| GPIO[1] | 0       | 입력 |
| GPIO[2] | 0       | 입력 |
| GPIO[3] | 0       | 입력 |

즉, GPIO_OUT에 값이 들어 있어도 GPIO_OE가 0이면 해당 핀은 출력으로 활성화하지 않습니다.

하드웨어의 실제 출력 Enable 신호는 다음과 같습니다.

```
io.gpioOe := gpioOeReg
```

외부 GPIO 회로는 이 신호를 이용해 핀의 방향을 제어합니다.

# 4. 왜 byte mask를 사용하는가?

이제 다음 문장을 이해해 보겠습니다.

> GPIO_OUT과 GPIO_OE는 byte mask를 적용해 부분 쓰기를 지원합니다.

우리 SoC의 데이터 버스는 32비트입니다.

32비트는 4바이트이므로 다음처럼 구성됩니다.

32-bit register

Byte 3

31–24

Byte 2

23–16

Byte 1

15–8

Byte 0

7–0

CPU는 항상 32비트 전체를 쓰는 것만은 아닙니다.

RISC-V에는 다음과 같은 Store 명령이 있습니다.

| 명령   | 저장 크기 |
| ---- | ----- |
| `SB` | 8비트   |
| `SH` | 16비트  |
| `SW` | 32비트  |

예를 들어 GPIO_OUT이 다음과 같다고 가정하겠습니다.

```
GPIO_OUT = 0x11223344
```

CPU가 `SB` 명령으로 최하위 바이트만 `0xAA`로 변경하려고 합니다.

```
SB x1, 0(x2)
```

이때 byte mask가 없으면 32비트 전체를 덮어쓸 위험이 있습니다.

byte mask를 사용하면 변경할 바이트만 선택할 수 있습니다.

```
기존 값 : 0x11223344
쓰기 값 : 0x000000AA
mask    : 0b0001 <--- byte3 8bits all 0, byte2 all 0, byte1 all 0, byte0 all 1

결과    : 0x112233AA
```

그래서 우리가 작성한 코드에 다음 부분이 있습니다.

```
for (i <- 0 until 4) {
  when(io.bus.cmd.mask(i)) {
    gpioOutReg(i * 8 + 7 downto i * 8) :=
      io.bus.cmd.data(i * 8 + 7 downto i * 8)
  }
}
```

이 코드는 CPU가 요청한 byte lane만 변경합니다.

GPIO_OE에도 같은 원칙을 적용합니다.

이렇게 하는 이유는 GPIO를 일반적인 메모리처럼 Load/Store 명령으로 다룰 수 있도록 하기 위해서입니다.

# 5. 왜 GPIO_IN에 2단계 동기화가 필요한가?

이 부분은 디지털 회로 설계에서 매우 중요한 개념입니다.

CPU 내부 회로는 일반적으로 동일한 클럭에 맞춰 동작합니다.
하지만 외부 버튼이나 센서에서 들어오는 신호는 CPU 클럭과 무관하게 변할 수 있습니다.

예를 들어 CPU 클럭이 다음과 같다고 가정하겠습니다.

```
CLK:    ↑     ↑     ↑     ↑     ↑
        |     |     |     |     |
GPIO_IN:      0  0  1  1  1
```

외부 입력이 클럭 상승 에지 근처에서 바뀌면 CPU 내부 레지스터가 입력을 안정적으로 샘플링하지 못할 수 있습니다.
이 현상을 Metastability(준안정성) 라고 합니다.

간단히 말해 디지털 회로가 0이나 1 중 하나로 안정적으로 결정되지 못하는 현상입니다.
이를 완화하기 위해 일반적으로 2개의 플립플롭을 직렬로 연결합니다.

### 2-Stage Synchronizer
```
External GPIO Input
   |
   v 
Synchronizer FF 0(첫 번째 샘플링 레지스터)
   |
   v
Synchronizer FF 1(두 번째 샘플링 레지스터)
   |
   v
GPIO_IN Register Read(CPU가 읽는 동기화된 입력)
```

우리 코드에서는 다음 두 레지스터가 이 역할을 합니다.

```
val gpioInSync0 = Reg(Bits(32 bits)) init (0)
val gpioInSync1 = Reg(Bits(32 bits)) init (0)

gpioInSync0 := io.gpioIn
gpioInSync1 := gpioInSync0
```

첫 번째 레지스터에서 입력을 샘플링하고, 두 번째 레지스터에서 다시 샘플링합니다.

그 결과 첫 번째 레지스터에서 발생한 준안정성이 두 번째 레지스터까지 전달될 확률을 크게 낮춥니다.

주의할 점은 다음과 같습니다.

- 2단계 동기화는 준안정성 위험을 줄이지 완전히 제거하지는 않습니다.
- 외부 입력의 변화는 두 클럭 정도 지연되어 CPU에 전달됩니다.
- 실제 FPGA 구현에서는 이 레지스터들이 동기화기로 인식되도록 관련 제약이나 속성도 고려할 수 있습니다.

이것이 GPIO_IN에만 동기화 회로를 추가한 이유입니다. GPIO_OUT과 GPIO_OE는 CPU 클럭 도메인에서 생성되므로 이 입력 동기화가 필요하지 않습니다.

# 6. 지금 만든 GPIO의 전체 동작

지금까지 내용을 연결하면 다음과 같습니다.

![APB GPIO — CORE-V MCU  documentation](https://images.openai.com/static-rsc-4/gHb-3dV6dvOJ36QLN-yfJ8_YFBeyqzWBNJE02a-GTWX81uYeqFuaukSZtjI9nlwXxCsUxqSdSYvrLf62OLUu672v-lKUJNs06JJXhwzhjNcGDsLX1nReXjdLkWN7zGz16n6VgQZQFP5M_6XlHucP3gLfAJfIY4t7lTBDmYhUy34)

[docs.openhwgroup.org](https://docs.openhwgroup.org/projects/core-v-mcu/doc-src/ip-blocks/apb_gpio.html)

예를 들어 CPU가 다음과 같이 실행한다고 생각해 보겠습니다.

```
1. GPIO_OUT = 0x00000001
2. GPIO_OE  = 0x00000001
3. GPIO_IN  읽기
```

결과적으로:

1. GPIO[0]에 출력할 데이터가 1로 설정됩니다.
2. GPIO[0]이 출력 방향으로 설정됩니다.
3. GPIO 입력 신호를 CPU가 읽습니다.

이제 CPU는 외부 LED를 제어하고 버튼 상태도 읽을 수 있습니다.

# 7. 이번 설계에서 아직 알아야 할 한 가지

현재 설계에서는 GPIO_OUT과 GPIO_OE를 분리했습니다.

이것은 GPIO 하드웨어를 이해하기에 좋은 구조이지만, 실제 핀의 동작을 결정하는 것은 외부 회로입니다.

예를 들어 핀의 출력 동작은 개념적으로 다음과 같습니다.

```
if (gpioOe[i] == 1)
    pin[i] = gpioOut[i]
else
    pin[i] = High-Impedance
```

입력 모드에서는 외부 회로가 핀의 전압을 결정하고 GPIO_IN을 통해 읽습니다.

즉, GPIO_OE는 단순히 CPU가 읽는 설정값이 아니라 실제 핀의 출력 드라이버를 제어하는 신호입니다.

다음 단계에서는 이 개념을 기준으로 GPIO 테스트벤치를 만들겠습니다. 단순히 레지스터에 값을 쓰고 읽는 것뿐 아니라, 
외부 입력 변화와 출력 Enable까지 검증해야 GPIO가 제대로 구현되었다고 할 수 있습니다.
