```
아니요, `r.showPublic()`이 아니라 **`r.simPublic()`**을 사용해야 합니다.

```

SpinalHDL 시뮬레이션 환경에서 내부 레지스터나 신호(`r`)에 접근하려면 해당 신호에 `simPublic()` 태그를 붙여주어야 
최적화(pruning) 과정에서 제거되지 않고 시뮬레이션 내에서 읽거나 쓸 수 있습니다.

사용하는 방법은 크게 두 가지가 있습니다.

### 1. 컴포넌트 내부에서 직접 선언하기 (`r.simPublic()`)

컴포넌트 정의부 안에서 레지스터를 선언할 때 바로 `simPublic()`을 호출할 수 있습니다.

```scala
class ACompo extends Component {
  val io = new Bundle {
    // ...
  }
  
  val r = Reg(UInt(8 bits)) init(0)
  r.simPublic() // 시뮬레이션에서 접근할 수 있도록 지정
  
  // 로직 작성...
}

```

### 2. 시뮬레이션 컴파일 시점에 지정하기 (`dut.r.simPublic()`)

컴포넌트 코드를 직접 수정하기 어렵거나 외부에서 동적으로 열어주고 싶을 때는 시뮬레이션 설정(SimConfig) 단계에서 지정할 수 있습니다.

```scala
SimConfig.compile {
  val dut = new ACompo()
  dut.r.simPublic() // 외부에서 simPublic 지정
  dut
}.doSim { dut =>
  dut.clockDomain.forkStimulus(10)
  
  // 시뮬레이션 중 r 값 읽기 및 쓰기
  println(dut.r.toInt) // 값 읽기
  dut.r #= 42          // 값 강제 할당 (필요한 경우)
}

```

### 💡 시뮬레이션에서의 접근 방식 요약

`simPublic()` 처리가 된 신호(`dut.r`)는 시뮬레이션 블록 안에서 다음과 같이 다룰 수 있습니다:

* **값 읽기:** `dut.r.toInt`, `dut.r.toBigInt`, `dut.r.toBoolean` 등
* **값 쓰기:** `dut.r #= 123` (`#=` 연산자 사용)

---

---

시뮬레이션에서 reset 신호를 off 시키는 방법은?

---

SpinalHDL 시뮬레이션에서 리셋 신호를 끄는(Deassert) 방법은 **클럭 도메인을 선언한 방식과 테스트벤치 제어 방식**에 따라 몇 가지로 나뉩니다.

---

### 1. `forkStimulus`를 통한 자동 해제 (가장 추천)

가장 일반적인 방법인 `forkStimulus`를 사용하면, 시뮬레이션 시작 시 **기본 16클럭 동안 리셋이 유지된 후 자동으로 리셋이 off(deassert)** 됩니다.

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 10 타임 unités 주기로 클럭 생성 및 초기 16클럭 동안 리셋 자동 수행 후 off
  dut.clockDomain.forkStimulus(period = 10) 
  
  dut.clockDomain.waitSampling()
  // 이 시점부터는 리셋이 자동으로 off된 상태입니다.
}

```

* **리셋 유지 클럭 수 변경하기:** `resetCycles` 인자를 주어 리셋 지속 시간을 조절할 수 있습니다.
```scala
// 예: 5클럭 동안만 리셋 유지 후 자동 off
dut.clockDomain.forkStimulus(period = 10, sleepDuration = 0, resetCycles = 5)

```

---

### 2. `deassertReset()` 메서드로 직접 제어하기

직접 리셋 타이밍을 세밀하게 제어하고 싶다면 `forkStimulus` 대신 클럭과 리셋을 따로 코딩한 뒤, **`deassertReset()`** 함수를 호출하여 리셋을 끌 수 있습니다.

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 1. 클럭 토글 스레드 시작
  dut.clockDomain.forkClock(period = 10)
  
  // 2. 리셋 수동 조작
  dut.clockDomain.assertReset()   // 리셋 ON (Active)
  dut.clockDomain.waitSampling(5) // 5클럭 대기
  
  dut.clockDomain.deassertReset() // 리셋 OFF (Deassert)
  
  // 이후 시뮬레이션 로직 진행...
}

```

---

### 3. 외부 I/O 포트로 연결된 리셋 직접 제어하기 (`#=`)

만약 하드웨어 디자인 내부에서 외부 입력 핀(`io.reset`)을 받아 `ClockDomain`에 매핑한 형태라면, 시뮬레이션 안에서 해당 I/O 핀에 직접 값을 주어 리셋을 off 시킬 수 있습니다.

**하드웨어 정의 예시:**

```scala
class ACompo extends Component {
  val io = new Bundle {
    val clk   = in Bool()
    val reset = in Bool()
    // ...
  }
  val myClockDomain = ClockDomain(io.clk, io.reset)
  val core = new ClockingArea(myClockDomain) {
    val r = Reg(UInt(8 bits)) init(0)
    r.simPublic()
  }
}

```

**시뮬레이션 제어 예시:**

```scala
SimConfig.compile(new ACompo()).doSim { dut =>
  // 클럭 토글 시작
  fork {
    dut.io.clk #= false
    while(true) {
      sleep(5)
      dut.io.clk #= !dut.io.clk.toBoolean
    }
  }

  // Active-High 리셋인 경우 (1이 리셋, 0이 정상 동작)
  dut.io.reset #= true   // 리셋 ON
  sleep(50)
  dut.io.reset #= false  // 리셋 OFF (정상 동작 시작)
}

```
