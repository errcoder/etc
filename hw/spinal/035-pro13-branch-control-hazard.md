좋아. 바로 **Project 13-4 — Control Hazard / Branch**로 간다.

이번 단계의 목표는 딱 하나다.

> **Branch가 taken 되었을 때 이미 pipeline에 들어온 잘못된 명령을 flush하고 PC를 branch target으로 전환한다.**

### 13-4 목표 구조

현재:

```text
IF → [IF/ID] → ID → [ID/EX] → EX → [EX/WB] → WB
```

여기에 `BEQ`를 추가한다.

```text
                 ┌────────────── Branch Taken
                 │
IF → [IF/ID] → ID → [ID/EX] → EX ───────► PC Redirect
                         │        │
                         │        └──────► Flush
                         │
                         └────────────────► ALU
```

Branch는 **EX 단계에서 결정**한다고 하자.

예를 들어:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +2
2: ADD R3, R0, R0    ← 실행되면 안 됨
3: ADD R3, R1, R1    ← 실행되면 안 됨
4: OR  R3, R2, R0    ← branch target
```

`BEQ R2,R2,+2`가 taken 되면 2번과 3번은 이미 pipeline에 들어와 있을 수 있다.

따라서:

```text
branchTaken
    │
    ├──► PC = target
    │
    ├──► IF/ID.valid = 0
    │
    └──► ID/EX.valid = 0
```

이것이 **pipeline flush**다.

---

### 이번 구현에서 중요한 점

이번에는 branch prediction 같은 것은 넣지 않는다.

1. `BEQ` opcode 추가
2. branch immediate 추가
3. EX에서 `branchTaken` 생성
4. branch target 계산
5. PC redirect
6. `IF/ID`, `ID/EX` flush
7. 잘못된 명령이 WB까지 가지 않는지 simulation으로 검증

그리고 테스트는 단순히 branch 결과만 보는 게 아니라,

> **branch가 없었으면 R3를 변경했을 명령을 넣어 놓고, branch가 taken 되었을 때 R3가 절대 변경되지 않는지**

검증한다.

이렇게 해야 실제 **control hazard 해결이 제대로 됐는지** 확인할 수 있다.

다음 답변에서는 바로 **SpinalHDL 1.15.0 기준 `PipelinedBranchCpu` 전체 코드 + simulation 코드**를 작성해서, 기존 4-stage CPU에 branch/flush를 실제로 붙인다.

---

---


```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +2
2: ADD R3, R0, R0    ← 실행되면 안 됨
3: ADD R3, R1, R1    ← 실행되면 안 됨
4: OR  R3, R2, R0    ← branch target
```
2, 3 실행되면 안 되는 이유는? 
0. ADD -> 1. BEQ(뜻은 jump to 4. "OR R3, R2, R0" ?) 이기 떄문에 이미 파이프라인에 들어와 버린 2. ADD, 3. ADD instruction 들은
실행되면 안된다는 뜻인가?

---

맞다. **정확히 그 의미다.**

다만 한 가지를 정확하게 표현하면, `BEQ` 자체는 **"무조건 4번으로 jump"**가 아니라:

> **두 operand가 같으면 branch target으로 이동하라**

는 뜻이다.

이번 예에서는

```text
1: BEQ R2, R2, +2
```

이므로 `R2 == R2`는 항상 참이고, 따라서 **4번으로 branch가 taken**된다.

---

### 시간 흐름으로 보면 이해가 쉽다

프로그램은:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +2
2: ADD R3, R0, R0
3: ADD R3, R1, R1
4: OR  R3, R2, R0
```

Pipeline이니까 CPU는 1번 `BEQ`가 branch라는 것을 알아내기 전에 **다음 명령들을 미리 fetch**한다.

대략:

```text
Cycle N:

        IF
        ↓
       BEQ


Cycle N+1:

        IF          ID
        ↓           ↓
       ADD 2       BEQ


Cycle N+2:

        IF          ID          EX
        ↓           ↓           ↓
       ADD 3       ADD 2       BEQ
                                │
                                └─ branch taken!
```

