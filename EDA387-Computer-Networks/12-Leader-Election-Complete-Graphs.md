# Self-Stabilizing Leader Election in Complete Graphs (Anonymous Systems, Randomization)

**Course:** EDA387/DIT663 Computer Networks, part 2 (Fault-tolerant Algorithms for Computer Networks), lecture "Self-stabilizing Leader Election", second part (Dolev, *Self-Stabilization*, §2.4 and §2.9: randomized self-stabilization and the scheduler-luck game). The first part of the same lecture, leader election in general ID-based graphs, has its own guide.

**Revised:** 21 October 2026

**† Supplementary material:** content that comes only from hidden PowerPoint slides. It may not be part of this year's examinable material. *This part of the lecture has no hidden slides, so nothing in this guide carries the † mark.*

**How to read this guide.** Every part starts with **In one sentence**, the whole idea in one line. Then comes the explanation in small steps, a **Kid example** (the same idea told with everyday things), pictures, and a **Remember** list with what you must know for the exam. **Technical clarification** marks a place where the slides are wrong or unclear, and gives the correct version. **Additional context** adds something useful that is not on the slides.

---

## Glossary

| Term | Full name | Meaning in simple words |
|---|---|---|
| Anonymous system | Uniform system | All processors are identical: same program, no identifiers. |
| Atomic step | — | A step the scheduler cannot interrupt or look inside. |
| Central daemon | Central demon | A scheduler that activates exactly one processor at a time for one atomic step. |
| Coarse atomicity | — | A coin toss is part of the same atomic step as the following read or write, so the scheduler cannot see the result before it is written. |
| Combined probability, *cp* | — | The product of the probabilities of all of luck's interventions in the SL-game. |
| Complete graph | Fully connected graph | A topology in which every processor is directly connected to every other processor. |
| (*cp*, *r*)-strategy | — | A strategy for luck that reaches a safe configuration within at most *r* expected rounds, using interventions with combined probability *cp*. |
| Distributed daemon | Distributed demon | A scheduler that may activate any number of processors concurrently. |
| Expected value | — | The long-run average of a random quantity. |
| Fine atomicity | — | A coin toss is a separate atomic step, so the scheduler sees the result before it is written. |
| Geometric distribution | — | The number of tries until the first success when each try succeeds independently with probability *cp*; its expected value is 1/*cp*. |
| LE | Leader Election task | The set of executions in which there is a single fixed leader throughout the execution. |
| Leader register, leaderᵢ | — | A one-bit, single-writer multi-reader register: 1 means "I am a leader candidate", 0 means "I am not". |
| *lᵢ*[*j*] | — | *pᵢ*'s local copy of the last value it read from leaderⱼ. |
| Luck | — | The player in the SL-game who may fix the outcomes of coin tosses. |
| Randomized self-stabilizing | — | From any configuration and under any fair scheduler, the algorithm reaches a safe configuration within a finite number of rounds (with probability 1; here, in a finite expected number). |
| Scheduler | Adversary | The player that chooses which processor moves next, trying to prevent stabilization. |
| SL-game | Scheduler-luck game | A proof technique for bounding the expected stabilization time of randomized algorithms. |
| Symmetry breaking | — | Making identical processors end up in different roles (for example one leader and many followers). |

## Contents

- [Before you start: the words you need](#before-you-start-the-words-you-need)
- [1. Deterministic leader election without identifiers](#1-deterministic-leader-election-without-identifiers)
  - [1.1 The question: leader election without identifiers](#11-the-question-leader-election-without-identifiers)
  - [1.2 A deterministic solution under a central daemon](#12-a-deterministic-solution-under-a-central-daemon)
  - [1.3 No deterministic solution under a distributed daemon](#13-no-deterministic-solution-under-a-distributed-daemon)
- [2. Randomized self-stabilization](#2-randomized-self-stabilization)
  - [2.1 Randomized self-stabilization](#21-randomized-self-stabilization)
  - [2.2 Coarse versus fine atomicity](#22-coarse-versus-fine-atomicity)
  - [2.3 The scheduler-luck game](#23-the-scheduler-luck-game)
- [3. The randomized leader election algorithm](#3-the-randomized-leader-election-algorithm)
  - [3.1 The randomized leader election algorithm](#31-the-randomized-leader-election-algorithm)
  - [3.2 Lemma 2.6: stabilization within 2^O(n) expected rounds](#32-lemma-26-stabilization-within-2on-expected-rounds)
  - [3.3 Lemma 2.7: no stabilization under fine atomicity](#33-lemma-27-no-stabilization-under-fine-atomicity)
- [4. Summary](#4-summary)
  - [4.1 Recap](#41-recap)
- [Revision notes and references](#revision-notes-and-references)

## Before you start: the words you need

- **Complete graph:** every processor is connected to **every** other processor. Everybody can see everybody.
- **Anonymous:** the processors have **no names (no IDs)** and run exactly the same program. They are like identical twins.
- **Deterministic:** no randomness. The same situation always gives the same action.
- **Randomized:** the processor may **flip a coin** and act on the result.
- **Scheduler (adversary):** the "bad guy" who decides who moves next and tries to stop the algorithm from working.
- **Expected value:** the average you get if you repeat something very many times. For example, a fair coin needs on average **2** throws to show heads.

**A complete graph with 4 processors** (every pair is linked):

```mermaid
flowchart LR
  A((p1)) --- B((p2))
  A --- C((p3))
  A --- D((p4))
  B --- C
  B --- D
  C --- D
```

> **Kid example.** Five identical twins sit around a table, all wearing the same clothes and with no name tags. They must choose exactly **one** of them to answer the phone. If they all follow the same rule at the same time, they all pick up or nobody does. This lecture shows when a simple rule is enough, when it is not, and how flipping coins helps.

## 1. Deterministic leader election without identifiers

**What this chapter is about:** can identical processors elect one leader without coins? Yes if they move one at a time, no if they may move together.

### 1.1 The question: leader election without identifiers

**In one sentence:** without IDs all processors look the same, so something must make one of them different ("break the symmetry").

The first part of the lecture elected the minimum identifier in an ID-based network. Now the question is:

**Is there a deterministic solution without identifiers in fully connected graphs?**

A leader election algorithm must end in a configuration where **exactly one** processor is the leader and every processor knows whether it is the leader or not. That final configuration is **not symmetric**. In an ID-based system, the identifiers already make processors different. In an **anonymous (uniform)** system all processors are identical, so something must **break the symmetry**. The answer depends on the scheduler.

**Additional context (Dolev §2.9).** in an ID-based complete graph the problem is trivial: every processor repeatedly appoints the processor with the maximal identifier. The difficulty here comes purely from having no identifiers.

```mermaid
flowchart LR
  A["START: all identical<br/>(symmetric)"] -- "something must<br/>break the symmetry" --> B["END: one leader,<br/>the others followers<br/>(NOT symmetric)"]
```

> **Kid example.** If the children have name tags, it is easy: "the child with the biggest number answers the phone". Without name tags, all the children are the same, so you need some trick to make exactly one of them different.

**Remember:**

- The final configuration (one leader) is **not symmetric**; anonymous processors start symmetric ⇒ need **symmetry breaking**.
- With IDs in a complete graph it is trivial (pick the maximal ID).
- Whether a deterministic solution exists **depends on the scheduler**.

### 1.2 A deterministic solution under a central daemon

**In one sentence:** if only one person moves at a time, the rule "stand up if nobody else is standing, otherwise sit down" always ends with exactly one person standing.

Under the **central daemon** model, **exactly one** processor among all processors is selected, arbitrarily, by an adversarial scheduler, to take a single atomic step.

The lecture illustrates it with five people in a circle, each **sitting** or **standing**. The goal is that exactly one person stands. The rule: when the scheduler picks you, look at everybody else; **stand up if nobody else is standing; otherwise sit down**.

- After the scheduler's first choice, **at least one person is standing**: either someone was already standing, or the chosen person saw nobody standing and stood up.
- If *x* > 1 people stand, then once *x* − 1 of them have been picked, each of them saw someone else standing and sat down. Exactly one person is left standing.
- From then on, everybody else stays seated (they always see the leader standing) and the leader stays standing (it never sees anyone else standing).

A possible run with five people (S = standing, s = sitting), in the spirit of the lecture's animation:

| Picked | Before | Rule applied | After |
|---|---|---|---|
| person 1 | s S s S S | others are standing: sit | s S s S S |
| person 2 | s S s S S | others are standing: sit | s s s S S |
| person 4 | s s s S S | others are standing: sit | s s s s S |
| person 5 | s s s s S | nobody else stands: stand | s s s s S |
| person 3 | s s s s S | others are standing: sit | s s s s S |

From the third pick on, exactly one person (person 5) is standing, and it stays that way.

**The rule, as a picture:**

```mermaid
flowchart LR
  P["you are picked:<br/>look at everybody else"] --> Q["is anybody else<br/>standing?"]
  Q -- "no" --> U["STAND UP"]
  Q -- "yes" --> D["SIT DOWN"]
```

> **Kid example.** A teacher taps one child on the shoulder at a time. The tapped child looks around: "Is anybody else standing? No? Then I stand. Yes? Then I sit." Because only one child moves at a time, two children can never jump up at the same moment, so the class always ends with exactly one child standing.

**Remember:**

- **Central daemon** = one processor per atomic step.
- Rule: **stand if nobody else stands, otherwise sit**.
- After the first pick at least one stands; extra standers sit when picked; then exactly one stays standing forever.

### 1.3 No deterministic solution under a distributed daemon

**In one sentence:** if the scheduler may move everybody at the same moment, identical processors always do the same thing, so they all stand, all sit, all stand, … forever.

Under the **distributed daemon** model, an arbitrary number of processors are selected by an adversarial scheduler to **concurrently** take atomic steps. Now the same rule fails:

1. Everybody is sitting. The scheduler activates **all** of them at once. Each looks around, sees nobody standing, and decides to stand.
2. All stand up. The scheduler activates all of them again. Each sees others standing and decides to sit.
3. All sit down. Back to step 1, forever.

```mermaid
stateDiagram-v2
    AllSitting: All sitting
    AllStanding: All standing
    AllSitting --> AllStanding: everyone activated together,<br/>each sees nobody standing
    AllStanding --> AllSitting: everyone activated together,<br/>each sees others standing
```

Since all processors are identical, start in the same state and see the same thing, any deterministic rule makes them all do the same thing. The scheduler can keep them perfectly in step forever. **Unfortunately, there is no deterministic solution under a distributed daemon.** That is why we turn to randomization.

**Additional context (Dolev §2.9).** the book concludes that there is no self-stabilizing uniform leader election algorithm without a central daemon (meaning a deterministic one); the randomized algorithm below needs no central daemon.

> **Kid example.** Now the teacher shouts "everybody, go!". All the children are sitting, each one sees nobody standing, so **all** stand up together. Next "go!": each sees others standing, so **all** sit down together. Identical children with the same rule and the same view always do the same thing, so this goes on forever. Only a coin flip (something different for each child) can break the tie.

**Remember:**

- **Distributed daemon** = any number of processors move **concurrently**.
- Identical processors + same view + deterministic rule ⇒ **same action** ⇒ the scheduler keeps them in step forever.
- **No deterministic solution under a distributed daemon** ⇒ use **randomization**.

## 2. Randomized self-stabilization

**What this chapter is about:** what "randomized self-stabilizing" means, why it matters exactly when the scheduler can see a coin, and a game (scheduler against luck) that turns probability proofs into simple ones.

### 2.1 Randomized self-stabilization

**In one sentence:** a randomized algorithm is self-stabilizing if, from any start and against any fair scheduler, it reaches a safe configuration within a finite expected number of rounds.

**Assumptions and definitions:**

- Processor activity is managed by a **scheduler**.
- The scheduler's assumption: **at most one atomic step is executed at any given time**, and **each atomic step includes a single read or write operation and a finite sequence of local computation**.
- The scheduler is regarded as an **adversary**: it aims to increase the algorithm's costs as much as possible.
- The scheduler is assumed to be **fair**, yet it has **unlimited resources** and chooses the next activated processor **online** (step by step, based on what has happened so far).

**An algorithm is randomized self-stabilizing for task LE if, starting with any system configuration and considering any fair scheduler, the algorithm reaches a safe configuration within a finite number of rounds** (in expectation; see the SL-game below).

**Randomized algorithms are often used to break symmetry** in a network of totally identical processors: if each processor flips its own coin, they will sooner or later get different results.

> **Kid example.** Five identical twins must pick one of them to answer the phone. If each one flips a coin, sooner or later exactly one gets heads while the others get tails. Nobody can say exactly when, but on average it does not take forever.

**Remember:**

- Scheduler = **adversary**: fair, unlimited resources, chooses **online**.
- One atomic step at a time; a step = **one read or write** + local computation.
- **Randomized self-stabilizing for LE**: any start, any fair scheduler ⇒ safe configuration within a finite (expected) number of rounds.
- Randomization **breaks symmetry**.

### 2.2 Coarse versus fine atomicity

**In one sentence:** with coarse atomicity the coin result is written before the scheduler can see it; with fine atomicity the scheduler sees the coin first and can play tricks.

When a processor tosses a coin, can the adversary see the result before the processor acts on it? This depends on the **atomicity** assumed:

- **Coarse atomicity, less power to the adversary.** A coin toss is an internal operation that is **not separable from the next read or write**. The adversary is **unaware** of the algorithm's random choice before that read or write has happened.
- **Fine atomicity, more power to the adversary.** A coin toss is a **separate operation** from the next read or write. The adversary **is aware** of the random choice before the read or write, and can schedule other processors in between.

**Coarse atomicity: one atomic step** (nothing can happen in the middle)

```mermaid
flowchart LR
    C1["toss coin"] --> C2["write result"]
```

**Fine atomicity: two atomic steps** (the scheduler peeks in the middle)

```mermaid
flowchart LR
    F1["toss coin"] -. "scheduler can see the result<br/>and run others here" .-> F2["write result"]
```

We will see that the leader election algorithm below **works under coarse atomicity** (Lemma 2.6) and **fails under fine atomicity** (Lemma 2.7).

> **Kid example.** Coarse atomicity is like tossing a coin under a cup and acting on it in the same instant: the troublemaker (the scheduler) cannot peek. Fine atomicity is like tossing the coin openly on the table and only acting a moment later: the troublemaker sees the result first and can let other players move in a way that spoils your plan.

**Remember:**

- **Coarse**: toss + next read/write = **one** atomic step ⇒ adversary **cannot see** the coin ⇒ **less** power.
- **Fine**: toss = **separate** step ⇒ adversary **sees** the coin ⇒ **more** power.
- The leader election algorithm **works under coarse** (Lemma 2.6), **fails under fine** (Lemma 2.7).

### 2.3 The scheduler-luck game

**In one sentence:** pretend "luck" chooses the coin results; if luck can win with combined probability *cp* in *r* rounds, the real algorithm stabilizes in at most *r*/*cp* expected rounds.

