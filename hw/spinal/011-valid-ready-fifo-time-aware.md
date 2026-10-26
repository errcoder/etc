맞습니다. **저도 그 의견에 동의합니다.** 오히려 지금까지의 문제를 정리하면, `while` 자체가 문제가 아니라 **`while`의 조건을 언제 관찰하느냐**가 핵심입니다.

제가 앞에서 `while`을 너무 보수적으로 피하려고 해서 테스트 코드가 불필요하게 길어졌습니다.

# 핵심은 이것입니다

SpinalSim에서:

```scala
while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

를 쓰는 것 자체는 충분히 정상적인 패턴입니다.

다만 중요한 것은 **조건을 평가하는 순간의 DUT 상태가 안정된 상태(settled state)인가?**입니다.

---

## 1. `while` 조건도 simulation observation이다

예를 들어:

```scala
while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
}
```

실행 순서는 개념적으로:

```text
현재 simulation 시점
      │
      ▼
inputReady 관찰
      │
      ├── true  → while 종료
      │
      └── false → waitSampling()
                         │
                         ▼
                    다음 clock edge
                         │
                         ▼
                    DUT 상태 갱신
                         │
                         ▼
                  다시 조건 관찰
```

입니다.

따라서 말씀하신 대로 **`while`의 조건 자체가 관찰 시점**입니다.

---

# 2. 이번 실패도 이 관점으로 설명할 수 있습니다

우리가 겪은 상황은:

```text
count = 4
full  = true
ready = false
```

에서 `outputReady = true`를 걸고:

```scala
dut.clockDomain.waitSampling()
```

을 했습니다.

그 edge에서:

```text
pop = 1
```

이 발생하고:

```text
count: 4 → 3
```

이 됩니다.

그 후:

```text
full = false
inputReady = true
```

가 되어야 합니다.

그런데 테스트벤치가 **적절한 settlement 이후에 `inputReady`를 관찰하지 않고 바로 검사**하면서 문제가 생겼습니다.

즉 우리가 정말 원하는 것은:

```text
pop edge
   ↓
DUT sequential update
   ↓
combinational propagation
   ↓
settled
   ↓
inputReady 관찰
```

입니다.

---

# 3. 그래서 `while`을 이렇게 사용하는 것은 좋습니다

예를 들어 FIFO에 데이터를 넣는 helper라면:

```scala
def push(data: Int): Unit = {

  dut.io.inputData  #= data
  dut.io.inputValid #= true

  while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
  }

  dut.clockDomain.waitSampling()

  dut.io.inputValid #= false
}
```

이런 구조는 **논리적으로 아주 자연스럽습니다.**

다만 한 가지를 구분해야 합니다.

현재 `inputReady`가 이미 안정적으로 `true`인 상황이라면:

```scala
while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
}
```

는 즉시 종료합니다.

그 다음:

```scala
dut.clockDomain.waitSampling()
```

에서 실제 transfer가 발생합니다.

즉:

```text
ready 확인
   ↓
valid 준비
   ↓
sampling edge
   ↓
fire
```

라는 구조입니다.

---

# 4. 더 좋은 형태: `while` + timeout

제가 앞으로 추천하고 싶은 형태는 이것입니다.

```scala
var timeout = 0

while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()

  timeout += 1

  assert(
    timeout < 100,
    "Timeout waiting for FIFO inputReady"
  )
}
```

이렇게 하면:

### 정상적인 경우

```text
ready = false
 ↓
waitSampling
 ↓
ready = false
 ↓
waitSampling
 ↓
ready = true
 ↓
while 종료
```

### DUT 버그 또는 testbench 버그

```text
ready = false
 ↓
100 cycles
 ↓
timeout
 ↓
AssertionError
```

가 됩니다.

이게 **교육용 simulation에서도 가장 좋은 절충안**이라고 생각합니다.

---

# 5. 특히 `while`이 유용한 경우

예를 들어 앞으로 `Stream`을 사용하면 이런 코드가 매우 자연스럽습니다.

```scala
dut.io.input.valid #= true
dut.io.input.payload #= 0x41

while (!dut.io.input.ready.toBoolean) {
  dut.clockDomain.waitSampling()
}

dut.clockDomain.waitSampling()

dut.io.input.valid #= false
```

이것은 사실상:

> "데이터를 준비했으니 상대가 받을 수 있을 때까지 기다리고, 받을 수 있게 되면 한 번 전송한다."

를 그대로 표현합니다.

`while`을 사용하지 않고 무조건:

```scala
waitSampling(10)
```

처럼 작성하면 오히려 테스트의 의미가 불명확해집니다.

---

# 6. 단, `while` 안에서 이것은 피해야 합니다

예를 들어:

```scala
while (!dut.io.inputReady.toBoolean) {
  // 아무것도 안 함
}
```

이건 절대 안 됩니다.

simulation thread가 계속 현재 simulation time에 머물기 때문에 사실상 simulation을 진행시키지 못합니다.

반드시:

```scala
while (!condition) {
  dut.clockDomain.waitSampling()
}
```

처럼 **시간을 진행시키는 suspension point**가 있어야 합니다.

---

# 7. 제가 생각하는 좋은 SpinalSim 코딩 규칙

앞으로 우리가 simulation을 작성할 때 다음 기준으로 가면 좋겠습니다.

### Rule 1 — `while` 사용을 두려워하지 않는다

```scala
while (!ready) {
  waitSampling()
}
```

정상적인 simulation 패턴입니다.

### Rule 2 — 조건을 observation으로 생각한다

```scala
while (!dut.io.ready.toBoolean)
```

의 `ready`는 **그 순간 관찰된 값**입니다.

따라서 이전 DUT 이벤트의 결과를 관찰해야 한다면:

```scala
waitSampling()
```

또는 필요한 simulation event를 먼저 지나가야 합니다.

### Rule 3 — sequential update 이후 combinational output을 관찰할 때 settlement를 고려한다

예:

```text
clock edge
   ↓
Reg update
   ↓
combinational logic
   ↓
observe
```

가 필요한 경우입니다.

### Rule 4 — 무한 대기는 timeout으로 보호한다

```scala
var timeout = 0