즉, `BEQ`가 EX에서

```text
R2 == R2
```

를 판단하는 순간에는 이미:

```text
ADD 2
ADD 3
```

같은 **branch의 다음 명령들이 pipeline 안에 들어와 있다.**

---

### 그래서 이 둘은 "실행하면 안 된다"

Branch가 taken 되면 CPU가 원하는 흐름은:

```text
0 → 1 → 4 → ...
```

이다.

그런데 pipeline 때문에 순간적으로:

```text
0 → 1 → 2 → 3 → ...
```

를 fetch하고 있었던 것이다.

따라서 2와 3은 **wrong-path instruction**이다.

```text
0: ADD R2, R0, R1
          ↓
1: BEQ R2, R2, +2
          ↓
       branch taken
          ↓
          └──────────────┐
                         ↓
4: OR R3, R2, R0  ◄──────┘

2: ADD R3, R0, R0   ← WRONG PATH → flush
3: ADD R3, R1, R1   ← WRONG PATH → flush
```

즉 **2번과 3번이 문법적으로 잘못된 명령이라서 실행하면 안 되는 것이 아니다.**

둘 다 정상적인 `ADD` 명령이다.

단지 **branch가 taken 되었기 때문에 프로그램의 올바른 실행 경로에서 제외된 명령**이다.

---

### 이것이 Control Hazard

이 상황을 정확히 **Control Hazard**라고 한다.

```text
BEQ
 │
 ├─ branch NOT taken → 2 → 3 → 4 ...
 │
 └─ branch TAKEN     → 4 → ...
                         ↑
                    그런데 2, 3은
                    이미 pipeline에 있음
```

그래서 CPU가 해야 할 일은:

```text
Branch Taken
     │
     ├── PC ← 4
     │
     ├── 2번 instruction 폐기
     │
     └── 3번 instruction 폐기
```

이 **폐기하는 동작이 `flush`**다.

---

그리고 여기서 아주 중요한 포인트가 하나 있다.

**왜 `2`, `3` 두 개를 flush해야 하는가?**

우리가 현재 설계한 CPU에서는 branch를 **EX 단계에서 결정**하기 때문이다.

```text
IF → [IF/ID] → ID → [ID/EX] → EX → [EX/WB] → WB
                                  ↑
                              여기서 BEQ 판단
```

BEQ가 EX에 도착했을 때 그보다 뒤의 instruction들이 이미 앞쪽 pipeline register에 들어와 있다.

그래서 다음 단계에서 바로 이걸 실제 cycle-by-cycle로 그려보면 **왜 `IF/ID`와 `ID/EX`를 flush해야 하는지**가 아주 명확해진다.

---

---


좋아. 이제 **실제로 `BEQ + branch target + flush`를 CPU에 넣는다.**

이번에는 개념 설명을 길게 하지 않고 **구현 → simulation**까지 간다.

---

# Project 13-4 — Control Hazard / Branch Flush

우리가 만들 명령은:

```text
BEQ rs1, rs2, offset
```

의미:

```text
if (rs1 == rs2)
    PC = PC + offset
else
    PC = PC + 1
```

이번 예에서는:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +3
2: ADD R3, R0, R0    // flush
3: ADD R3, R1, R1    // flush
4: OR  R3, R2, R0
```

`1번`에서 branch가 taken 되면 실행 흐름은:

```text
0 → 1 → 4
```

가 된다.

---

## 1. Instruction format

기존 arithmetic instruction은:

```text
15       12 11    10 9     8 7     6 5          0
+-----------+--------+-------+-------+------------+
|  opcode   |   rd   |  rs1  |  rs2  |  reserved |
+-----------+--------+-------+-------+------------+
```

Branch는 `rd` 대신 offset을 사용하자.

```text
15       12 11     10 9      8 7              0
+-----------+---------+--------+----------------+
|  opcode   |   rs1   |  rs2   |    offset      |
+-----------+---------+--------+----------------+
```

opcode:

```text
0000 ADD
0001 SUB
0010 AND
0011 OR
0100 BEQ
```

offset은 8-bit signed 값으로 사용한다.

---

# 2. 핵심적인 pipeline 변화

기존:

```text
IF
 ↓