**Purpose:** to prove **upper bounds on the stabilization time of randomized algorithms** via probability calculations. Analysing every possible outcome of every random choice is very complicated, especially for self-stabilizing algorithms that make no assumption about the initial state. The **SL-game** avoids this: it **fixes** the outcome of the random function, so the proof looks like a proof of a deterministic algorithm.

#### Rules of the game

Given a randomized algorithm *AL*, we define a game between two players, **scheduler** and **luck**:

- **Scheduler** wants to **prevent** the algorithm from fulfilling its task. It chooses the activation order (interleaving) of the processors.
- **Luck** may **determine the result** of the random function.

In each turn:

1. The scheduler chooses the next processor to activate, and it makes a step.
2. If during this step the processor uses a random function, **luck may intervene** and fix the result (or a subset of the desired results). If luck fixes the result to be any of *g* values out of *h* possible results, the probability of this intervention is ***p* = *g*/*h***.
3. Luck may also choose **not** to intervene; then every result has equal probability.
4. Both players have unlimited computational resources and decide based on the history so far. When the step ends, a new turn begins.

```mermaid
sequenceDiagram
    participant S as Scheduler (adversary)
    participant P as Activated processor
    participant L as Luck
    loop every turn
        S->>P: choose which processor takes the next step
        alt the step uses a random function
            P->>L: coin toss requested
            L-->>P: may fix the outcome (probability g/h)
        end
        P-->>S: step finished, new configuration
    end
```

**Examples of *p* = *g*/*h*:** luck fixes a fair coin to "heads": *g* = 1 of *h* = 2, so *p* = 1/2. Luck fixes a die to "5 or 6": *g* = 2 of *h* = 6, so *p* = 1/3.