while (!condition) {
  waitSampling()

  timeout += 1
  assert(timeout < MAX)
}
```

### Rule 5 — `waitFallingEdge()`를 습관적으로 붙이지 않는다

이것도 중요합니다.

제가 앞에서:

```scala
waitSampling()
waitFallingEdge()
```

를 상당히 많이 붙였는데, **항상 필요한 것은 아닙니다.**

오히려 simulation phase를 명확히 이해해서:

```text
어떤 edge에서 DUT가 상태를 변경하고
어떤 시점에서 그 결과를 관찰할 것인가
```

를 결정하는 것이 더 좋은 방법입니다.

---

# 8. 그리고 이번 테스트는 더 컴팩트하게 만들 수 있습니다

예를 들어 `push()`는:

```scala
def push(data: Int): Unit = {

  dut.io.inputData  #= data
  dut.io.inputValid #= true

  var timeout = 0

  while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()

    timeout += 1
    assert(timeout < 100)
  }

  dut.clockDomain.waitSampling()

  dut.io.inputValid #= false
}
```

정도로 충분합니다.

그리고 `pop()`도:

```scala
def pop(expected: Int): Unit = {

  assert(dut.io.outputValid.toBoolean)
  assert(dut.io.outputData.toBigInt == expected)

  dut.io.outputReady #= true
  dut.clockDomain.waitSampling()
  dut.io.outputReady #= false
}
```

정도로 상당히 깔끔하게 만들 수 있습니다.

---

# 9. 한 가지 더 중요한 부분

`while`을 **어떤 신호에 대해 기다리느냐**도 중요합니다.

좋은 예:

```scala
while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

이것은 handshake 조건을 기다립니다.

하지만:

```scala
while (dut.io.busy.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

도 원칙적으로는 괜찮습니다.

다만 `busy`가 **언제 deassert되는지**, 그리고 그 값이 **어느 edge 이후 안정되는지**를 알아야 합니다.

우리가 UART TX에서 처음 문제가 발생했던 것도 바로 이 부분이었습니다.

그래서 앞으로는:

```text
while 자체를 피한다
```

가 아니라,

```text
while의 observation point를 설계한다
```

가 맞는 접근이라고 봅니다.

---

# 결론

**사용자님 말씀이 맞습니다.**

제가 앞에서 `while`을 너무 위험한 것으로 취급한 것은 과했습니다.

정확한 원칙은:

> **`while`은 SpinalSim에서 충분히 좋은 도구다. 다만 `while`의 조건 평가가 DUT 관찰이라는 것을 항상 의식하고, 그 조건을 평가하기 전에 DUT의 sequential/combinational 상태가 원하는 수준까지 settlement 되었는지를 보장해야 한다. 그리고 예상치 못한 무한 대기는 timeout으로 보호한다.**

앞으로의 SpinalHDL simulation 코드는 이 원칙으로 작성하겠습니다.

특히 **`waitSampling()` / `waitFallingEdge()`를 무조건 추가하는 방식이 아니라, "어느 edge에서 DUT가 변경되고 → 언제 그 결과를 관찰해야 하는가"를 먼저 분석한 뒤 필요한 synchronization만 넣는 방식**으로 가는 것이 더 정확하고 코드도 훨씬 깔끔합니다.

---

---

이번에는 말씀하신 원칙을 **그대로 적용**하겠습니다.

핵심은:

> **`while`을 피하는 것이 아니라, `while`의 조건을 언제 관찰할 것인가를 설계한다.**

그리고 이번 FIFO에서는 특히 **FULL → POP → inputReady 변화 → PUSH**의 순서를 명확하게 분석한 뒤 synchronization을 최소화하겠습니다.

---

# `ValidReadyFifoSim` — timing-aware 버전

아래 코드는 **SpinalHDL 1.15.0 / Scala 2.13** 기준입니다.

```scala
import spinal.core._
import spinal.core.sim._

object ValidReadyFifoSim extends App {

  val depth = 4