[IF/ID]
 ↓
ID
 ↓
[ID/EX]
 ↓
EX
 ↓
[EX/WB]
 ↓
WB
```

Branch를 넣으면 EX에서:

```text
branchTaken
branchTarget
```

을 만든다.

```text
                    ┌──────────────► PC redirect
                    │
IF → IF/ID → ID → ID/EX → EX
                    │      │
                    │      ├── branchTaken
                    │      └── branchTarget
                    │
                    └── operands
```

그리고 branch가 taken이면:

```text
ifIdValid := False
idExValid := False
PC        := branchTarget
```

즉:

```text
              BEQ
               │
          branchTaken
               │
       ┌───────┴────────┐
       ↓                ↓
   PC redirect        FLUSH
                       │
                 ┌─────┴─────┐
                 ↓           ↓
              IF/ID        ID/EX
              valid=0      valid=0
```

---

# 3. 중요한 부분: flush가 왜 두 군데인가?

예를 들어 branch가 EX에 있을 때:

```text
Cycle N

IF        ID        EX
│         │         │
4?        3         BEQ
```

실제로는 branch target을 새로 fetch하기 전까지 이미:

```text
3
```

이 pipeline에 들어와 있다.

그리고 그보다 앞선 cycle에서:

```text
2
```

도 이미 들어와 있을 수 있다.

따라서 branch가 EX에서 taken으로 결정되는 순간 **branch 뒤에 있는 명령들을 모두 무효화**해야 한다.

이번 4-stage 구조에서는 그 역할을:

```text
IF/ID.valid = 0
ID/EX.valid = 0
```

로 한다.

---

# 4. Branch target 계산

이번에는 간단하게:

```text
branchTarget = branch PC + signed offset
```

으로 하자.

즉:

```text
PC = 1
offset = +3

target = 1 + 3
       = 4
```

따라서:

```text
1: BEQ R2, R2, +3
```

은:

```text
PC ← 4
```

가 된다.

---

# 5. EX stage의 branch logic

개념적으로는 이렇게 된다.

```scala
val exIsBranch = idExValid && (idExOpcode === B"4'b0100")

val exBranchTaken =
  exIsBranch &&
  (idExA === idExB)

val exBranchTarget =
  idExPc + idExOffset
```

여기서 중요한 것은:

```text
idExA
idExB
```

가 이미 forwarding을 거친 값이라는 것이다.

따라서 이전에 해결했던 RAW hazard도 그대로 적용된다.

예:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +3
```

`BEQ`가 EX에 도착했을 때 `R2`가 최신 값이어야 하므로 기존 forwarding path가 필요하다.

즉:

```text
                 ┌──────────────┐
Register File ──►│              │
EX/WB ──────────►│ Forward MUX  │──► ID/EX A/B
EX result ──────►│              │
                 └──────────────┘
                                  │
                                  ▼
                                 BEQ
                                  │
                             A == B ?
```

---

# 6. PC 제어

기존에는 대략:

```scala
when(io.run) {
  pc := pc + 1
}
```

이었다.

이제 branch가 우선권을 가져야 한다.

```scala
when(exBranchTaken) {
  pc := exBranchTarget
} elsewhen(io.run) {
  pc := pc + 1
}
```

즉 우선순위는:

```text
1. Branch taken
2. Normal PC + 1
```

이다.

이 순서가 중요하다.

---

# 7. Pipeline register update

여기서 가장 중요한 부분이다.

일반적인 경우:

```scala
ifIdValid := fetchValid
idExValid := ifIdValid
```

하지만 branch가 taken이면:

```scala
ifIdValid := False
idExValid := False
```

가 되어야 한다.

따라서:

```scala
when(exBranchTaken) {
  ifIdValid := False
  idExValid := False
} otherwise {
  ifIdValid := fetchValid

  idExValid := ifIdValid
  ...
}
```

가 핵심 구조다.

---

