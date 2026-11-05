Trap Error Path 검증 성공
좋습니다! 이번에는 BNE 분기와 오류 루프 반복까지 실제 시뮬레이션으로 확인했습니다.
1. 실행 흐름 분석
Cycle	PC / 동작	의미
9	STORE 0x00000000	RAM[0] = 1
15	LOAD 0x20000000	잘못된 주소 접근
16	LOAD RSP error=true	Load Access Fault 응답
19	0x80000020	Trap Handler 진입
20	0x80000024	비교값 6 설정
21	0x80000028	BNE 실행 위치
22	0x8000002C	파이프라인의 선행 Fetch
30	0x8000004C	오류 루프 진입

여기서 중요한 부분이 있습니다.

0x8000002C와 0x80000030, 0x80000034의 Fetch가 관찰되었다고 해서 해당 명령어들이 실제로 실행되었다는 뜻은 아닙니다.

분기 전 파이프라인에서 발생한 선행 Fetch일 수 있습니다. 이번 테스트에서는 해당 주소의 복구 경로 Store가 발생하지 않았고, 오류 루프 Fetch가 33회 관찰되었으므로 분기 이후 동작도 확인할 수 있습니다.
2. 최종 검증 결과
검증 항목	결과
Load Access Fault 발생	PASS
Error Response 수신	PASS
BNE 분기 목적지 Fetch	PASS
오류 루프 반복 33회	PASS
복구 경로 CSR Store 없음	PASS
복귀 프로그램 Store 없음	PASS
전체 Store 수 = 1	PASS

3. 지금까지 완성한 Trap 처리
현재까지 다음 두 가지 경로를 모두 검증했습니다.
정상 복구 경로
1. Load Access Fault 발생
2. mcause 확인 및 비교
3. mepc 수정
4. MRET 실행
5. 원래 프로그램 재개
검증 완료

복구 불가능한 예외 경로
1. Load Access Fault 발생
2. mcause 비교 실패
3. BNE 분기
4. 오류 루프 진입
5. 오류 루프 반복
검증 완료

4. 다음 학습 단계
다음은 GPIO Memory-Mapped I/O를 추가하는 것을 추천합니다.

현재 CPU는 RAM에만 접근할 수 있습니다. 이제 DBusDecoder에 GPIO 주소 영역을 추가하면 CPU가 Load/Store 명령으로 외부 입출력을 제어할 수 있습니다.

예정 구조:
             VexRiscv
                 |
               DBus
                 |
             DBusDecoder
              /       \
           RAM         GPIO
          0x0000      0x1000


GPIO를 추가하면 다음을 검증할 수 있습니다.
1. CPU의 Store로 GPIO 출력 변경
2. GPIO 입력값을 CPU가 Load
3. 주소에 따른 RAM/GPIO 디코딩
4. 미매핑 주소의 Load Access Fault 유지
기존 Trap, MRET, Error Response 기능은 그대로 유지하면서 진행하겠습니다.

---

---

맞습니다. 제가 이전 학습 이력을 제대로 반영하지 못했습니다. 죄송합니다.

과거 대화를 다시 확인해 보니, 2026년 11월 3일에 이미 CPU → DBusDecoder → RAM/GPIO 구조를 다뤘습니다.

당시 진행한 내용은 다음과 같습니다.

- RAM 주소 영역: `0x00000000 ~ 0x000003FF`
- GPIO 주소: `0x10000000`
- `DBusDecoder`에서 RAM과 GPIO 주소 디코딩
- `SimpleGpio`를 통한 GPIO 출력 제어
- RAM Load 응답과 GPIO Store 경로 분리
- GPIO 출력 `0x55` 및 RAM 값 보존을 확인하는 테스트 구성

따라서 GPIO를 다음 학습 단계로 다시 제안한 것은 명백히 잘못된 판단입니다.&#x20;

## 현재 학습 진도 정정

| 기능                         | 상태         |
| -------------------------- | ---------- |
| VexRiscv + Instruction ROM | 완료         |
| DBusDecoder + RAM          | 완료         |
| DBusDecoder + GPIO         | 이전 학습에서 진행 |
| Load / Store               | 완료         |
| Unmapped Load Error        | 완료         |
| Trap / CSR                 | 완료         |
| MRET 정상 복구                 | 검증 완료      |
| BNE 기반 예외 분기               | 검증 완료      |
| Error Loop                 | 검증 완료      |

GPIO는 다시 구현하지 않겠습니다.

현재 진행 위치는 RAM/GPIO 버스 구조가 아니라, Trap 및 예외 처리까지 검증한 VexRiscv SoC입니다.