> **Kid example.** Two players play a board game with the algorithm. The **scheduler** is a naughty player who decides whose turn it is and wants the game to go badly. **Luck** is a friendly player who is allowed to "decide" what the dice show, but every time luck does so, we write down how unlikely that throw was. In the end we multiply all those chances together: that is *cp*.

**Definitions.**

- If luck intervenes ℓ times and the *x*-th intervention has probability Pr(*x*), the **combined probability** is

```math
cp = \prod_{x=1}^{\ell} \Pr(x).
```

- ***r*** is the expected number of rounds until stabilization, assuming every intervention succeeds.
- Luck has a **(*cp*, *r*)-strategy** to win the game if it has a strategy to reach a safe configuration in an expected number of at most *r* rounds, with interventions whose combined probability is no more than *cp*.

**Example:** luck fixes three fair coins. Then *cp* = ½ · ½ · ½ = 1/8.

#### Theorem 2.4: from a luck strategy to an expected time

**Theorem 2.4.** If luck has a (*cp*, *r*)-strategy, then the algorithm reaches a safe configuration within at most ***r*/*cp*** expected number of rounds.

*Proof sketch.*

1. In a real execution, the random function's outputs may differ from what luck would have chosen. Then the system may reach an arbitrary state.
2. But from that arbitrary configuration **a new SL-game can be started**, in which, with probability *cp*, a safe configuration is reached within *r* expected rounds. Once in a safe configuration, luck is no longer needed.
3. So each "try" succeeds with probability *cp*. The probability that the first try fails and the second succeeds is (1 − *cp*) · *cp*; in general, the *x*-th try is the first success with probability (1 − *cp*)ˣ⁻¹ · *cp*.
4. The expected number of tries is

```math
\sum_{x=1}^{\infty} x \,(1-cp)^{x-1}\, cp = \frac{1}{cp}.
```

5. Each try takes *r* expected rounds. Since the expectation of a sum is the sum of the expectations, the expected number of rounds until a safe configuration is ***r*/*cp***. ∎