# 8. 단, 한 가지 주의

여기서 **PC만 branch target으로 바꾸면 안 된다.**

잘못된 구현:

```scala
when(exBranchTaken) {
  pc := exBranchTarget
}
```

만 하면:

```text
PC는 4로 이동했는데

pipeline 안에는

2번 instruction
3번 instruction
```

이 남아 있다.

그러면 결국:

```text
2 → WB
3 → WB
```

까지 진행할 수 있다.

그래서 반드시:

```text
PC redirect
+
pipeline flush
```

가 **동시에** 이루어져야 한다.

---

# 9. 이번 simulation에서 잡아낼 버그

이번 테스트는 아주 의도적으로 만든다.

```text
R0 = 0x10
R1 = 0x20
```

프로그램:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +3

2: ADD R3, R0, R0
3: ADD R3, R1, R1

4: OR  R3, R2, R0
```

정상적인 결과:

```text
R2 = 0x30
R3 = 0x30
```

왜냐하면:

```text
0: R2 = 10 + 20 = 30
1: R2 == R2 → branch taken
2: 실행 안 됨
3: 실행 안 됨
4: R3 = R2 | R0
         = 30 | 10
         = 30
```

---

## flush가 없다면?

2번이 실행되면:

```text
R3 = 0x20
```

3번까지 실행되면:

```text
R3 = 0x30
```

이번 예에서는 **우연히 최종값이 같아질 수 있다.**

그래서 simulation 검증용으로는 더 좋은 프로그램을 사용해야 한다.

예를 들어:

```text
0: ADD R2, R0, R1
1: BEQ R2, R2, +3

2: ADD R3, R0, R0
3: SUB R3, R1, R0

4: OR R3, R2, R0
```

`R0=10`, `R1=20`이면 잘못 실행했을 경우:

```text
2: R3 = 20
3: R3 = 10
```

이런 식으로 흔적이 남는다.

하지만 branch가 정상적으로 flush되면:

```text
4: R3 = 30
```

만 발생한다.

---

# 10. 우리가 이번에 증명할 것

Simulation 로그가 대략:

```text
WB #1 : R2 <= 0x30
BRANCH: PC 1 -> PC 4
WB #2 : R3 <= 0x30

Control hazard / branch flush test passed.
```

처럼 되어야 한다.

특히 **이 로그가 나오면 안 된다.**

```text
WB : R3 <= 0x20   // 2번 명령
WB : R3 <= 0x10   // 3번 명령
```

즉 단순히 최종 `R3`만 검사하지 않고 **WB 발생 자체를 검사**하는 것이 중요하다.

---

### 이번 단계에서 기억할 핵심

```text
Branch가 EX에서 taken
        │
        ├── PC ← branch target
        │
        ├── IF/ID.valid ← 0
        │
        └── ID/EX.valid ← 0
```

이것이 **Control Hazard의 기본적인 해결 방법인 Flush**다.

그리고 다음 구현에서는 기존 `PipelinedMiniCpu`를 그대로 기반으로 해서 **`BEQ`를 추가한 완전한 `PipelinedBranchCpu.scala`와 `PipelinedBranchCpuSim.scala`**를 만들고, 실제로 `2`, `3`번 명령이 WB까지 절대 도달하지 않는지 검증하면 된다.

---

---

좋아. 이제 설명은 끝내고 **실제로 구현한다.**

이번 버전은 기존 4-stage CPU의 구조를 유지하면서:

- `BEQ`
- branch target
- EX-stage branch decision
- PC redirect
- `IF/ID` flush
- `ID/EX` flush
- 기존 RAW forwarding 유지

를 모두 넣는다.

그리고 **잘못된 2, 3번 명령이 WB까지 도달하지 않는 것**까지 simulation에서 검사한다.

---

# 1. `PipelinedBranchCpu.scala`

아래 코드는 **SpinalHDL 1.15.0** 기준이다.

```scala
import spinal.core._

