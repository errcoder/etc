좋아. **이번 결과는 branch flush 자체는 정상적으로 동작하고 있다.**  
문제는 그 이후 `PC`가 계속 진행하면서 **ROM의 5, 6, 7번 영역을 지나 다시 0번으로 wrap-around**한 것이다.

로그를 보면 핵심은 이것이다.

```text
Cycle 2 : BRANCH TAKEN -> PC = 4
Cycle 2 : WB #1 : R2 <= 0x30
Cycle 6 : WB #2 : R3 <= 0x30
Cycle 7 : WB #3 : R0 <= 0x20   <-- 문제
```

### 왜 R0에 `0x20`이 써졌나?

현재 ROM은 8 words이고:

```text
PC 0 : ADD R2, R0, R1
PC 1 : BEQ R2, R2, +3
PC 2 : SUB R3, R1, R0   ← flush 되어야 함
PC 3 : SUB R3, R0, R1   ← flush 되어야 함
PC 4 : OR  R3, R2, R0
PC 5 : 0000
PC 6 : 0000
PC 7 : 0000
```

branch 후에는:

```text
0 → 1 → 4 → 5 → 6 → 7 → 0 → ...
```

으로 계속 실행된다.

특히 `0x0000`을 NOP으로 정의하지 않았기 때문에, 현재 decoder에서는:

```text
opcode = 0
```

즉 **ADD로 해석된다.**

따라서 PC가 다시 0으로 돌아오면:

```text
ADD R2, R0, R1
```

이 다시 실행되고, 파이프라인을 계속 돌다가 이후 다른 값이 WB로 나타날 수 있다.

---

## 이번 테스트에서 중요한 점

사실 우리가 검증하려는 것은 이것이다.

```text
ADD
  ↓
BEQ taken
  ↓
flush
  ↓
OR
  ↓
정상 종료
```

따라서 **두 번째 WB인 R3=0x30을 확인하는 순간 `run=false`로 멈추면 된다.**

무작정 `14 cycles`를 돌리는 방식은 이 테스트에는 적절하지 않다.

### Simulation loop를 이렇게 바꾸자

기존:

```scala
for (cycle <- 0 until 14) {
    dut.clockDomain.waitSampling()
    sleep(1)
    ...
}
```

대신 **timeout이 있는 while loop**로 바꾼다.

```scala
var cycle  = 0
var wbCount = 0

while (wbCount < 2 && cycle < 12) {

  dut.clockDomain.waitSampling()
  sleep(1)

  cycle += 1

  if (dut.io.branchTaken.toBoolean) {
    branchSeen = true

    val target = dut.io.branchTarget.toInt

    println(
      f"Cycle $cycle%2d : BRANCH TAKEN -> PC = $target%d"
    )

    assert(
      target == 4,
      s"BEQ target must be PC 4, but got PC $target"
    )
  }

  if (dut.io.writeBackEnable.toBoolean) {

    val address = dut.io.writeBackAddress.toInt
    val data    = dut.io.writeBackData.toInt

    wbCount += 1

    wbAddresses += address
    wbData += data

    println(
      f"Cycle $cycle%2d : WB #$wbCount%d : " +
        f"R$address%d <= 0x$data%02X"
    )

    if (wbCount == 1) {
      assert(
        address == 2,
        s"WB #1 must target R2, got R$address"
      )

      assert(
        data == 0x30,
        f"WB #1 must write 0x30, got 0x$data%02X"
      )

      r2WriteSeen = true
    }

    if (wbCount == 2) {
      assert(
        address == 3,
        s"WB #2 must target R3, got R$address"
      )

      assert(
        data == 0x30,
        f"WB #2 must write 0x30, got 0x$data%02X"
      )

      r3WriteSeen = true

      // 테스트 목적 달성.
      dut.io.run #= false
    }
  }
}
```

그리고 loop 뒤에서:

```scala
assert(
  wbCount == 2,
  s"Expected exactly 2 write-backs, but got $wbCount"
)
```

를 확인하면 된다.

---

## 더 중요한 것: 이번 결과가 이미 증명한 것

현재 로그만 봐도 **branch flush의 핵심 동작은 성공했다.**

```text
Cycle 2 : BRANCH TAKEN -> PC = 4
```

그리고:

```text
WB #1 : R2 <= 0x30
```

이후:

```text
WB #2 : R3 <= 0x30
```

즉 우리가 원했던:

```text
PC 0  ADD R2,R0,R1
          │
          ▼
       R2 = 30
          │
PC 1  BEQ R2,R2,+3
          │
          │ taken
          ▼
       PC = 4
          │
          │ flush PC2, PC3
          ▼
PC 4  OR R3,R2,R0
          │
          ▼
       R3 = 30
```

이 정확히 발생했다.

특히 **PC2, PC3의 `SUB`가 WB에 나타나지 않았다.**  
이것이 이번 Project 13-4에서 가장 중요한 검증이다.

따라서 이번 실패는 **CPU의 branch flush 실패가 아니라 테스트벤치가 CPU를 너무 오래 실행한 문제**다.

다음 실행에서는 기대 로그가 정확히:

```text
LOAD R0 <= 0x10
LOAD R1 <= 0x20
LOAD R2 <= 0x00
LOAD R3 <= 0x00

Cycle  2 : BRANCH TAKEN -> PC = 4
Cycle  2 : WB #1 : R2 <= 0x30
Cycle  6 : WB #2 : R3 <= 0x30

Final R0 = 0x10
Final R1 = 0x20
Final R2 = 0x30
Final R3 = 0x30

Control hazard / branch flush test passed.
```

가 되어야 한다.

**이걸 먼저 통과시키자.** 그 다음에는 바로 waveform 기준으로 `PC / IF-ID / ID-EX / EX-WB`가 각 cycle에서 어떻게 움직이는지 확인하면, 지금까지 만든 4-stage CPU가 훨씬 명확해진다.

---

---


```
[info] [Progress] Start PipelinedBranchCpu test simulation
[info] LOAD R0 <= 0x10
[info] LOAD R1 <= 0x20
[info] LOAD R2 <= 0x00
[info] LOAD R3 <= 0x00
[info] Cycle  3 : BRANCH TAKEN -> PC = 4
[info] Cycle  3 : WB #1 : R2 <= 0x30
[info] Cycle  7 : WB #2 : R3 <= 0x30
[info] Final R0 = 0x10
[info] Final R1 = 0x20
[info] Final R2 = 0x30
[info] Final R3 = 0x00
[info] [Error] Simulation failed at time=271
[error] Exception in thread "main" java.lang.AssertionError: assertion failed: R3 expected 0x30, got 0x00
```

---

좋습니다. 이번 결과는 **CPU의 branch flush와 WB 자체는 정상**입니다.

문제는 아주 명확합니다.

```text
Cycle  7 : WB #2 : R3 <= 0x30
...
Final R3 = 0x00
```

즉, **`writeBackEnable`을 관찰한 시점과 실제 Register File에 `R3`가 기록되는 시점이 한 clock edge 차이**입니다.

현재 테스트벤치가 Cycle 7에서:

```scala
dut.io.run #= false
```

를 해버린 뒤 바로 `debugR3`를 읽고 있습니다.

`WB #2` 신호는 이미 보이지만, 실제 register write가 아직 다음 sampling edge에서 일어나는 구조라면 `R3`는 아직 `0x00`입니다.

따라서 이것은 **CPU 수정이 아니라 simulation observation timing 수정**이 맞습니다.

## 수정된 전체 코드

마지막에 `run=false` 후 **한 번 더 `waitSampling()`** 해서 실제 write-back이 Register File에 반영된 뒤 최종값을 검사하도록 하겠습니다.