**Additional context (checking the sum).** the formula Σ *x*(1 − *q*)ˣ⁻¹*q* = 1/*q* is the mean of a geometric distribution. Quick check with *q* = 1/2: 1·½ + 2·¼ + 3·⅛ + 4·1/16 + … = 2 = 1/*q*.

```mermaid
flowchart LR
  T1["try 1<br/>(r rounds)"] -- "real coins disagree<br/>(prob 1 − cp)" --> T2["try 2<br/>(r rounds)"] -- "disagree again" --> T3["try 3 ..."]
  T1 -- "coins agree<br/>(prob cp)" --> S["SAFE"]
  T2 -- "agree (prob cp)" --> S
  T3 -- "agree (prob cp)" --> S
```

On average we need 1/*cp* tries, and each try costs *r* rounds, so on average *r* · (1/*cp*) = *r*/*cp* rounds.

> **Kid example.** To win a game you need a lucky streak that happens with probability *cp* = 1/8 and takes *r* = 10 rounds to play out. On average you need 8 attempts, so on average 8 · 10 = 80 rounds. That is *r*/*cp* = 10 / (1/8) = 80.

**Remember:**

- **SL-game**: **scheduler** (adversary, chooses who moves) versus **luck** (may fix coin results).
- One intervention fixing *g* of *h* results has probability **p = g/h**; **cp** = product of all interventions' probabilities.
- **(*cp*, *r*)-strategy**: luck reaches safe within *r* expected rounds with combined probability *cp*.
- **Theorem 2.4**: expected stabilization time **≤ *r*/*cp***, because the number of tries is **geometric** with mean **1/*cp***.

## 3. The randomized leader election algorithm

**What this chapter is about:** a tiny coin-flipping algorithm, the proof that it works under coarse atomicity (with a time bound), and the trick that breaks it under fine atomicity.

### 3.1 The randomized leader election algorithm

**In one sentence:** every processor reads all leader bits and flips a new coin for its own bit if it sees **no** leader while being 0, or **another** leader while being 1.

The studied algorithm uses the **shared-memory model** for networks with the topology of a **complete graph** (every processor can read every other processor's register). Each processor *pᵢ* has a one-bit **leader register** leaderᵢ that it writes and everyone reads. The coin returns 0 or 1 with equal probability.

```text
01 do forever
02     forall p_j ∈ N(i) do
03         l_i[j] := read(leader_j)
04     if (leader_i = 0 and {∀ j ≠ i | l_i[j] = 0}) or
05        (leader_i = 1 and {∃ j ≠ i | l_i[j] = 1}) then
06         write leader_i := random({0, 1})
07 end
```

In words: *pᵢ* reads everybody's leader bit. It tosses a coin and writes the result into its own register in two situations:

- it is **not** a leader and sees **no** leader at all (somebody must try to become leader), or
- it **is** a leader but sees **another** leader too (too many leaders, somebody must try to step down).

Otherwise (exactly the right situation from its point of view) it does nothing.

```mermaid
flowchart TD
    S(["pi's iteration"]) --> R["Read leader_j of every other processor into l_i[j]"]
    R --> A{"leader_i = 0 ?"}
    A -- yes --> B{"all l_i[j] = 0 ?<br/>(no leader seen)"}
    B -- yes --> T["write leader_i := random({0,1})"]
    B -- no --> N["do nothing"]
    A -- "no (leader_i = 1)" --> C{"some l_i[j] = 1 ?<br/>(another leader seen)"}
    C -- yes --> T
    C -- no --> N
    T --> S
    N --> S
```

**The four situations in one table:**

| My bit | What I see in the others | Do I flip a coin? |
|---|---|---|
| 0 | nobody is 1 (no leader) | **yes** (we need a leader) |
| 0 | somebody is 1 | no (fine, there is a leader) |
| 1 | somebody else is 1 too | **yes** (too many leaders) |
| 1 | nobody else is 1 | no (fine, I am the only leader) |

**Task LE:** the set of executions in which there exists a **single fixed leader throughout the execution**.

**Safe configuration:**

- for **exactly one** processor, say *pᵢ*: leaderᵢ = 1 and *lᵢ*[*j*] = 0 for every *j* ≠ *i* (it is a leader and has seen that nobody else is);
- for **every other** processor *pⱼ* ≠ *pᵢ*: leaderⱼ = 0 and *lⱼ*[*i*] = 1 (it is not a leader and has seen the leader), as well as *lₖ*[*j*] = 0 for every *k* ≠ *j* (nobody has a stale copy saying *pⱼ* is a leader).

In any fair execution that starts in a safe configuration, *pᵢ* is the single leader, so the execution is in LE. (How can you prove it? Check that no processor's condition in lines 04–05 can ever become true: *pᵢ* only ever reads 0 from the others, and every other processor is 0 and only ever reads 1 from *pᵢ*, so nobody tosses a coin and nobody writes.)

**Additional context (Dolev §2.9).** the book's definition of a safe configuration has only the first two conditions; the lecture adds the third (no stale 1 in any local copy), which makes the "nobody ever writes again" argument immediate.

> **Kid example.** Children in a room each hold a card: "1 = I am the boss" or "0 = I am not". Each child looks at everybody's card. If you hold 0 and nobody holds 1, the room has no boss, so you flip a coin to decide your card. If you hold 1 and someone else also holds 1, there are too many bosses, so you flip a coin again. In every other case you keep your card. The room is happy when exactly one child holds 1 and everybody has seen it.

**Remember:**

- Complete graph, shared memory, one **bit** leaderᵢ per processor; fair coin.
- **Toss** if (leaderᵢ = 0 and **no** 1 seen) or (leaderᵢ = 1 and **another** 1 seen); otherwise do nothing.
- **LE** = a **single fixed leader** throughout.
- **Safe**: exactly one leader with bit 1 that has read only 0s; all others 0 and have read the leader's 1; no stale 1 copies (the lecture's third condition; the book has only the first two).

### 3.2 Lemma 2.6: stabilization within 2^O(n) expected rounds

**In one sentence:** luck needs at most *n* coin results (one per processor) within 2*n* rounds, so *cp* ≥ 1/2ⁿ and the expected time is at most 2*n* · 2ⁿ rounds.

**Lemma 2.6.** The algorithm stabilizes within 2^O(*n*) expected number of rounds. More precisely, using Theorem 2.4 the expected number of rounds is at most **2*n* · 2ⁿ**.

The proof presents a **(1/2ⁿ, 2*n*)-strategy** for luck. Then by Theorem 2.4 the expected number of rounds is at most *r*/*cp* = 2*n* / (1/2ⁿ) = 2*n* · 2ⁿ.