앞으로는 이미 학습한 내용을 반복하지 않고, 이 지점에서 이어가겠습니다.

---

---

맞습니다. 현재 확인 가능한 과거 학습 내용과 제시한 `DBusDecoder` 코드를 기준으로 보면, GPIO는 CPU에서 GPIO로 쓰는(Store) 기능을 구현했고, GPIO 입력을 CPU가 읽는(Load) 기능은 완성하지 않았습니다.

다만 제시한 `DBusDecoder` 코드에도 중요한 부분이 있습니다.

```
io.gpio.cmd.valid := io.cpu.cmd.valid && gpioHit
io.gpio.cmd.payload := io.cpu.cmd.payload
```

GPIO로 명령을 전달하지만, 응답 경로는 RAM에 대해서만 구현되어 있습니다.

```
io.cpu.rsp.ready := ramLoadPending && io.ram.rsp.ready
io.cpu.rsp.data := io.ram.rsp.data
io.cpu.rsp.error := io.ram.rsp.error
```

즉, 현재 구조에서는 GPIO Load 응답이 CPU로 전달되지 않습니다.

## 남은 GPIO 기능

| 기능                 | 상태    |
| ------------------ | ----- |
| CPU → GPIO Store   | 구현    |
| GPIO 출력 레지스터       | 구현    |
| 외부 GPIO 입력 신호      | 확인 필요 |
| GPIO 입력 레지스터 읽기    | 미완성   |
| GPIO Load 응답 → CPU | 미완성   |

따라서 다음 단계는 GPIO를 다시 만드는 것이 아니라 GPIO 입력 레지스터와 DBus Load 응답 경로를 추가하는 것이 맞습니다.

이때 RAM과 GPIO의 응답이 겹치지 않도록 응답 소유권을 추적하는 구조도 함께 점검해야 합니다. 특히 현재 `ramLoadPending`만으로는 GPIO Load 응답을 처리할 수 없습니다.

---

---

확인했습니다. 이 `SimpleGpio` 코드라면 현재 GPIO는 명확하게 write-only로 구현되어 있습니다.

## 현재 SimpleGpio 동작

```
when(io.bus.cmd.fire && io.bus.cmd.wr) {
  gpioValue := io.bus.cmd.payload.data
}
```

- `cmd.fire && cmd.wr`일 때만 GPIO 레지스터 갱신
- `io.value`를 통해 저장된 값을 외부 출력
- `cmd.ready := True`이므로 모든 명령을 수락
- `rsp.ready := False`이므로 Load 응답은 발생하지 않음

따라서 CPU가 GPIO 주소에 Store하면 출력값은 변경되지만, Load하면 CPU에 유효한 응답이 돌아오지 않습니다.

## 다음에 추가할 기능

기존 출력 기능을 유지하면서 입력 기능을 추가하는 것이 자연스럽습니다.

확장할 GPIO 인터페이스

CPU Store → GPIO 출력 레지스터

기존

GPIO 외부 입력 → GPIO 입력 레지스터

추가

CPU Load → GPIO 입력값 응답

추가

예를 들어 다음처럼 구성할 수 있습니다.

- `0x10000000`: GPIO 출력 레지스터 (Write)
- `0x10000004`: GPIO 입력 레지스터 (Read)

다만 구현에 앞서 한 가지 중요한 사항이 있습니다. `DBusDecoder`의 RAM/GPIO 응답 선택 로직도 함께 수정해야 합니다. GPIO Load를 추가하면서 RAM의 응답 경로가 깨지지 않도록 해야 하기 때문입니다.

다음은 `SimpleGpio`의 입출력 인터페이스와 DBusDecoder의 응답 경로를 함께 분석한 뒤 수정하는 순서로 진행하겠습니다.

---

---

좋습니다. 이번에는 GPIO를 단순히 Store로 출력값을 변경하는 수준에서 끝내지 않고, 실제 SoC 주변장치로 사용할 수 있는 32-bit Memory-Mapped GPIO를 완성하고 CPU 명령어로 검증하겠습니다.

기존 `SimpleGpio`는 참고만 하고, 필요하면 새로 작성하겠습니다. 이전에 구현한 RAM/GPIO 디코딩 구조와 Trap/MRET 기능도 유지하겠습니다.

# GPIO 구현 목표

이번 GPIO는 입력과 출력을 모두 지원하는 기본적인 양방향 GPIO Peripheral로 설계하겠습니다.

## 1. GPIO 레지스터 맵

GPIO Base Address는 기존 주소인 `0x10000000`을 유지합니다.