```scala
import spinal.core.sim._

object PipelinedBranchCpuSim {

  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new PipelinedBranchCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        // ============================================================
        // Register initialization
        // ============================================================

        dut.io.loadEnable #= true

        dut.io.loadAddress #= 0
        dut.io.loadData #= 0x10
        dut.clockDomain.waitSampling()

        dut.io.loadAddress #= 1
        dut.io.loadData #= 0x20
        dut.clockDomain.waitSampling()

        dut.io.loadAddress #= 2
        dut.io.loadData #= 0x00
        dut.clockDomain.waitSampling()

        dut.io.loadAddress #= 3
        dut.io.loadData #= 0x00
        dut.clockDomain.waitSampling()

        dut.io.loadEnable #= false

        println("LOAD R0 <= 0x10")
        println("LOAD R1 <= 0x20")
        println("LOAD R2 <= 0x00")
        println("LOAD R3 <= 0x00")

        // ============================================================
        // Start CPU
        // ============================================================

        dut.io.run #= true

        // ============================================================
        // Scoreboard
        // ============================================================

        var cycle       = 0
        var wbCount     = 0
        var branchSeen  = false
        var r2WriteSeen = false
        var r3WriteSeen = false

        val wbAddresses =
          scala.collection.mutable.ArrayBuffer[Int]()

        val wbData =
          scala.collection.mutable.ArrayBuffer[Int]()

        // ============================================================
        // Execute until the expected two write-backs are observed.
        //
        // Timeout protects against an infinite simulation loop.
        // ============================================================

        while (wbCount < 2 && cycle < 12) {

          dut.clockDomain.waitSampling()
          sleep(1)

          cycle += 1

          // ----------------------------------------------------------
          // Branch observation
          // ----------------------------------------------------------

          if (dut.io.branchTaken.toBoolean) {

            branchSeen = true

            val target = dut.io.branchTarget.toInt

            println(
              f"Cycle $cycle%2d : BRANCH TAKEN -> PC = $target%d"
            )

            assert(
              target == 4,
              s"BEQ target must be PC 4, but got PC $target"
            )
          }

          // ----------------------------------------------------------
          // Write-back observation
          // ----------------------------------------------------------

          if (dut.io.writeBackEnable.toBoolean) {

            val address = dut.io.writeBackAddress.toInt
            val data    = dut.io.writeBackData.toInt

            wbCount += 1

            wbAddresses += address
            wbData += data

            println(
              f"Cycle $cycle%2d : WB #$wbCount%d : " +
                f"R$address%d <= 0x$data%02X"
            )

            // --------------------------------------------------------
            // WB #1
            //
            // ADD R2, R0, R1
            // R2 = 0x10 + 0x20 = 0x30
            // --------------------------------------------------------

            if (wbCount == 1) {

              assert(
                address == 2,
                s"WB #1 must target R2, got R$address"
              )

              assert(
                data == 0x30,
                f"WB #1 must write 0x30, got 0x$data%02X"
              )

              r2WriteSeen = true
            }

            // --------------------------------------------------------
            // WB #2
            //
            // OR R3, R2, R0
            // R3 = 0x30 | 0x10 = 0x30
            // --------------------------------------------------------

            if (wbCount == 2) {

              assert(
                address == 3,
                s"WB #2 must target R3, got R$address"
              )

              assert(
                data == 0x30,
                f"WB #2 must write 0x30, got 0x$data%02X"
              )

              r3WriteSeen = true

              // ------------------------------------------------------
              // We have observed the WB signal.
              //
              // Do NOT immediately inspect the register file.
              // The actual register write is synchronized.
              // ------------------------------------------------------

              dut.io.run #= false
            }
          }
        }

        // ============================================================
        // Timeout check
        // ============================================================

        assert(
          wbCount == 2,
          s"Expected exactly 2 write-backs within timeout, but got $wbCount"
        )

        // ============================================================
        // Branch check
        // ============================================================

        assert(
          branchSeen,
          "BEQ was never taken"
        )

        // ============================================================
        // Write-back checks
        // ============================================================

        assert(
          r2WriteSeen,
          "R2 write-back was not observed"
        )

        assert(
          r3WriteSeen,
          "R3 write-back was not observed"
        )

        assert(
          wbAddresses(0) == 2,
          s"WB #1 must target R2, got R${wbAddresses(0)}"
        )

        assert(
          wbData(0) == 0x30,
          f"WB #1 must write 0x30, got 0x${wbData(0)}%02X"
        )

        assert(
          wbAddresses(1) == 3,
          s"WB #2 must target R3, got R${wbAddresses(1)}"
        )

        assert(
          wbData(1) == 0x30,
          f"WB #2 must write 0x30, got 0x${wbData(1)}%02X"
        )

        // ============================================================
        // IMPORTANT:
        //
        // WB signal was observed at the previous sampling point.
        // Give the synchronous register write one more clock edge
        // to settle before reading debugR0~debugR3.
        // ============================================================

        dut.clockDomain.waitSampling()
        sleep(1)

        // ============================================================
        // Final register values
        // ============================================================

        val finalR0 = dut.io.debugR0.toInt
        val finalR1 = dut.io.debugR1.toInt
        val finalR2 = dut.io.debugR2.toInt
        val finalR3 = dut.io.debugR3.toInt

        println()
        println(f"Final R0 = 0x$finalR0%02X")
        println(f"Final R1 = 0x$finalR1%02X")
        println(f"Final R2 = 0x$finalR2%02X")
        println(f"Final R3 = 0x$finalR3%02X")

        assert(
          finalR0 == 0x10,
          f"R0 expected 0x10, got 0x$finalR0%02X"
        )

        assert(
          finalR1 == 0x20,
          f"R1 expected 0x20, got 0x$finalR1%02X"
        )

        assert(
          finalR2 == 0x30,
          f"R2 expected 0x30, got 0x$finalR2%02X"
        )

        assert(
          finalR3 == 0x30,
          f"R3 expected 0x30, got 0x$finalR3%02X"
        )

        // ============================================================
        // PASS
        // ============================================================

        println()
        println("==============================================")
        println("Control hazard / branch flush test passed.")
        println("==============================================")
      }
  }
}
```