**Luck's strategy.** Whenever some processor *pᵢ* tosses a coin (line 06), luck intervenes:

- if **leaderⱼ = 0 for all *j* ≠ *i***, luck fixes the coin toss to be **1**;
- **otherwise**, luck fixes it to be **0**.

Under **coarse atomicity**, line 06 is atomic: at the end of the step, leaderᵢ already holds the coin's result, and nobody could act in between.

In other words, luck always makes the tossing processor choose the value that moves the system towards "exactly one leader".

```mermaid
flowchart LR
  T["pi tosses a coin"] --> Q["are ALL the other<br/>leader bits 0?"]
  Q -- "yes" --> One["luck says 1:<br/>pi becomes the only leader"]
  Q -- "no" --> Zero["luck says 0:<br/>pi steps aside"]
```

**Observation 1 (no coin toss means already safe).** Within **less than 2*n* successive rounds**, each processor *pᵢ* reads all leader registers and, if necessary, tosses a coin and writes the outcome. So if **no processor tosses a coin within the first 2*n* rounds**, the system reaches a safe configuration: nobody writes during these rounds, so everybody reads fixed values. A processor with leaderᵢ = 0 that did not toss must have found some *pⱼ* with leaderⱼ = 1, and a processor with leaderⱼ = 1 that did not toss must have found that it is the **only** one with value 1.

