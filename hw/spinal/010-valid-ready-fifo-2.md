Yes — this failure is from the **testbench timing**, not from the `ValidReadyFifo` RTL.

And this is exactly the kind of SpinalSim scheduling issue we want to catch carefully.

# 1. Why it failed

The important part of the test was:

```scala
dut.io.outputReady #= true

dut.clockDomain.waitSampling()

dut.io.outputReady #= false

assert(
  dut.io.inputReady.toBoolean,
  "FIFO should become ready after one pop"
)
```

At the sampling edge:

```text
Before edge:

count = 4
full  = true
ready = false

outputReady = true
outputValid = true

        ↓ sampling edge

pop = outputValid && outputReady
    = true
```

So the FIFO performs:

```text
count: 4 → 3
```

But `count` is a register. The resulting:

```text
inputReady = !full
```

is only updated **after the sequential state update and combinational reevaluation**.

Your test checks `inputReady` too soon.

---

# 2. The important distinction

This:

```scala
dut.clockDomain.waitSampling()
```

means roughly:

> Wait until the next sampling edge occurs.

It does **not** mean:

> Wait until every signal affected by that edge has completely propagated and settled, then check every combinational output.

For this particular test, we should give the simulator another event boundary before checking the new combinational state.

A simple solution is:

```scala
dut.clockDomain.waitFallingEdge()
```

after the pop.

---

# 3. Corrected section

Replace this:

```scala
dut.io.outputReady #= true

dut.clockDomain.waitSampling()

dut.io.outputReady #= false

assert(
  dut.io.inputValid.toBoolean,
  "E must still be valid"
)

assert(
  dut.io.inputReady.toBoolean,
  "FIFO should become ready after one pop"
)
```

with:

```scala
dut.io.outputReady #= true

// Pop A at the next active sampling edge.
dut.clockDomain.waitSampling()

// Release outputReady after the pop edge.
dut.io.outputReady #= false

// Give the registered count update and dependent
// combinational signals time to settle.
dut.clockDomain.waitFallingEdge()

assert(
  dut.io.inputValid.toBoolean,
  "E must still be valid"
)

assert(
  dut.io.inputReady.toBoolean,
  "FIFO should become ready after one pop"
)
```

That is the key fix.

---

# 4. But there is an even more important point

At the edge where A is popped:

```text
Before edge:

FIFO:
A B C D
FULL

E:
valid = 1
ready = 0
```

Therefore:

```text
push = valid && ready
     = 1 && 0
     = 0
```

and:

```text
pop = outputValid && outputReady
    = 1 && 1
    = 1
```

So **E does NOT enter the FIFO at the same edge that A leaves it.**

After the edge:

```text
FIFO:

B
C
D

count = 3
full  = false
inputReady = true
```

Meanwhile E is still being held:

```text
E:
valid = 1
ready = 1
```

Therefore, at the **next** sampling edge:

```text
push = 1 && 1
     = 1
```

and E enters.

So the sequence is:

```text
Cycle N
────────────────────────
FIFO = A B C D
FULL

E valid=1
FIFO ready=0

        │
        │ pop A
        ▼

Cycle N+1
────────────────────────
FIFO = B C D
FULL=false

E valid=1
FIFO ready=1

        │
        │ push E
        ▼

Cycle N+2
────────────────────────
FIFO = B C D E
FULL=true
```

This is **exactly the behavior we wanted to demonstrate**.

---

# 5. One more correction to my previous test

There was another issue in my earlier test code:

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

I specifically warned against unbounded `while` loops because of our earlier UART simulation problem, but then used one here. **That was inconsistent and not a good choice for this learning test.**

We should fix that too.

For this FIFO, where we know the expected ready behavior, we don't need a `while` at all.

Use:

```scala
def push(data: Int): Unit = {

  dut.io.inputData  #= data
  dut.io.inputValid #= true

  assert(
    dut.io.inputReady.toBoolean,
    s"FIFO should be ready before pushing 0x$data"
  )

  dut.clockDomain.waitSampling()

  dut.io.inputValid #= false

  dut.clockDomain.waitFallingEdge()
}
```

This is much safer for our educational simulation.

---

# 6. Corrected complete test

I recommend replacing the previous simulation with this version.