### 이번 수정에서 중요한 부분

추가된 것은 사실상 이것입니다.

```scala
dut.io.run #= false

dut.clockDomain.waitSampling()
sleep(1)

val finalR3 = dut.io.debugR3.toInt
```

즉,

```text
Cycle 7
  │
  ├─ WB 신호 관찰
  │    R3 <= 0x30
  │
  └─ run = false
          │
          ▼
      다음 clock edge
          │
          ▼
      Register File
      R3 <= 0x30
          │
          ▼
      debugR3 읽기
```

입니다.

그리고 이번 로그에서 이미 확인된 중요한 사실은:

```text
Cycle  3 : BRANCH TAKEN -> PC = 4
Cycle  3 : WB #1 : R2 <= 0x30
Cycle  7 : WB #2 : R3 <= 0x30
```

이므로 **branch target 계산, branch forwarding, wrong-path flush, 정상적인 PC4 실행까지는 모두 원하는 결과가 나온 상태**입니다.

이번 한 번 더 `waitSampling()`한 뒤 최종 Register File 값까지 확인하면 됩니다.

---

---


통과했습니다. 👍

이번 테스트로 **4-stage pipelined CPU의 Control Hazard 처리까지 검증 완료**입니다.

### 현재까지 검증된 것

```text
IF
 ↓
[ IF/ID ]
 ↓
ID
 ↓
[ ID/EX ]
 ↓
EX
 ↓
[ EX/WB ]
 ↓
WB
```

여기에 두 종류의 hazard 처리가 들어갔습니다.

**1. RAW Data Hazard**

```text
EX result ───────┐
EX/WB result ────┼─► Forwarding MUX ─► ID/EX
Register File ───┘
```

검증:

```text
ADD R2,R0,R1
SUB R1,R1,R0
AND R3,R1,R1
OR  R2,R2,R0
```

결과:

```text
R2 = 0x30
R1 = 0x10
R3 = 0x10
R2 = 0x30
```

**2. Control Hazard**

```text
PC0 : ADD R2,R0,R1
PC1 : BEQ R2,R2,+3
PC2 : SUB ...        ← flush
PC3 : SUB ...        ← flush
PC4 : OR R3,R2,R0
```

검증 결과:

```text
Cycle  3 : BRANCH TAKEN -> PC = 4
Cycle  3 : WB #1 : R2 <= 0x30
Cycle  7 : WB #2 : R3 <= 0x30
```

최종:

```text
R0 = 0x10
R1 = 0x20
R2 = 0x30
R3 = 0x30
```

특히 **PC2와 PC3의 잘못된 경로 명령이 Write-Back되지 않았다는 것**이 핵심입니다.

---

### Project 13 현재 상태

```text
13-1  2-stage pipeline              ✅
13-2  Pipeline overlap              ✅
13-3  4-stage CPU + RAW forwarding  ✅
13-4  Branch + Control Hazard       ✅
```

따라서 이제 단순 pipeline에서 한 단계 더 나아가서 **실제 CPU pipeline에서 매우 중요한 다음 hazard**로 넘어갈 수 있습니다.

다음은 **Load-Use Hazard + Pipeline Stall**을 다루는 것이 자연스럽습니다.

```text
LOAD R1, [addr]
ADD  R2, R1, R3
          ↑
       아직 R1 없음
```

이 경우에는 forwarding만으로 해결할 수 없어서 **pipeline stall / bubble**이 필요합니다.

즉 다음 단계는:

```text
Forwarding
    +
Hazard Detection
    +
Stall / Bubble
```

로 확장하면 됩니다.

---