**Observation 2 (with luck, a leader appears and stays).** If some processor does toss a coin, then by luck's strategy and lines 04–06, after the first coin toss **at least one leader register holds 1**. (If at the moment of the toss all other leader registers are 0, luck gives the tosser 1; otherwise some other register already holds 1.) Moreover, once leaderⱼ = 1 for some *j*, there is a *k* (possibly *k* = *j*) such that **leaderₖ = 1 for the rest of the execution**:

- Let *L* be the set of processors whose leader register holds 1 after the first coin toss.
- If some *pₖ* ∈ *L* never tosses a coin again, then leaderₖ = 1 forever (it only changes by tossing).
- Otherwise every processor in *L* tosses again. Let *pₖ* be the **last** processor in *L* to toss (for the first time). By luck's strategy, at *pₖ*'s toss all other leader values are 0, so luck sets *pₖ*'s coin to **1**. From then on leaderₖ = 1 and leaderⱼ = 0 for every *j* ≠ *k*.

**Success probability.** Each processor *pᵢ* tosses a coin **at most once**:

- If luck set *pᵢ*'s first toss to **0**, then in all later readings *pᵢ* sees leaderₖ = 1 (the eventual leader), so its condition never holds again.
- If luck set *pᵢ*'s first toss to **1**, then all other leader values were 0 at that moment (that is when luck chooses 1). After this atomic step *pᵢ* is the only processor with value 1, so it will not toss again either.