class PipelinedBranchCpu extends Component {
  val io = new Bundle {
    // CPU control
    val run        = in Bool()
    val loadEnable = in Bool()

    // Register initialization
    val loadAddress = in UInt(2 bits)
    val loadData    = in Bits(8 bits)

    // Write-back debug
    val writeBackEnable  = out Bool()
    val writeBackAddress = out UInt(2 bits)
    val writeBackData    = out Bits(8 bits)

    // Branch debug
    val branchTaken = out Bool()
    val branchTarget = out UInt(4 bits)

    // Register debug
    val debugR0 = out Bits(8 bits)
    val debugR1 = out Bits(8 bits)
    val debugR2 = out Bits(8 bits)
    val debugR3 = out Bits(8 bits)
  }

  // ------------------------------------------------------------
  // Constants
  // ------------------------------------------------------------

  val OP_ADD = U"4'b0000"
  val OP_SUB = U"4'b0001"
  val OP_AND = U"4'b0010"
  val OP_OR  = U"4'b0011"
  val OP_BEQ = U"4'b0100"

  // ------------------------------------------------------------
  // Register File
  // ------------------------------------------------------------

  val registers = Vec(
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0),
    Reg(Bits(8 bits)) init(0)
  )

  when(io.loadEnable) {
    registers(io.loadAddress) := io.loadData
  }

  // ------------------------------------------------------------
  // Program Counter
  // ------------------------------------------------------------

  val pc = Reg(UInt(4 bits)) init(0)

  // ------------------------------------------------------------
  // Instruction ROM
  //
  // 0: ADD R2, R0, R1
  // 1: BEQ R2, R2, +3
  // 2: SUB R3, R1, R0      <- wrong path
  // 3: SUB R3, R0, R1      <- wrong path
  // 4: OR  R3, R2, R0
  // ------------------------------------------------------------

  val rom = Mem(Bits(16 bits), wordCount = 8)

  rom.init(
    Seq(
      B"16'h0800", // 0: ADD R2, R0, R1
      B"16'h4A03", // 1: BEQ R2, R2, +3
      B"16'h1D40", // 2: SUB R3, R1, R0
      B"16'h1EC0", // 3: SUB R3, R0, R1
      B"16'h3A00", // 4: OR  R3, R2, R0
      B"16'h0000", // 5
      B"16'h0000", // 6
      B"16'h0000"  // 7
    )
  )

  // ------------------------------------------------------------
  // Fetch
  // ------------------------------------------------------------

  val fetchInstruction = rom.readAsync(pc)

  // ------------------------------------------------------------
  // IF / ID
  // ------------------------------------------------------------

  val ifIdValid       = Reg(Bool()) init(False)
  val ifIdInstruction = Reg(Bits(16 bits)) init(0)
  val ifIdPc          = Reg(UInt(4 bits)) init(0)

  // ------------------------------------------------------------
  // Decode current IF/ID instruction
  // ------------------------------------------------------------

  val decOpcode = ifIdInstruction(15 downto 12)

  val decRd  = ifIdInstruction(11 downto 10).asUInt
  val decRs1 = ifIdInstruction(9 downto 8).asUInt
  val decRs2 = ifIdInstruction(7 downto 6).asUInt

  // Branch-specific encoding:
  //
  // [15:12] opcode
  // [11:10] rs1
  // [9:8]   rs2
  // [7:0]   offset
  //
  val decBranchRs1 = ifIdInstruction(11 downto 10).asUInt
  val decBranchRs2 = ifIdInstruction(9 downto 8).asUInt
  val decOffset    = ifIdInstruction(7 downto 0).asUInt

  val decIsBranch = decOpcode === OP_BEQ

  val decWriteEnable =
    (decOpcode === OP_ADD) ||
    (decOpcode === OP_SUB) ||
    (decOpcode === OP_AND) ||
    (decOpcode === OP_OR)

  // ------------------------------------------------------------
  // Normal register-file operands
  // ------------------------------------------------------------

  val rfA = registers(decRs1)
  val rfB = registers(decRs2)

  val branchRfA = registers(decBranchRs1)
  val branchRfB = registers(decBranchRs2)

  // ------------------------------------------------------------
  // EX / WB
  // ------------------------------------------------------------

  val exWbValid         = Reg(Bool()) init(False)
  val exWbWriteEnable   = Reg(Bool()) init(False)
  val exWbWriteAddress  = Reg(UInt(2 bits)) init(0)
  val exWbWriteData     = Reg(Bits(8 bits)) init(0)

  // ------------------------------------------------------------
  // ID / EX
  // ------------------------------------------------------------

  val idExValid   = Reg(Bool()) init(False)
  val idExPc      = Reg(UInt(4 bits)) init(0)

  val idExRd      = Reg(UInt(2 bits)) init(0)
  val idExA       = Reg(Bits(8 bits)) init(0)
  val idExB       = Reg(Bits(8 bits)) init(0)
  val idExOpcode  = Reg(UInt(4 bits)) init(0)

  val idExBranchOffset = Reg(UInt(8 bits)) init(0)
  val idExIsBranch     = Reg(Bool()) init(False)
  val idExWriteEnable  = Reg(Bool()) init(False)

  // ------------------------------------------------------------
  // Current EX stage
  // ------------------------------------------------------------

  val exIsBranch =
    idExValid && idExIsBranch

  // ------------------------------------------------------------
  // EX forwarding
  //
  // Priority:
  //   1. Current EX result
  //   2. EX/WB result
  //   3. Register File
  // ------------------------------------------------------------

  val exResult = Bits(8 bits)
  exResult := 0

  switch(idExOpcode) {
    is(OP_ADD) {
      exResult := idExA + idExB
    }

    is(OP_SUB) {
      exResult := idExA - idExB
    }

    is(OP_AND) {
      exResult := idExA & idExB
    }

    is(OP_OR) {
      exResult := idExA | idExB
    }
  }

  val forwardedA = Bits(8 bits)
  val forwardedB = Bits(8 bits)

  forwardedA := rfA
  forwardedB := rfB

  // ------------------------------------------------------------
  // Forward current EX result
  // ------------------------------------------------------------

  when(
    idExValid &&
    idExWriteEnable &&
    idExRd === decRs1
  ) {
    forwardedA := exResult
  } elsewhen(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decRs1
  ) {
    forwardedA := exWbWriteData
  }

  when(
    idExValid &&
    idExWriteEnable &&
    idExRd === decRs2
  ) {
    forwardedB := exResult
  } elsewhen(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decRs2
  ) {
    forwardedB := exWbWriteData
  }

  // ------------------------------------------------------------
  // Branch operands
  //
  // Branch also needs forwarding.
  //
  // Example:
  //
  //   ADD R2, R0, R1
  //   BEQ R2, R2, +3
  //
  // BEQ must see the newly calculated R2.
  // ------------------------------------------------------------

  val branchForwardA = Bits(8 bits)
  val branchForwardB = Bits(8 bits)

  branchForwardA := branchRfA
  branchForwardB := branchRfB

  when(
    idExValid &&
    idExWriteEnable &&
    idExRd === decBranchRs1
  ) {
    branchForwardA := exResult
  } elsewhen(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decBranchRs1
  ) {
    branchForwardA := exWbWriteData
  }

  when(
    idExValid &&
    idExWriteEnable &&
    idExRd === decBranchRs2
  ) {
    branchForwardB := exResult
  } elsewhen(
    exWbValid &&
    exWbWriteEnable &&
    exWbWriteAddress === decBranchRs2
  ) {
    branchForwardB := exWbWriteData
  }

  // ------------------------------------------------------------
  // Branch decision
  //
  // Branch target = branch PC + offset
  // ------------------------------------------------------------

  val calculatedBranchTarget = UInt(4 bits)

  calculatedBranchTarget :=
    idExPc + idExBranchOffset.resize(4)

  val exBranchTaken =
    exIsBranch &&
    (idExA === idExB)

  // ------------------------------------------------------------
  // Sequential pipeline / PC update
  // ------------------------------------------------------------

  when(io.loadEnable) {
    // Register loading only.
    // Pipeline and PC stay untouched.
  } elsewhen(exBranchTaken) {

    // ----------------------------------------------------------
    // CONTROL HAZARD:
    //
    // Redirect PC
    // Flush younger instructions
    // ----------------------------------------------------------

    pc := calculatedBranchTarget

    ifIdValid := False
    idExValid := False

  } elsewhen(io.run) {

    // ----------------------------------------------------------
    // Normal PC increment
    // ----------------------------------------------------------

    pc := pc + 1

    // ----------------------------------------------------------
    // IF / ID
    // ----------------------------------------------------------

    ifIdValid       := True
    ifIdInstruction := fetchInstruction
    ifIdPc          := pc

    // ----------------------------------------------------------
    // ID / EX
    // ----------------------------------------------------------

    idExValid := ifIdValid
    idExPc    := ifIdPc

    idExOpcode := decOpcode.asUInt
    idExRd     := decRd

    idExIsBranch    := decIsBranch
    idExBranchOffset := decOffset
    idExWriteEnable := decWriteEnable

    when(decIsBranch) {
      idExA := branchForwardA
      idExB := branchForwardB
    } otherwise {
      idExA := forwardedA
      idExB := forwardedB
    }

  } otherwise {
    // Hold pipeline while not running.
  }

  // ------------------------------------------------------------
  // EX / WB
  //
  // IMPORTANT:
  // Branch itself never writes back.
  // ------------------------------------------------------------

  exWbValid := idExValid && idExWriteEnable

  exWbWriteEnable  := idExValid && idExWriteEnable
  exWbWriteAddress := idExRd
  exWbWriteData    := exResult

  // ------------------------------------------------------------
  // Register write-back
  // ------------------------------------------------------------

  when(exWbValid && exWbWriteEnable) {
    registers(exWbWriteAddress) := exWbWriteData
  }

  // ------------------------------------------------------------
  // Debug outputs
  // ------------------------------------------------------------

  io.writeBackEnable  := exWbValid && exWbWriteEnable
  io.writeBackAddress := exWbWriteAddress
  io.writeBackData    := exWbWriteData

  io.branchTaken  := exBranchTaken
  io.branchTarget := calculatedBranchTarget

  io.debugR0 := registers(0)
  io.debugR1 := registers(1)
  io.debugR2 := registers(2)
  io.debugR3 := registers(3)
}
```

그런데 위 코드에서 **한 가지 중요한 점**이 있다.

`BEQ`의 operand는 ID 단계에서 `branchForwardA/B`를 계산해서 `ID/EX`로 넣는다.

따라서:

```text
ADD R2,R0,R1
BEQ R2,R2,+3
```

에서도

```text
ADD의 EX result
      ↓