| 주소           | 레지스터     | 접근  | 기능           |
| ------------ | -------- | --- | ------------ |
| `0x10000000` | GPIO_OUT | R/W | GPIO 출력값     |
| `0x10000004` | GPIO_IN  | R   | 외부 GPIO 입력값  |
| `0x10000008` | GPIO_OE  | R/W | 핀별 출력 활성화 설정 |

모든 레지스터는 32-bit입니다.

예를 들어 다음과 같이 동작합니다.

- GPIO_OUT에 `0x00000055`를 Store하면 출력 레지스터가 `0x55`가 됩니다.
- GPIO_OE에 `0x000000FF`를 Store하면 하위 8개 핀을 출력으로 설정합니다.
- 외부 입력이 `0x000000A5`이면 GPIO_IN을 Load했을 때 `0xA5`를 읽습니다.
- GPIO_OUT과 GPIO_OE는 Byte Mask를 지원합니다.

외부 핀은 다음 인터페이스로 분리하겠습니다.

```
gpioIn  : in Bits(32 bits)
gpioOut : out Bits(32 bits)
gpioOe  : out Bits(32 bits)
```

`gpioIn`에는 입력 동기화 회로도 고려하겠습니다. 실제 외부 핀 입력은 CPU Clock과 비동기일 수 있기 때문입니다.

## 2. 전체 SoC 구조

VexRiscv CPU

Load / Store Instructions

DBusSimpleBus

DBusDecoder

Address Decode + Response Routing

SimpleDataRam

0x00000000

GpioPeripheral

0x10000000

GPIO External Interface

gpioIn

gpioOut

gpioOe

특히 DBusDecoder는 기존의 `ramLoadPending`만으로는 부족합니다. RAM, GPIO, Unmapped Load의 응답 소유권을 구분해야 합니다.

각 장치의 응답 타이밍도 맞춰야 합니다. GPIO Load 응답을 추가하면서 기존 RAM의 동기식 Load 응답이 깨지는 일이 없도록 하겠습니다.

# 3. 시뮬레이션 검증 계획

단순히 GPIO 레지스터를 읽고 쓰는 테스트 하나로 끝내지 않고, CPU가 실제 명령을 실행하도록 검증하겠습니다.

| 테스트            | 검증 내용                   |
| -------------- | ----------------------- |
| GPIO Output    | CPU Store → GPIO_OUT 변경 |
| GPIO Direction | CPU Store → GPIO_OE 변경  |
| GPIO Input     | 테스트벤치 입력 → CPU Load     |
| GPIO Readback  | GPIO_OUT / GPIO_OE Load |
| Byte Mask      | 일부 바이트만 Store           |
| RAM Isolation  | GPIO 접근이 RAM을 변경하지 않는지  |
| Address Decode | RAM/GPIO 주소 구분          |
| Unmapped Load  | 기존 Load Access Fault 유지 |
| Trap / MRET    | 기존 예외 처리 기능 유지          |

그리고 파형에서도 다음 신호를 관찰하겠습니다.

- CPU DBus `cmd.valid`, `cmd.ready`, `cmd.wr`
- CPU DBus `cmd.address`, `cmd.payload.data`, `cmd.mask`
- GPIO 입력 동기화 레지스터
- GPIO_OUT, GPIO_OE
- GPIO Load 응답 `rsp.ready`, `rsp.data`, `rsp.error`
- RAM Load 응답과 GPIO Load 응답의 구분

# 4. 진행 순서

코드를 한꺼번에 작성하지 않고 다음 순서로 진행하겠습니다.

1. 현재 DBusDecoder와 DBusSimplePlugin의 응답 타이밍 확인
2. GPIO 레지스터 인터페이스 및 동기화 구조 설계
3. GpioPeripheral 구현
4. DBusDecoder의 RAM/GPIO/Error 응답 라우팅 확장
5. CPU 프로그램을 이용한 GPIO 시뮬레이션
6. Trap/MRET 및 Unmapped Load 회귀 테스트

특히 이전에 발생했던 시뮬레이션 타이밍 오류를 피하기 위해 각 단계에서 실제 handshake와 응답 시점을 확인하겠습니다. 코드를 제시하기 전에 관련 SpinalHDL/VexRiscv 구현과 신호 방향도 확인하겠습니다.

## 먼저 현재 DBusDecoder 전체 코드를 보여주세요.

과거에 제시한 코드는 Unmapped Load Error 처리까지 추가된 버전과 초기 RAM/GPIO 버전이 다릅니다. 현재 실제 사용 중인 코드를 기준으로 작업해야 기존 기능을 훼손하지 않습니다.

현재 사용 중인 `DBusDecoder.scala` 전체 코드를 보내주세요. 그 코드를 기준으로 GPIO 응답 경로까지 정확하게 설계하겠습니다.

---

---