So luck intervenes at most *n* times, each time fixing one fair coin (probability 1/2). The combined probability is at least

```math
cp \ge \left(\tfrac{1}{2}\right)^n = \frac{1}{2^n}.
```

**Putting things together.** Within at most 2*n* rounds every processor stops tossing coins, and during these 2*n* rounds each processor tosses at most once. So luck wins the game within 2*n* rounds with combined probability 1/2ⁿ: a (1/2ⁿ, 2*n*)-strategy. By Theorem 2.4:

```math
\text{expected rounds} \le \frac{r}{cp} = \frac{2n}{1/2^n} = 2n\,2^n \in 2^{O(n)}.
```

*Example:* with *n* = 3 processors, the bound is 2 · 3 · 2³ = 48 expected rounds; with *n* = 10 it is 2 · 10 · 1024 = 20 480. The bound grows exponentially with *n*, which is why the lecture calls this a simple but slow algorithm.

| *n* | *r* = 2*n* | *cp* = 1/2ⁿ | bound *r*/*cp* = 2*n* · 2ⁿ |
|---|---|---|---|
| 3 | 6 | 1/8 | 48 |
| 5 | 10 | 1/32 | 320 |
| 10 | 20 | 1/1024 | 20 480 |

```mermaid
flowchart LR
    A["Any configuration"] -- "luck fixes every coin:<br/>≤ n tosses, ≤ 2n rounds" --> B["Exactly one leader,<br/>everyone has read it (safe)"]
    A -. "real coins disagree with luck:<br/>start a new try" .-> A
```

**Technical clarification.** One sentence of the lecture says "Luck wins the game within 2ⁿ rounds"; the rest of the lecture, the rest of the proof and the book all say **2*n*** rounds, which is what the bound 2*n*·2ⁿ uses.

**A small run with luck, *n* = 3** (bits of *p*₁, *p*₂, *p*₃; start: everybody is 1):

| Step | Who moves | What it sees | Tosses? | Luck fixes | Bits afterwards |
|---|---|---|---|---|---|
| start | — | — | — | — | 1, 1, 1 |
| 1 | *p*₁ | it is 1, others are 1 | yes | not all others 0 ⇒ **0** | 0, 1, 1 |
| 2 | *p*₂ | it is 1, *p*₃ is 1 | yes | not all others 0 ⇒ **0** | 0, 0, 1 |
| 3 | *p*₃ | it is 1, others are 0 | no | — | 0, 0, 1 |
| 4 | *p*₁, *p*₂ | they are 0, *p*₃ is 1 | no | — | 0, 0, 1 |

Luck fixed 2 coins, so this run happens with probability ½ · ½ = 1/4, which is more than the worst case 1/2³ = 1/8.

> **Kid example.** In the boss game, a friendly fairy (luck) secretly decides each coin: "if nobody else is boss, you become boss; otherwise you are not". Each child needs at most one coin flip, so the fairy needs at most *n* lucky flips, and the chance that real coins do exactly what the fairy wants is at least 1/2ⁿ. With 3 children that is 1 in 8, so on average about 8 tries of at most 6 rounds each: 48 rounds.

**Remember:**

- **Lemma 2.6**: expected stabilization **≤ 2*n* · 2ⁿ ∈ 2^O(*n*)** rounds (coarse atomicity).
- Luck's strategy: all others 0 ⇒ fix **1**; otherwise fix **0**.
- Obs. 1: no toss within **2*n*** rounds ⇒ already safe. Obs. 2: after the first toss a 1 exists, and some leaderₖ = 1 **forever**.
- Each processor tosses **at most once** ⇒ ≤ *n* interventions ⇒ **cp ≥ 1/2ⁿ**; *r* = **2*n*** ⇒ (1/2ⁿ, 2*n*)-strategy.
- Examples: *n* = 3 ⇒ 48; *n* = 10 ⇒ 20 480 (exponential, slow).
- The slide's "2ⁿ rounds" is a typo for **2*n***.

### 3.3 Lemma 2.7: no stabilization under fine atomicity

**In one sentence:** if the scheduler can see each coin before it is written, it waits until everyone has the same result and then lets them all write together, forever.

**Lemma 2.7.** The algorithm is **not** self-stabilizing under fine atomicity (where a coin toss is a separate atomic step).

*Proof: a winning strategy for the scheduler.*

1. Start in a configuration where **all leader registers hold 1**. Every processor sees another leader, so each must toss a coin.
2. The scheduler activates one processor to toss. Because the toss is a separate step, the scheduler **sees the result** before it is written. If the result is 1, the scheduler lets this processor toss again, until the result is **0**.
3. The scheduler then **suspends** this processor **before it writes** its 0.
4. It does the same with the next processor, and the next, until **every** processor is about to write 0.
5. Now the scheduler lets all of them write. **All leader registers hold 0.** Every processor sees no leader, so each must toss again.
6. In the same way the scheduler makes every processor hold a pending **1**, then lets them all write: **all registers hold 1**. Back to step 1.

```mermaid
stateDiagram-v2
    All1: All leader registers = 1
    Pending0: Every processor holds a tossed 0, not yet written
    All0: All leader registers = 0
    Pending1: Every processor holds a tossed 1, not yet written
    All1 --> Pending0: scheduler reruns each toss until it shows 0,<br/>then pauses the processor before the write
    Pending0 --> All0: scheduler lets all processors write
    All0 --> Pending1: same trick, waiting for 1s
    Pending1 --> All1: scheduler lets all processors write
```

The system never has exactly one leader, so it never stabilizes. The lecture animates this with five people: one by one each tosses until the coin shows 0 (=0 above each head) while still standing, and then all of them sit down together.

**Additional context (Dolev §2.9).** the book adds that the resulting schedule is fair with probability 1 (every processor keeps being activated), so this is a legitimate fair scheduler, not a cheat.

The lesson: **the exact atomicity assumption decides whether a randomized algorithm works.** Under coarse atomicity the scheduler cannot react to the coin before it is written, so it cannot keep everyone in step.

> **Kid example.** Five children all stand (all are "boss"). Each must flip a coin. The naughty teacher watches each coin land on the table **before** the child acts. If a child gets "stay standing", the teacher says "flip again". When a child finally gets "sit", the teacher says "freeze, don't sit yet". Once all five are frozen with "sit", the teacher says "go!" and all sit together. Then the same trick with "stand". The class is never left with exactly one child standing.

**Remember:**

- **Lemma 2.7**: **not** self-stabilizing under **fine** atomicity.
- Scheduler's trick: see each coin, **re-toss** until it shows the value it wants, **pause before the write**, then release everybody together: all 1 ⇒ all 0 ⇒ all 1 …
- The schedule is still **fair** (with probability 1).
- Lesson: the **atomicity** assumption decides whether a randomized algorithm works.

## 4. Summary

**What this chapter is about:** the whole lecture on one page.

### 4.1 Recap

**In one sentence:** identical processors need luck to choose a leader; coins work when the scheduler cannot peek (coarse atomicity), taking at most 2*n* · 2ⁿ expected rounds, and fail when it can (fine atomicity).

- In **anonymous** complete graphs, leader election needs **symmetry breaking**. A **deterministic** rule ("stand if nobody stands, else sit") works under a **central daemon** but **no deterministic solution exists under a distributed daemon**.
- **Randomized self-stabilization:** from any configuration, under any fair scheduler, a safe configuration is reached within a finite (expected) number of rounds.
- **Coarse atomicity** (toss and write in one step) gives the adversary less power than **fine atomicity** (toss as a separate step).
- **SL-game:** scheduler versus luck. If luck has a **(*cp*, *r*)-strategy**, the algorithm stabilizes within **r/cp** expected rounds (**Theorem 2.4**), because the number of tries is geometric with mean 1/*cp*.
- **Algorithm:** toss a coin if you are 0 and see no leader, or you are 1 and see another leader. Under coarse atomicity luck has a **(1/2ⁿ, 2*n*)-strategy**, giving at most **2*n*·2ⁿ ∈ 2^O(*n*)** expected rounds (**Lemma 2.6**). Under fine atomicity the scheduler can prevent stabilization forever (**Lemma 2.7**).

```mermaid
flowchart LR
  A["anonymous processors:<br/>need symmetry breaking"] --> B["central daemon:<br/>deterministic rule works"]
  A --> C["distributed daemon:<br/>no deterministic rule"] --> D["use coins"]
  D --> E["coarse atomicity:<br/>≤ 2n·2ⁿ expected rounds<br/>(Lemma 2.6)"]
  D --> F["fine atomicity:<br/>scheduler wins forever<br/>(Lemma 2.7)"]
```

> **Kid example.** The whole story: identical twins cannot choose one phone answerer if they always act together. Flipping coins helps, as long as the naughty teacher cannot look at the coins before the twins act. If the teacher can look, the teacher can always keep the twins in step, and they never choose.

## Revision notes and references

Technical clarifications, where the lecture material is corrected or made precise, appear in sections [3.2](#32-lemma-26-stabilization-within-2on-expected-rounds). Where the slides and the course book differ, this guide follows the book and says so at that point.

- S. Dolev, *Self-Stabilization*, MIT Press, 2000: §2.4 and §2.9 (randomized self-stabilization and the scheduler-luck game).