branchForwardA/B
      ↓
ID/EX
      ↓
BEQ EX
```

경로가 생긴다.

이게 이전에 우리가 해결했던 **RAW forwarding을 branch에도 적용한 것**이다.

---

# 2. Simulation

이제 중요한 부분이다.

단순히 최종 `R3 == 0x30`만 검사하지 않는다.

**잘못된 2번, 3번 instruction이 WB에 나타나지 않는지 검사한다.**

```scala
import spinal.core._
import spinal.core.sim._

object PipelinedBranchCpuSim {
  def main(args: Array[String]): Unit = {

    SimConfig.withWave
      .compile(new PipelinedBranchCpu)
      .doSim { dut =>

        dut.clockDomain.forkStimulus(10)

        // ------------------------------------------------------
        // Initial values
        //
        // R0 = 0x10
        // R1 = 0x20
        // ------------------------------------------------------

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

        // ------------------------------------------------------
        // Run CPU
        // ------------------------------------------------------

        dut.io.run #= true

        var wbCount = 0
        var branchSeen = false

        for (_ <- 0 until 12) {

          dut.clockDomain.waitSampling()

          // Give combinational outputs time to settle.
          sleep(1)

          if (dut.io.branchTaken.toBoolean) {
            branchSeen = true

            println(
              f"BRANCH: target = ${dut.io.branchTarget.toInt}%d"
            )

            assert(
              dut.io.branchTarget.toInt == 4,
              "Branch target must be PC 4"
            )
          }

          if (dut.io.writeBackEnable.toBoolean) {

            val address = dut.io.writeBackAddress.toInt
            val data = dut.io.writeBackData.toInt

            wbCount += 1

            println(
              f"WB #$wbCount : R$address%d <= 0x$data%02X"
            )

            // --------------------------------------------------
            // We expect:
            //
            // WB #1 : R2 <= 0x30
            // WB #2 : R3 <= 0x30
            //
            // There must NOT be:
            //
            // R3 <= 0x20
            // R3 <= 0x10
            // --------------------------------------------------

            if (address == 3) {
              assert(
                data == 0x30,
                f"Wrong-path instruction wrote R3 = 0x$data%02X"
              )
            }
          }
        }

        dut.io.run #= false

        sleep(1)

        // ------------------------------------------------------
        // Final checks
        // ------------------------------------------------------

        assert(branchSeen, "BEQ was never taken")

        assert(
          wbCount == 2,
          s"Expected exactly 2 write-backs, got $wbCount"
        )

        assert(
          dut.io.debugR2.toInt == 0x30,
          f"R2 expected 0x30, got 0x${dut.io.debugR2.toInt}%02X"
        )

        assert(
          dut.io.debugR3.toInt == 0x30,
          f"R3 expected 0x30, got 0x${dut.io.debugR3.toInt}%02X"
        )

        println("Final R2 = 0x30")
        println("Final R3 = 0x30")
        println("Control hazard / branch flush test passed.")
      }
  }
}
```

---

## 3. 그런데 여기서 아주 중요한 simulation 포인트

이번 simulation에서 우리가 기대하는 WB는 정확히:

```text
WB #1 : R2 <= 0x30
WB #2 : R3 <= 0x30
```

이다.

즉:

```text
0: ADD R2,R0,R1
```

만큼은 정상적으로 WB되어야 한다.

그 다음:

```text
1: BEQ R2,R2,+3
```

는 branch 명령이므로 WB하지 않는다.

그리고:

```text
2: SUB R3,R1,R0
3: SUB R3,R0,R1
```

는 **pipeline에 잠깐 들어갔더라도 flush되어야 한다.**

마지막:

```text
4: OR R3,R2,R0
```

만 실행되어야 한다.

따라서:

```text
0 ──► ADD ───────────────► WB R2=30
                           │