  SimConfig
    .withWave
    .doSim(ValidReadyFifo(8, depth)) { dut =>

      dut.clockDomain.forkStimulus(10)

      // ------------------------------------------------------------
      // Initial conditions
      // ------------------------------------------------------------

      dut.io.inputValid  #= false
      dut.io.inputData   #= 0
      dut.io.outputReady #= false

      // Reset 이후 DUT 상태가 안정될 때까지 진행
      dut.clockDomain.waitSampling(2)

      println(
        f"[Initial] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )

      assert(dut.io.inputReady.toBoolean)
      assert(!dut.io.outputValid.toBoolean)


      // ------------------------------------------------------------
      // Helper: FIFO에 한 개의 데이터를 넣는다.
      //
      // inputValid=1을 만들고 sampling edge를 한 번 통과시키면
      // push = inputValid && inputReady 가 발생한다.
      // ------------------------------------------------------------

      def push(data: Int): Unit = {

        dut.io.inputData  #= data
        dut.io.inputValid #= true

        dut.clockDomain.waitSampling()

        assert(
          dut.io.inputReady.toBoolean,
          f"FIFO must be ready when pushing 0x$data%02X"
        )
      }


      // ------------------------------------------------------------
      // 1. FIFO를 A/B/C/D로 채운다.
      // ------------------------------------------------------------

      println("[TEST] Fill FIFO")

      push(0x41)
      push(0x42)
      push(0x43)
      push(0x44)

      // 다음 sampling edge에서 inputValid를 false로 만들기 전에
      // 현재 FULL 상태를 관찰한다.
      //
      // 중요한 점:
      // count는 마지막 sampling edge에서 갱신된 register이고,
      // inputReady는 count에 의존하는 combinational signal이다.
      //
      // 따라서 여기서는 새로운 edge를 추가하지 않고
      // 현재 settled 상태를 관찰한다.

      assert(!dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x41)

      println(
        f"[FULL] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // 2. FIFO가 FULL일 때 E를 유지한다.
      // ------------------------------------------------------------

      println("[TEST] Hold E while FIFO is FULL")

      dut.io.inputData  #= 0x45
      dut.io.inputValid #= true

      // 현재 시점에서는 아직 sampling edge가 발생하지 않았으므로
      // E는 DUT 내부 FIFO에 들어가지 않는다.
      //
      // inputReady=false인 상태에서 valid=true를 유지하는 것이
      // 바로 valid/ready handshake의 의미이다.

      assert(dut.io.inputValid.toBoolean)
      assert(!dut.io.inputReady.toBoolean)

      println(
        f"[HOLD] valid=${dut.io.inputValid.toBoolean} " +
        f"ready=${dut.io.inputReady.toBoolean} " +
        f"data=0x${dut.io.inputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // 3. A를 POP한다.
      // ------------------------------------------------------------

      println("[TEST] Pop A")

      dut.io.outputReady #= true

      /*
       * 여기서 중요한 timing 분석:
       *
       * 현재:
       *
       *   count = 4
       *   inputReady = !full = false
       *   outputValid = true
       *   outputReady = true
       *   inputValid = true
       *
       * 다음 sampling edge에서:
       *
       *   pop  = outputValid && outputReady = true
       *   push = inputValid && inputReady  = false
       *
       * 따라서:
       *
       *   count : 4 -> 3
       *   readPtr : A -> B
       *
       * 그리고 edge 이후 combinational logic이 다시 계산되어
       *
       *   inputReady : false -> true
       *
       * 가 된다.
       *
       * 따라서 POP을 발생시키는 edge를 먼저 통과시킨 뒤
       * inputReady를 관찰해야 한다.
       */

      dut.clockDomain.waitSampling()

      // ------------------------------------------------------------
      // 4. while로 inputReady가 settlement될 때까지 기다린다.
      // ------------------------------------------------------------
      //
      // 여기서 while 조건 자체가 "관찰"이다.
      //
      // waitSampling() 직후에도 simulation event ordering 때문에
      // combinational 결과가 아직 관찰 가능한 상태가 아닐 수 있다.
      //
      // 그런 경우 while의 다음 iteration에서 다시 sampling edge를
      // 통과시키고 재관찰한다.
      //
      // 즉, 불필요하게 waitFallingEdge()를 무조건 삽입하지 않는다.
      //
      // 대신 timeout을 넣어 무한 대기를 방지한다.
      // ------------------------------------------------------------

      var timeout = 0
      val timeoutLimit = 20

      while (!dut.io.inputReady.toBoolean) {

        dut.clockDomain.waitSampling()

        timeout += 1

        assert(
          timeout < timeoutLimit,
          "Timeout: FIFO did not become ready after pop"
        )
      }

      println(
        f"[AFTER POP] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )

      assert(dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x42)


      // ------------------------------------------------------------
      // 5. 이제 E를 PUSH한다.
      // ------------------------------------------------------------

      println("[TEST] Transfer held E")

      /*
       * 현재:
       *
       *   inputValid = true
       *   inputData  = E
       *   inputReady = true
       *
       * 따라서 다음 sampling edge에서:
       *
       *   push = true
       *
       * 를 발생시킨다.
       *
       * outputReady는 이제 false로 만들어서
       * 이번 edge에서는 PUSH만 발생하도록 한다.
       */

      dut.io.outputReady #= false

      dut.clockDomain.waitSampling()

      // PUSH 이후에는 E가 FIFO에 들어갔으므로
      // FIFO는 다시 FULL 상태가 된다.
      //
      // inputValid는 여기서 더 이상 필요하지 않으므로 제거한다.

      dut.io.inputValid #= false

      assert(!dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x42)

      println(
        f"[AFTER E] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // 6. FIFO 순서 검증
      //
      // 현재 FIFO에는:
      //
      //   B -> C -> D -> E
      //
      // 가 들어 있어야 한다.
      // ------------------------------------------------------------

      def popAndCheck(expected: Int): Unit = {

        assert(dut.io.outputValid.toBoolean)

        val actual =
          dut.io.outputData.toInt

        println(
          f"[READ] expected=0x$expected%02X actual=0x$actual%02X"
        )

        assert(
          actual == expected,
          f"FIFO order error: expected 0x$expected%02X, got 0x$actual%02X"
        )

        dut.io.outputReady #= true

        dut.clockDomain.waitSampling()

        dut.io.outputReady #= false
      }


      popAndCheck(0x42) // B
      popAndCheck(0x43) // C
      popAndCheck(0x44) // D
      popAndCheck(0x45) // E


      // ------------------------------------------------------------
      // 7. 마지막 POP 이후 FIFO가 EMPTY인지 확인
      // ------------------------------------------------------------

      /*
       * 마지막 pop의 sampling edge에서
       *
       *   count : 1 -> 0
       *
       * 으로 변경된다.
       *
       * 따라서 그 edge 이후 empty가 true가 되어야 한다.
       *
       * 여기서도 무조건 falling edge를 추가하지 않고,
       * 현재 관찰 시점에서 상태를 확인한다.
       */

      assert(!dut.io.outputValid.toBoolean)

      println(
        f"[EMPTY] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )

      println("[PASS] ValidReadyFifo test passed.")
    }
}
```

# 그런데 여기서 한 가지 중요한 점

위 코드의 `push()`를 자세히 보면:

```scala
def push(data: Int): Unit = {

  dut.io.inputData  #= data
  dut.io.inputValid #= true

  dut.clockDomain.waitSampling()

  assert(dut.io.inputReady.toBoolean)
}
```

이 helper는 **현재 FIFO가 ready라는 것을 전제로 하는 push**입니다.

따라서 A/B/C/D를 넣을 때는 정확히 맞습니다.

하지만 FULL 상태에서 E를 들고 있는 상황은 다릅니다.

```text
inputValid = 1
inputReady = 0
```

이 상태에서는 **transfer가 일어나지 않습니다.**

즉,

```text
valid = 1
ready = 0
       ↓
      HOLD
```

이고,

A를 POP한 뒤:

```text
count:      4 → 3
                 ↓
inputReady: 0 → 1
                 ↓
valid=1, ready=1
                 ↓
다음 sampling edge
                 ↓
E transfer
```

라는 **두 개의 서로 다른 edge**가 존재합니다.

---

# 이번 코드에서 가장 중요한 `while`

제가 이번에 의도적으로 사용한 부분은 이것입니다.

```scala
dut.clockDomain.waitSampling()

var timeout = 0

while (!dut.io.inputReady.toBoolean) {

  dut.clockDomain.waitSampling()

  timeout += 1

  assert(
    timeout < timeoutLimit,
    "Timeout: FIFO did not become ready after pop"
  )
}
```

이것이 우리가 합의한 원칙에 정확히 대응합니다.

### ① 먼저 DUT 상태를 변화시키는 edge

```scala
dut.clockDomain.waitSampling()
```

여기서 POP이 발생합니다.

```text
count 4 → 3
```

### ② 그 결과를 관찰

```scala
while (!dut.io.inputReady.toBoolean)
```

여기서 `inputReady`를 관찰합니다.

### ③ 아직 settlement가 안 됐다면

```scala
dut.clockDomain.waitSampling()
```

하고 다시 관찰합니다.

즉:

```text
          sampling edge
               │
               ▼
        ┌──────────────┐
        │   POP 발생   │
        │ count 4 → 3  │
        └──────┬───────┘
               │
               ▼
      combinational settlement
               │
               ▼
      while condition 관찰
               │
        ┌──────┴──────┐
        │             │
      false          true
        │             │
        ▼             ▼
   다시 sampling    종료
```

이 방식의 좋은 점은 **`waitFallingEdge()`를 "혹시 모르니까" 넣지 않았다는 것**입니다.

---

# 그리고 timeout은 매우 중요합니다

예를 들어 RTL에 버그가 있어서:

```text
count = 4
      ↓ POP
count = 4   ← 버그
```

라면

```scala
while (!dut.io.inputReady.toBoolean)
```

는 영원히 끝나지 않을 수 있습니다.

그래서:

```scala
var timeout = 0

while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()

  timeout += 1

  assert(timeout < timeoutLimit)
}
```

으로 보호했습니다.

이것은 `while`을 나쁘게 사용하는 것이 아니라 오히려 **시뮬레이션에서 `while`을 안전하게 사용하는 좋은 패턴**이라고 볼 수 있습니다.

---

# 이번 테스트의 핵심 시나리오

전체 테스트는 결국 이것을 검증합니다.

```text
FIFO depth = 4

처음
   ↓
[A]
[A B]
[A B C]
[A B C D]
   ↓
FULL

inputValid = 1
inputData  = E
inputReady = 0

   ↓ POP A

[B C D]
   ↓
inputReady = 1

   ↓ PUSH E

[B C D E]
   ↓
FULL
```

그리고 마지막으로:

```text
B → C → D → E
```

순서가 유지되는지를 검증합니다.

---

## 특히 이번 예제에서 기억할 원칙

앞으로 SpinalSim 코드를 작성할 때 저는 다음 기준으로 작성하겠습니다.

> **1. 먼저 어느 edge에서 DUT state가 변경되는지 분석한다.**

> **2. 그 state 변경으로 어떤 combinational signal이 바뀌는지 분석한다.**

> **3. 그 결과를 언제 관찰해야 하는지 결정한다.**

> **4. 그 다음에 필요한 `waitSampling()`, `waitFallingEdge()` 등을 넣는다.**

> **5. 조건을 반복해서 기다리는 것이 자연스럽다면 `while`을 적극적으로 사용한다.**

> **6. `while`에는 timeout을 넣어 무한 대기를 방지한다.**

즉 **"wait를 많이 넣어서 안전하게 만든다"가 아니라 "DUT의 timing을 이해하고 필요한 wait만 넣는다"**가 앞으로의 기준입니다.

---

---

관찰 시점 타이밍 오류가 많다. 다시 점검해 보라

---

맞습니다. 다시 **event ordering 관점에서 한 줄씩 점검해 보니**, 제가 방금 작성한 코드에도 실제로 잘못된 관찰 시점이 있습니다.

특히 한 군데는 명백한 오류이고, 몇 군데는 제가 의도한 timing 설명과 실제 코드가 일치하지 않습니다.

이번에는 코드를 고치기 전에 **왜 잘못되었는지부터 정확히 짚겠습니다.**

---

# 1. 가장 명백한 오류: `push()`의 관찰 위치가 잘못되었다

제가 작성한 코드:

```scala
def push(data: Int): Unit = {

  dut.io.inputData  #= data
  dut.io.inputValid #= true

  dut.clockDomain.waitSampling()

  assert(
    dut.io.inputReady.toBoolean,
    f"FIFO must be ready when pushing 0x$data%02X"
  )
}
```

이것은 잘못됐습니다.

왜냐하면 `inputReady`는:

```scala
io.inputReady := !full
```

이고,

```scala
full = count === depth
```

입니다.

즉 **push를 발생시킨 sampling edge에서 `count`가 증가한 결과로 `inputReady`가 오히려 내려갈 수 있습니다.**

### D를 넣는 순간을 보면

FIFO depth = 4라고 합시다.

D를 넣기 직전:

```text
count      = 3
full       = false
inputReady = true
```

그래서:

```scala
inputValid = true
inputData  = D
```

다음 sampling edge에서:

```text
push = valid && ready
     = 1 && 1
     = 1
```

따라서 DUT는:

```text
count : 3 → 4
```

로 변경합니다.

그 결과 edge 이후:

```text
full       = true
inputReady = false
```

가 됩니다.

그런데 제가 작성한 코드는 **바로 이 edge 이후에**

```scala
assert(dut.io.inputReady.toBoolean)
```

를 했습니다.

즉 제가 실제로 확인한 것은:

> "D를 넣기 직전 FIFO가 ready였는가?"

가 아니라

> **"D를 넣은 이후에도 FIFO가 ready인가?"**

입니다.

당연히 `false`입니다.

따라서 이 코드는 D에서 실패합니다.

---

# 2. 이것이 바로 "행동 조건"과 "행동 후 상태"를 혼동한 것

`push()`에서 정말 확인하고 싶은 것은:

```text
push가 발생하기 직전
inputValid = 1
inputReady = 1
```

입니다.

그런데 저는 sampling edge를 통과한 뒤:

```text
inputReady
```

를 확인했습니다.

둘은 완전히 다른 관찰입니다.

정확히는:

```text
          PUSH 직전                    PUSH 직후

inputValid    1                           1
inputReady    1                           0   ← D일 경우
              │
              │ sampling edge
              ▼
count         3              →            4
```

따라서 `push()` helper는 **edge 전에 ready를 검사해야 합니다.**

---

# 3. 더 중요한 문제: 제가 `while`에 대해 설명한 부분도 부정확했다

제가 쓴:

```scala
dut.clockDomain.waitSampling()

while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

을 설명하면서,

> `waitSampling()` 이후 combinational settlement가 안 됐을 수도 있으니 while에서 다시 sampling한다

고 설명했습니다.

이 설명은 **너무 단순화되어 있고 부정확합니다.**

우리가 정말 구분해야 하는 것은:

### A. DUT state를 변경시키는 sampling edge

예:

```text
count 4 → 3
```

### B. 그 edge에 의해 발생한 combinational propagation

```text
count = 3
    ↓
full = false
    ↓
inputReady = true
```

### C. testbench가 그 값을 관찰하는 시점

```scala
dut.io.inputReady.toBoolean
```

입니다.

즉 우리가 반드시 질문해야 하는 것은:

> **`waitSampling()`이 반환되는 시점에 SpinalSim에서 해당 combinational 결과가 이미 관찰 가능한 상태인가?**

입니다.

이걸 확실히 확인하지 않고 "while에서 다시 waitSampling하면 된다"고 말한 것은 제가 너무 쉽게 처리했습니다.

---

# 4. `while` 자체는 문제가 없다

여기서 사용자께서 앞서 말씀하신 원칙이 정확합니다.

```scala
while (!dut.io.inputReady.toBoolean) {
    ...
}
```

자체는 아주 좋은 코드입니다.

문제는:

```scala
while (조건)
```

의 **조건을 언제 평가하느냐**입니다.

예를 들어 현재 simulation time에서:

```text
count = 3
inputReady = true
```

라면:

```scala
while (!dut.io.inputReady.toBoolean)
```

은 즉시 종료됩니다.

반대로 아직 DUT state가 바뀌기 전:

```text
count = 4
inputReady = false
```

를 관찰한다면 loop에 들어갑니다.

그러므로 중요한 것은:

```text
while을 쓰지 말자
```

가 아니라

```text
while 조건을 평가하기 전에
어떤 DUT event가 끝났어야 하는가?
```

입니다.

이 부분은 사용자께서 지적하신 원칙이 정확합니다.

---

# 5. `popAndCheck()`도 다시 살펴봐야 한다

제가 작성한:

```scala
def popAndCheck(expected: Int): Unit = {

  assert(dut.io.outputValid.toBoolean)

  val actual =
    dut.io.outputData.toInt

  assert(actual == expected)

  dut.io.outputReady #= true

  dut.clockDomain.waitSampling()

  dut.io.outputReady #= false
}
```

이것은 상대적으로 괜찮지만, **마지막 pop 이후의 관찰 시점이 함수 밖에 있다는 점**을 명확히 해야 합니다.

예를 들어 마지막 E를 읽으면:

```text
count = 1
```

에서 sampling edge:

```text
pop = 1

count : 1 → 0
```

그리고 그 결과:

```text
outputValid = !empty
            = false
```

가 됩니다.

따라서 마지막 `waitSampling()` 이후에:

```scala
assert(!dut.io.outputValid.toBoolean)
```

를 하는 것은 논리적으로 맞는 위치입니다.

다만 이것 역시 실제 SpinalSim event ordering을 확인하지 않고 무조건 "settled됐다"고 가정하면 안 됩니다.

---

# 6. 더 좋은 구조는 "행동 전 관찰"과 "행동 후 관찰"을 분리하는 것

이번 테스트에서는 이렇게 생각하는 것이 가장 명확합니다.

## PUSH

PUSH의 성립 조건:

```text
valid = 1
ready = 1
```

따라서 **PUSH를 발생시키기 전에** 이것을 관찰합니다.

```scala
assert(dut.io.inputReady.toBoolean)
```

그 다음:

```scala
waitSampling()
```

으로 PUSH를 발생시킵니다.

그리고 그 뒤에는:

> "PUSH 이후 FIFO 상태가 어떻게 되었는가?"

를 관찰합니다.

---

## POP

마찬가지입니다.

POP 조건:

```text
outputValid = 1
outputReady = 1
```

따라서 POP 전에:

```scala
assert(dut.io.outputValid.toBoolean)
```

그리고:

```scala
outputReady #= true
waitSampling()
```

그 다음 POP 결과를 관찰합니다.

---

# 7. 특히 FULL에서의 READ + held WRITE가 핵심이다

이번 테스트에서 가장 중요한 부분은:

```text
FIFO = A B C D
FULL

E가 들어오고 싶음

valid = 1
ready = 0
```

입니다.

이때:

```text
E는 아직 FIFO에 들어간 것이 아니다.
```

그런데 A를 POP하면:

```text
A B C D
 ↓ pop

B C D
```

가 됩니다.

그리고 **그 결과**:

```text
count 4 → 3
full  true → false
ready false → true
```

가 됩니다.

이제 E가 이미:

```text
valid = 1
data  = E
```

로 유지되고 있으므로:

```text
valid = 1
ready = 1
```

가 됩니다.

그리고 **다음 sampling edge에서 E가 들어갑니다.**

즉 정확한 edge 구조는:

```text
        EDGE #1                         EDGE #2
           │                               │
           ▼                               ▼

FULL       A B C D                         B C D E
           │                               │
           │ POP A                         │ PUSH E
           │                               │
count      4 → 3                           3 → 4
           │                               │
ready      0 → 1                           1 → 0
```

이것이 이번 simulation의 핵심입니다.

---

# 8. 그래서 `while`은 이렇게 사용하는 것이 더 자연스럽다

A를 POP한 뒤:

```scala
dut.io.outputReady #= true
dut.clockDomain.waitSampling()
```

여기까지가 **POP을 발생시키는 동작**입니다.

그 다음:

```scala
var timeout = 0

while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()

  timeout += 1
  assert(timeout < 20)
}
```

이것은 의미상:

> "POP이 완료된 후, FIFO가 다시 ready가 될 때까지 기다린다."

가 됩니다.

다만 여기서도 저는 **실제 SpinalSim 1.15.0의 event ordering을 기준으로 이 코드가 정말 필요한지 다시 검증한 후** 최종 simulation 코드를 제시하는 것이 맞다고 봅니다.

---

# 9. 이번에 제가 잘못 작성한 부분을 정리하면

| 코드 | 문제 |
|---|---|
| `push()` 후 `assert(inputReady)` | ❌ 명백한 timing/semantic 오류 |
| D push 후 ready를 검사 | ❌ D가 들어간 후 FIFO는 FULL이므로 ready=false |
| `while` 자체 | ✅ 문제 없음 |
| `while` 조건 관찰 시점 설명 | ⚠️ 제가 너무 단순하게 설명함 |
| POP 후 `inputReady` 관찰 | △ 실제 event ordering을 정확히 검증해야 함 |
| `popAndCheck()`의 기본 구조 | 대체로 맞음 |
| 무조건 `waitFallingEdge()` 추가 | ❌ 원칙적으로 맞지 않음 |

---

# 10. 따라서 지금은 코드를 다시 쓰는 것보다 먼저 이 원칙으로 재설계해야 한다

앞으로 `ValidReadyFifoSim`은 다음 순서로 작성하는 것이 맞습니다.

```text
① 현재 DUT 상태를 관찰
        ↓
② 다음 edge에서 발생할 handshake를 결정
        ↓
③ input/output 신호를 설정
        ↓
④ sampling edge를 발생
        ↓
⑤ edge 이후 DUT state 변화 분석
        ↓
⑥ 그 결과를 관찰
        ↓
⑦ 필요한 경우 while로 다음 조건까지 대기
```

특히 **"waitSampling()을 넣었으니까 이제 모든 신호가 안전하게 관찰 가능하다"라고 가정하지 않는 것**이 중요합니다.

그리고 이번에는 제가 실제로 코드 전체를 다시 작성하기 전에, **각 `waitSampling()` 앞뒤에서 `count`, `inputReady`, `outputValid`, `outputData`, `inputValid`, `outputReady`가 각각 어떤 값이어야 하는지를 edge-by-edge로 먼저 추적한 뒤** 코드를 만드는 것이 맞습니다.

방금 지적하신 대로, 이전 답변은 그 분석이 충분하지 않았습니다.

---

---

좋습니다. 이번에는 **코드를 먼저 쓰지 않고 edge-by-edge 상태를 완전히 추적한 뒤**, 그 결과를 그대로 simulation code로 옮기겠습니다.

그리고 중요한 결론부터 말씀드리면, 제가 앞서 제안했던

```scala
while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

을 **이번 위치에 그대로 사용하는 것도 위험합니다.**

왜냐하면 `inputValid = true`인 상태에서 이 `while`이 다음 sampling edge까지 진행하면, 그 edge에서 **E가 실제로 FIFO에 들어가 버릴 수 있기 때문**입니다.

이 부분이 이번 재검토에서 가장 중요한 발견입니다.

---

# 1. 우리가 검증할 RTL

대상 FIFO의 핵심은 다음입니다.

```scala
val empty = count === 0
val full  = count === depth

io.inputReady  := !full
io.outputValid := !empty
io.outputData  := mem(readPtr)

val push = io.inputValid && io.inputReady
val pop  = io.outputValid && io.outputReady
```

그리고:

```scala
when(push) {
    ...
}

when(pop) {
    ...
}

when(push && !pop) {
    count := count + 1
} elsewhen(!push && pop) {
    count := count - 1
}
```

이번에는 depth = 4입니다.

---

# 2. 먼저 초기 상태

초기:

```text
count = 0
writePtr = 0
readPtr  = 0
```

따라서:

```text
empty       = true
full        = false

inputReady  = true
outputValid = false
```

---

# 3. A를 PUSH

PUSH 직전:

```text
count       = 0
inputValid  = 1
inputReady  = 1
inputData   = A

push = 1
```

다음 sampling edge:

```text
count = 0 → 1
writePtr = 0 → 1
```

edge 이후:

```text
count       = 1
inputReady  = true
outputValid = true
outputData  = A
```

따라서 **A를 넣은 후 `inputReady == true`를 검사해도 됩니다.**

---

# 4. B PUSH

PUSH 직전:

```text
count      = 1
inputReady = true
inputData  = B
inputValid = true
```

sampling:

```text
count = 1 → 2
```

결과:

```text
count      = 2
inputReady = true
outputData = A
```

---

# 5. C PUSH

```text
count = 2
```

sampling:

```text
count = 2 → 3
```

결과:

```text
count      = 3
inputReady = true
outputData = A
```

---

# 6. D PUSH

이 부분이 앞서 작성한 코드의 오류가 발생한 곳입니다.

D PUSH **직전**:

```text
count      = 3
inputValid = 1
inputReady = 1
```

따라서:

```text
push = 1
```

sampling edge:

```text
count = 3 → 4
```

그리고 edge 이후:

```text
count      = 4
full       = true
inputReady = false
```

따라서 D를 넣은 뒤에는:

```scala
assert(!dut.io.inputReady.toBoolean)
```

가 맞습니다.

제가 이전 코드에서:

```scala
assert(dut.io.inputReady.toBoolean)
```

라고 한 것은 명백히 잘못됐습니다.

---

# 7. 이제 E를 HOLD

현재:

```text
FIFO = A B C D
count = 4
```

그리고:

```scala
dut.io.inputData  #= 0x45
dut.io.inputValid #= true
```

현재:

```text
inputValid = true
inputReady = false
```

따라서:

```text
push = false
```

입니다.

중요한 것은 **여기서는 sampling edge를 발생시키지 않는 것**입니다.

그냥 E를 들고 있습니다.

```text
valid = 1
ready = 0

       ↓

     HOLD E
```

---

# 8. 이제 A를 POP

이제:

```scala
dut.io.outputReady #= true
```

현재:

```text
outputValid = true
outputReady = true

pop = true
```

그리고:

```scala
dut.clockDomain.waitSampling()
```

이 sampling edge에서:

```text
pop  = true
push = false
```

왜 `push=false`인가?

현재 edge 직전에는:

```text
inputValid = true
inputReady = false
```

이기 때문입니다.

따라서:

```text
count = 4 → 3
readPtr = 0 → 1
```

그리고 edge 이후:

```text
FIFO logical contents = B C D
```

---

# 9. 이 시점의 관찰이 중요하다

sampling edge가 끝난 후 DUT 상태는:

```text
count       = 3
inputReady  = true
outputValid = true
outputData  = B
```

입니다.

그리고 여기서 **바로 관찰**합니다.

```scala
assert(dut.io.inputReady.toBoolean)
assert(dut.io.outputData.toInt == 0x42)
```

여기에는 `waitFallingEdge()`가 필요하지 않습니다.

왜냐하면 우리가 알고 싶은 것은 정확히:

> **A를 POP한 sampling edge의 결과**

이기 때문입니다.

---

# 10. 그런데 `while`을 여기 넣으면 안 된다

이것이 이번 재검토의 핵심입니다.

다음과 같이 하면:

```scala
while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
}
```

현재 edge 이후 `inputReady`가 이미 `true`라면 loop body는 **0회 실행**됩니다.

괜찮습니다.

그런데 만약 simulation event ordering 때문에 현재 관찰 시점에서 아직 `inputReady`가 false로 보인다고 가정해 봅시다.

그러면:

```scala
while (!inputReady) {
    waitSampling()
}
```

가 다음 sampling edge로 갑니다.

그런데 현재:

```text
inputValid = true
inputReady = true
```

입니다.

따라서 다음 edge에서:

```text
push = true
```

가 되어 E가 FIFO에 들어갑니다.

결과:

```text
count = 3 → 4
inputReady = true → false
```

가 됩니다.

즉 `while`을 빠져나올 때 오히려:

```text
inputReady = false
```

가 될 수도 있습니다.

**따라서 이 경우 `while`은 잘못된 synchronization 방법입니다.**

이것은 상당히 중요한 timing 문제입니다.

---

# 11. 그러면 `while`은 어디에 사용해야 하는가?

`while`은 **조건이 만족될 때까지 sampling edge를 계속 진행해도 DUT transaction을 의도치 않게 발생시키지 않는 경우**에 사용해야 합니다.

예를 들어 FIFO가 비어 있고:

```text
outputValid = false
```

인데 output 쪽에서 데이터를 기다리는 상황이라면:

```scala
while (!dut.io.outputValid.toBoolean) {
    dut.clockDomain.waitSampling()
}
```

같은 구조가 자연스럽습니다.

왜냐하면 `outputReady`를 false로 유지하고 있다면:

```text
outputReady = false
```

이므로 기다리는 동안 `pop`이 발생하지 않습니다.

반면 지금 E의 경우는:

```text
inputValid = true
```

로 이미 transaction 의도가 걸려 있으므로, `waitSampling()`을 추가하는 순간 transaction이 발생할 수 있습니다.

---

# 12. 따라서 이번 테스트의 올바른 전략

이번 테스트는 `while`을 억지로 넣을 필요가 없습니다.

정확히 하나의 edge를 알고 있습니다.

```text
POP A
  ↓
sampling edge
  ↓
count 4 → 3
  ↓
inputReady = true
```

그러므로:

```scala
dut.io.outputReady #= true
dut.clockDomain.waitSampling()

assert(dut.io.inputReady.toBoolean)
```

이것이 더 정확합니다.

**필요 없는 synchronization을 넣지 않는 것**이 이번 원칙에 맞습니다.

---

# 13. 이제 E를 PUSH

A가 빠진 직후:

```text
FIFO = B C D

inputValid = true
inputData  = E
inputReady = true
```

이므로 다음 edge에서:

```text
push = true
```

가 됩니다.

여기서 중요한 점:

```scala
dut.io.outputReady #= false
```

를 먼저 해야 합니다.

왜냐하면 이번에는 POP을 하지 않고 **E만 PUSH**하려고 하기 때문입니다.

따라서:

```scala
dut.io.outputReady #= false
dut.clockDomain.waitSampling()
```

이 edge:

```text
push = true
pop  = false
```

결과:

```text
count = 3 → 4
```

그리고:

```text
FIFO = B C D E
inputReady = false
```

---

# 14. 그 후 `inputValid`를 내린다

이제 E는 이미 들어갔습니다.

따라서:

```scala
dut.io.inputValid #= false
```

로 내려도 됩니다.

여기서는 **sampling edge를 또 기다릴 필요가 없습니다.**

왜냐하면 `inputValid`는 다음 transaction을 위한 입력이고, 현재 transaction은 이미 직전 sampling edge에서 끝났기 때문입니다.

---

# 15. READ 쪽도 동일하게 분석

현재:

```text
FIFO = B C D E
```

입니다.

POP B를 하기 직전:

```text
outputValid = true
outputData  = B
outputReady = false
```

먼저 데이터를 관찰합니다.

```scala
assert(dut.io.outputValid.toBoolean)
assert(dut.io.outputData.toInt == 0x42)
```

그 다음:

```scala
dut.io.outputReady #= true
dut.clockDomain.waitSampling()
```

이 edge에서:

```text
pop = true
```

그리고:

```text
count = 4 → 3
```

그 직후:

```scala
dut.io.outputReady #= false
```

다음 READ도 동일합니다.

---

# 16. 그러면 최종 `ValidReadyFifoSim`은 이렇게 작성하는 것이 훨씬 정확하다

```scala
import spinal.core._
import spinal.core.sim._

object ValidReadyFifoSim extends App {

  val depth = 4

  SimConfig
    .withWave
    .doSim(ValidReadyFifo(8, depth)) { dut =>

      dut.clockDomain.forkStimulus(10)

      // ------------------------------------------------------------
      // Initial
      // ------------------------------------------------------------

      dut.io.inputValid  #= false
      dut.io.inputData   #= 0
      dut.io.outputReady #= false

      dut.clockDomain.waitSampling(2)

      assert(dut.io.inputReady.toBoolean)
      assert(!dut.io.outputValid.toBoolean)

      println(
        f"[Initial] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // PUSH A
      //
      // 관찰: sampling edge 이전의 inputReady
      // 행동: 다음 sampling edge에서 PUSH
      // ------------------------------------------------------------

      dut.io.inputData  #= 0x41
      dut.io.inputValid #= true

      assert(dut.io.inputReady.toBoolean)

      dut.clockDomain.waitSampling()


      // ------------------------------------------------------------
      // PUSH B
      // ------------------------------------------------------------

      dut.io.inputData #= 0x42

      assert(dut.io.inputReady.toBoolean)

      dut.clockDomain.waitSampling()


      // ------------------------------------------------------------
      // PUSH C
      // ------------------------------------------------------------

      dut.io.inputData #= 0x43

      assert(dut.io.inputReady.toBoolean)

      dut.clockDomain.waitSampling()


      // ------------------------------------------------------------
      // PUSH D
      //
      // 이 edge 이후 count = 4가 되므로
      // inputReady는 false가 된다.
      // ------------------------------------------------------------

      dut.io.inputData #= 0x44

      assert(dut.io.inputReady.toBoolean)

      dut.clockDomain.waitSampling()

      assert(!dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x41)

      println(
        f"[FULL] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // HOLD E
      //
      // E는 아직 FIFO에 들어가지 않는다.
      //
      // inputValid = 1
      // inputReady = 0
      //
      // 따라서 push = 0
      // ------------------------------------------------------------

      dut.io.inputData  #= 0x45
      dut.io.inputValid #= true

      assert(dut.io.inputValid.toBoolean)
      assert(!dut.io.inputReady.toBoolean)

      println(
        f"[HOLD E] valid=${dut.io.inputValid.toBoolean} " +
        f"ready=${dut.io.inputReady.toBoolean} " +
        f"data=0x${dut.io.inputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // POP A
      //
      // 현재:
      //
      //   outputValid = 1
      //   outputReady = 1
      //
      // 따라서 이 sampling edge에서 POP 발생.
      //
      // 동시에:
      //
      //   inputValid = 1
      //   inputReady  = 0
      //
      // 이므로 E의 PUSH는 발생하지 않는다.
      // ------------------------------------------------------------

      dut.io.outputReady #= true

      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x41)

      dut.clockDomain.waitSampling()

      dut.io.outputReady #= false


      // ------------------------------------------------------------
      // POP 결과 관찰
      //
      // 바로 이전 sampling edge에서:
      //
      //   count : 4 -> 3
      //   readPtr: A -> B
      //
      // 따라서 이제:
      //
      //   inputReady = true
      //   outputData = B
      // ------------------------------------------------------------

      assert(dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x42)

      println(
        f"[AFTER POP A] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // PUSH E
      //
      // 현재:
      //
      //   inputValid = 1
      //   inputReady = 1
      //
      // 따라서 다음 sampling edge에서 E가 PUSH된다.
      // ------------------------------------------------------------

      assert(dut.io.inputReady.toBoolean)

      dut.clockDomain.waitSampling()

      dut.io.inputValid #= false

      // PUSH E 이후:
      //
      //   count = 4
      //   FIFO = B C D E
      //   inputReady = false

      assert(!dut.io.inputReady.toBoolean)
      assert(dut.io.outputValid.toBoolean)
      assert(dut.io.outputData.toInt == 0x42)

      println(
        f"[AFTER PUSH E] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )


      // ------------------------------------------------------------
      // FIFO 순서 검증
      //
      // 현재 FIFO:
      //
      //   B -> C -> D -> E
      // ------------------------------------------------------------

      def popAndCheck(expected: Int): Unit = {

        assert(dut.io.outputValid.toBoolean)

        val actual = dut.io.outputData.toInt

        println(
          f"[READ] expected=0x$expected%02X " +
          f"actual=0x$actual%02X"
        )

        assert(
          actual == expected,
          f"FIFO order error: expected=0x$expected%02X " +
          f"actual=0x$actual%02X"
        )

        dut.io.outputReady #= true

        dut.clockDomain.waitSampling()

        dut.io.outputReady #= false
      }


      popAndCheck(0x42) // B
      popAndCheck(0x43) // C
      popAndCheck(0x44) // D
      popAndCheck(0x45) // E


      // ------------------------------------------------------------
      // 마지막 POP 결과 관찰
      //
      // 마지막 sampling edge:
      //
      //   count : 1 -> 0
      //
      // 따라서:
      //
      //   outputValid = false
      // ------------------------------------------------------------

      assert(!dut.io.outputValid.toBoolean)
      assert(dut.io.inputReady.toBoolean)

      println(
        f"[EMPTY] ready=${dut.io.inputReady.toBoolean} " +
        f"valid=${dut.io.outputValid.toBoolean} " +
        f"data=0x${dut.io.outputData.toInt}%02X"
      )

      println("[PASS] ValidReadyFifo test passed.")
    }
}
```

---

# 17. 이번 코드에서 의도적으로 `while`을 제거한 이유

이 부분이 중요합니다.

사용자께서 처음 말씀하신 원칙:

> "`while`은 좋은 도구다."

이것에는 저도 동의합니다.

하지만 그것이:

> "`while`을 반드시 넣어야 한다."

는 뜻은 아닙니다.

이번 테스트에서는 우리가 정확하게 알고 있습니다.

```text
POP A
 ↓
딱 하나의 sampling edge
 ↓
count 4 → 3
 ↓
inputReady = true
```

따라서:

```scala
dut.clockDomain.waitSampling()

assert(dut.io.inputReady.toBoolean)
```

가 가장 직접적인 표현입니다.

여기에:

```scala
while (!dut.io.inputReady.toBoolean) {
    dut.clockDomain.waitSampling()
}
```

를 넣으면 오히려 **E가 unintended push되는 새로운 timing bug를 만들 가능성**이 있습니다.

---

# 18. 이번 수정에서 가장 중요한 원칙

이번에는 `waitSampling()`의 개수를 최소화하는 것 자체가 목적이 아닙니다.

다음 질문을 먼저 했습니다.

### PUSH 직전

```text
inputValid = ?
inputReady = ?
```

### sampling edge

```text
push = valid && ready
```

### edge 이후

```text
count = ?
inputReady = ?
outputValid = ?
outputData = ?
```

이렇게 분석했습니다.

그 결과:

```text
PUSH:
  ready를 edge 전에 검사

POP:
  valid/data를 edge 전에 검사

POP 결과:
  waitSampling() 직후 결과 검사

PUSH 결과:
  waitSampling() 직후 결과 검사
```

라는 구조가 나왔습니다.

---

# 19. 그리고 `while`을 사용할 때의 더 정확한 기준

앞으로는 이렇게 구분하겠습니다.

### 좋은 `while`

```scala
while (!dut.io.outputValid.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

**조건이 만족될 때까지 기다리는 동안 transaction이 발생하지 않는 경우.**

예:

```text
outputReady = false
```

이므로 기다리는 동안 POP이 발생하지 않습니다.

---

### 위험한 `while`

```scala
while (!dut.io.inputReady.toBoolean) {
  dut.clockDomain.waitSampling()
}
```

인데 동시에:

```text
inputValid = true
```

인 경우.

왜냐하면 loop 내부의 `waitSampling()`이 다음 edge를 발생시키면서:

```text
inputValid = 1
inputReady = 1
```

가 되는 순간 실제 PUSH가 발생할 수 있기 때문입니다.

즉:

> **`while` 조건뿐만 아니라 loop body의 `waitSampling()`이 진행시키는 다음 edge에서 어떤 transaction이 발생하는지도 반드시 분석해야 합니다.**

이것이 이번에 발견한 가장 중요한 timing 포인트입니다.

---

## 결론

이번에는 이전 코드의 문제를 단순히 `waitFallingEdge()`를 하나 더 넣어서 해결하지 않았습니다.

오히려:

```text
D PUSH
 ↓
FULL

E valid 유지
 ↓
POP A
 ↓
sampling edge
 ↓
count 4 → 3
 ↓
inputReady = true
 ↓
즉시 관찰
 ↓
다음 sampling edge
 ↓
E PUSH
```

라는 **정확한 두 edge의 transaction sequence**를 기준으로 코드를 다시 작성했습니다.

그리고 이 분석을 적용하면, 이번 테스트에서는 **`while`을 억지로 사용할 필요가 없다는 결론**도 자연스럽게 나옵니다.

이제 이 버전을 실제로 돌려보면, 출력 결과와 `wave.vcd`를 기준으로 **각 sampling edge에서 `push/pop/count/inputReady/outputValid`가 우리가 계산한 것과 정확히 일치하는지**까지 같이 검증하는 것이 다음 단계입니다.