```scala
import spinal.core._
import spinal.core.sim._

object ValidReadyFifoSim extends App {

  SimConfig
    .withWave
    .doSim(
      ValidReadyFifo(
        dataWidth = 8,
        depth = 4
      )
    ) { dut =>

      dut.clockDomain.forkStimulus(10)

      dut.io.inputValid  #= false
      dut.io.inputData   #= 0
      dut.io.outputReady #= false

      dut.clockDomain.waitSampling(2)

      // ----------------------------------------------------------
      // Push one item
      // ----------------------------------------------------------

      def push(data: Int): Unit = {

        dut.io.inputData  #= data
        dut.io.inputValid #= true

        assert(
          dut.io.inputReady.toBoolean,
          f"FIFO should be ready before pushing 0x$data%02X"
        )

        dut.clockDomain.waitSampling()

        dut.io.inputValid #= false

        dut.clockDomain.waitFallingEdge()
      }

      // ----------------------------------------------------------
      // Pop one item
      // ----------------------------------------------------------

      def pop(expected: Int): Unit = {

        assert(
          dut.io.outputValid.toBoolean,
          "FIFO should contain data"
        )

        val actual =
          dut.io.outputData.toBigInt.intValue

        println(
          f"[POP] expected=0x$expected%02X actual=0x$actual%02X"
        )

        assert(
          actual == expected,
          f"Expected 0x$expected%02X but got 0x$actual%02X"
        )

        dut.io.outputReady #= true

        dut.clockDomain.waitSampling()

        dut.io.outputReady #= false

        dut.clockDomain.waitFallingEdge()
      }

      // ==========================================================
      // TEST 1 — Fill FIFO
      // ==========================================================

      println("[TEST] Fill FIFO")

      push(0x41)
      push(0x42)
      push(0x43)
      push(0x44)

      println(
        s"[FULL] " +
        s"inputReady=${dut.io.inputReady.toBoolean} " +
        s"outputValid=${dut.io.outputValid.toBoolean} " +
        f"outputData=0x${dut.io.outputData.toBigInt}%02X"
      )

      assert(
        !dut.io.inputReady.toBoolean,
        "FIFO should not be ready when full"
      )

      assert(
        dut.io.outputValid.toBoolean,
        "FIFO should have valid output when full"
      )

      assert(
        dut.io.outputData.toBigInt == 0x41,
        "FIFO head should be A"
      )

      println("[PASS] FIFO is FULL")

      // ==========================================================
      // TEST 2 — Hold E while FULL
      // ==========================================================

      println("[TEST] Hold E while FIFO is FULL")

      dut.io.inputData  #= 0x45
      dut.io.inputValid #= true

      dut.clockDomain.waitSampling(3)

      assert(
        dut.io.inputValid.toBoolean,
        "Producer must keep valid asserted"
      )

      assert(
        !dut.io.inputReady.toBoolean,
        "FIFO must remain not-ready while full"
      )

      assert(
        dut.io.inputData.toBigInt == 0x45,
        "Producer must keep E stable"
      )

      println(
        f"[HOLD] valid=${dut.io.inputValid.toBoolean} " +
        s"ready=${dut.io.inputReady.toBoolean} " +
        f"data=0x${dut.io.inputData.toBigInt}%02X"
      )

      println("[PASS] E is being held while FIFO is FULL")

      // ==========================================================
      // TEST 3 — Pop A
      // ==========================================================

      println("[TEST] Pop A")

      dut.io.outputReady #= true

      // A is consumed at this sampling edge.
      dut.clockDomain.waitSampling()

      dut.io.outputReady #= false

      // Wait for count/full/inputReady to settle.
      dut.clockDomain.waitFallingEdge()

      assert(
        dut.io.inputValid.toBoolean,
        "E must still be valid"
      )

      assert(
        dut.io.inputData.toBigInt == 0x45,
        "E must still be held"
      )

      assert(
        dut.io.inputReady.toBoolean,
        "FIFO should become ready after one pop"
      )

      assert(
        dut.io.outputValid.toBoolean,
        "FIFO should still contain B/C/D"
      )

      assert(
        dut.io.outputData.toBigInt == 0x42,
        "FIFO head should now be B"
      )

      println(
        f"[SPACE] inputReady=${dut.io.inputReady.toBoolean} " +
        f"head=0x${dut.io.outputData.toBigInt}%02X"
      )

      println("[PASS] FIFO became READY after popping A")

      // ==========================================================
      // TEST 4 — E is now transferred
      // ==========================================================

      println("[TEST] Transfer E")

      // E is already valid.
      // FIFO is now ready.
      // Therefore the next sampling edge performs push(E).

      dut.clockDomain.waitSampling()

      dut.io.inputValid #= false

      dut.clockDomain.waitFallingEdge()

      println("[PASS] E entered FIFO")

      // ==========================================================
      // TEST 5 — Verify FIFO order
      // ==========================================================

      pop(0x42)
      pop(0x43)
      pop(0x44)
      pop(0x45)

      assert(
        !dut.io.outputValid.toBoolean,
        "FIFO should be empty"
      )

      println("[PASS] FIFO order = B -> C -> D -> E")
      println("[Done] Valid/Ready FIFO test passed.")

      simSuccess()
    }
}
```

---

# 7. Expected result

Now the important portion should look like:

```text
[TEST] Fill FIFO
[FULL] inputReady=false outputValid=true outputData=0x41
[PASS] FIFO is FULL

[TEST] Hold E while FIFO is FULL
[HOLD] valid=true ready=false data=0x45
[PASS] E is being held while FIFO is FULL

[TEST] Pop A
[SPACE] inputReady=true head=0x42
[PASS] FIFO became READY after popping A

[TEST] Transfer E
[PASS] E entered FIFO

[POP] expected=0x42 actual=0x42
[POP] expected=0x43 actual=0x43
[POP] expected=0x44 actual=0x44
[POP] expected=0x45 actual=0x45

[PASS] FIFO order = B -> C -> D -> E
[Done] Valid/Ready FIFO test passed.
```

---

# 8. What you just demonstrated

This is the most important waveform sequence in this lesson:

```text
             FULL
              │
              ▼
E valid ──────1─────────────────────────────
E data  ──────45────────────────────────────
FIFO ready ───0───────────────1─────────────
                              │
                              │
                              ▼
                         E transferred
```

The crucial property is:

> **`valid=1, ready=0` → the producer holds the data.**

Then:

> **`valid=1, ready=1` → one transfer occurs.**

And that transfer is exactly:

```scala
fire = valid && ready
```

This is the foundation for SpinalHDL `Stream`.

After this test passes, **the next step should be converting this hand-written `inputValid/inputReady/inputData` interface to an actual `Stream[Bits(8 bits)]`**, and then comparing our FIFO against SpinalHDL's `StreamFifo`.