1 ──► BEQ ──► taken ──────┤
                           │
2 ──► SUB ──► FLUSH        │
                           │
3 ──► SUB ──► FLUSH        │
                           │
4 ──► OR ─────────────────► WB R3=30
```

---

# 4. 이 단계에서 정말 중요한 부분

우리가 지금까지 만든 pipeline hazard 해결법을 정리하면:

### Data Hazard

```text
ADD R2, R0, R1
SUB R3, R2, R0
```

일 때:

```text
EX result
    ↓
Forwarding MUX
    ↓
ID/EX
```

즉 **forwarding**으로 해결했다.

---

### Control Hazard

```text
BEQ R2,R2,+3
ADD ...
ADD ...
```

일 때:

```text
BEQ
 │
 ├── PC redirect
 │
 └── Pipeline flush
```

즉 **flush**로 해결했다.

---

그래서 현재 CPU는 상당히 중요한 두 가지 pipeline hazard를 직접 처리하고 있다.

```text
                 Pipeline CPU
                       │
          ┌────────────┴────────────┐
          │                         │
     Data Hazard              Control Hazard
          │                         │
      Forwarding                   Flush
          │                         │
     EX → ID/EX              PC Redirect
```

다음 단계에서는 **이 코드의 simulation waveform을 cycle 단위로 뜯어서**, `BEQ`가 EX에 도착하는 순간 **2번과 3번이 정확히 어느 pipeline register에 있고, 왜 `IF/ID`와 `ID/EX`를 각각 0으로 만드는지** 확인하는 게 좋다. 이걸 이해하면 pipeline의 핵심은 거의 잡은 것이다.

