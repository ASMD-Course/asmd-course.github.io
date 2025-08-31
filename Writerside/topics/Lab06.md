# Lab06

## Objectives

- get acquainted with our modelling stack, possibly improve it
- play with Petri Nets 
- design new models

## Tasks
1. **VERIFIER**: Code and do some analysis on the Readers & Writers Petri Net. Add a test to check that in no path long 
at most 100 states mutual exclusion fails (no more than 1 writer, and no readers and writers together). Can you extract 
a small API for representing safety properties? What other properties can be extracted? How the boundness assumption 
can help?

2. **DESIGNER**: Code and do some analysis on a variation of the Readers & Writers Petri Net: it should be the minimal 
variation you can think of, such that if a process says it wants to read, it eventually (surely) does so. 
How would you show evidence that your design is right? What about a variation where at most two process can write?

3. **ARTIST**: Create a variation/extension of PetriNet meta-model, with priorities: each transition is given a numerical 
priority, and no transition can fire if one with higher priority can fire. Show an example that your pretty 
new “abstraction” works as expected. Another interesting extension is “coloring”: tokens have a value attached, 
and this is read/updated by transitions.

4. **PETRINET-LLM**: We know that LLMs/ChatGPT can arguably help in write/improve/complete/implement/reverse-engineer 
standard ProgLangs. But is it of help in designing Petri Nets? Does it truly “understand” the model? Does it understand 
our DSL by examples?

### 1. Verifier

Using the framework showed in the course for modelling PetriNets we can actually code the Readers & Writers like this:

```scala
object PNReadersWriters:
  export u06.utils.MSet

  enum Place:
    case P1,P2,P3,P4,P5,P6,P7

  export Place.*
  export u06.modelling.PetriNet.*
  export u06.modelling.SystemAnalysis.*

  def readersWriters = PetriNet[Place](
    // t1
    MSet(P1) ~~> MSet(P2),

    // t2, t3
    MSet(P2) ~~> MSet(P3), MSet(P2) ~~> MSet(P4),

    // t4
    MSet(P3, P5) ~~> MSet(P6, P5),

    // t5
    MSet(P4, P5) ~~> MSet(P7) ^^^ MSet(P6),

    // t6
    MSet(P6) ~~> MSet(P1),

    // t7
    MSet(P7) ~~> MSet(P5, P1)
  ).toSystem
```
For testing mutual exclusion, and in general safety properties, I've defined an API for testing system properties.
In `SystemPropertyAnalysis`, a general property could be represented as `SystemProperty[S]` that is a function from a 
state to a boolean.
Those could be combined using both logical operators `and`, `or` and `not` and temporal operators `invariant`, `eventually` 
and `never`. Those works with a length bound, so we can check some properties on bounded paths.

Once we have this API we can define property checks using ScalaCheck framework, for example mutual exclusion can be defined as:

```scala
      given Arbitrary[Marking[Place]] =
        Arbitrary:
            for
              n <- Gen.choose(5, 10)
            yield
            MSet.ofList(List.fill(n)(P1) ++ List(P5))

    property("Mutual exclusion for path") = forAll: (initial: Marking[Place]) =>
        val noReadersWritersTogether: SystemProperty[Marking[Place]] = m => !(m(P6) > 0 && m(P7) > 0)
        val atMostOneWriter: SystemProperty[Marking[Place]] = m => m(P7) <= 1
    
        system.invariant(initial, depth)(noReadersWritersTogether and atMostOneWriter)
```

> Note: This method became very slow if we define a very huge depth (>20). This is caused by an explosion of number of paths
> to check. A possible optimization could be to use some memoization technique to avoid checking the same state multiple times.

Generally we can test any property that is a function from a state to a boolean. So other than safety properties you
can add liveness, fairness, etc...
The fact that we're working with bounded paths allows us to reason about the system in a finite (extensional) way, making it easier to
prove properties like mutual exclusion, absence of deadlocks, and other safety properties.

### 2. Designer
For the following task I've designed a variation of the Readers & Writers Petri Net such that if a process says it wants to read,
it eventually (surely) does so. The idea is to add a new place, called `P3H` (High Priority),
that is used to give priority to a process that wants to read.

![Diagram of Readers & Writers with High Priority](PNM.png)

From `P3` a process can age to `P3H` using transition `t4`, when a process is in `P3H` then it will block
,with inhibitor arc, both transition `T7` (normal reader transition) and `T7` (writer transition). So, if the entry
flag is set (`P5 > 0`) then the high priority reader will be able to read or will eventually wait until all conditions 
are met.

For showing evidence that the design is right we can use the same property API defined before to check if the 
property holds. First we need to define the new Petri Net:

```scala
  def readersWritersWithPriority = PetriNet[Place](
    MSet(P1) ~~> MSet(P2),
  
    MSet(P2) ~~> MSet(P3),
    
    MSet(P2) ~~> MSet(P4),
    
    MSet(P3) ~~> MSet(P3H),
    
    MSet(P3H, P5) ~~> MSet(P6, P5),
    
    MSet(P3, P5) ~~> MSet(P6, P5) ^^^ MSet(P3H),
    
    MSet(P4, P5) ~~> MSet(P7) ^^^ MSet(P6, P3H),
    
    MSet(P6) ~~> MSet(P1),
    
    MSet(P7) ~~> MSet(P5, P1)
  ).toSystem
```

And then we can define the property that checks if a process that wants to read will eventually read:

```scala
  property("High priority readers go before writers") = forAll: (initial: Marking[Place]) =>
    val highPriorityWaiting: SystemProperty[Marking[Place]] = m => m(P3H) > 0 && m(P5) == 1
    val highPriorityCanGo: SystemProperty[Marking[Place]] = m => system.next(m).exists(mn => mn(P6) > m(P6))

    system.invariant(initial, depth)(highPriorityWaiting implies highPriorityCanGo)

  property("Readers can pass to high priority") = forAll: (initial: Marking[Place]) =>
    val hasNormalReader: SystemProperty[Marking[Place]] = m => m(P3) > 0
    val canAge: SystemProperty[Marking[Place]] = m => system.next(m).exists(mn => mn(P3H) > m(P3H))

    system.invariant(initial, depth)(hasNormalReader implies canAge)

  property("Reader will eventually read") = forAll: (initial: Marking[Place]) =>
    val readerRequest: SystemProperty[Marking[Place]] = m => m(P3) > 0 || m(P3H) > 0
    val willBeServed: SystemProperty[Marking[Place]] = m => system.eventually(m, depth)(m2 => m2(P6) > 0)
    val readerServiceGuarantee: SystemProperty[Marking[Place]] = readerRequest implies willBeServed

    system.invariant(initial, depth)(readerServiceGuarantee)
```
For the second part of the task, a variation where at most two process can write, we can simply add a new place `P8` that
will maintain the number of writers in the system. Transition `T3` is fireable only when `P8 > 0` and when a writer finishes  (`T9`) 
we return a token to `P8`.

![Diagram of Readers & Writers with max two writers](PNM2.png)

```scala
def readersAndMaxTwoWriters = PetriNet[Place](

  MSet(P1) ~~> MSet(P2),

  MSet(P2) ~~> MSet(P3),

  MSet(P8, P2) ~~> MSet(P4),

  MSet(P3, P5) ~~> MSet(P6, P5),

  MSet(P4, P5) ~~> MSet(P7) ^^^ MSet(P6),

  MSet(P6) ~~> MSet(P1),

  MSet(P7) ~~> MSet(P5, P1, P8)
).toSystem
```
We check the property using the same API defined before:

```scala
  property("No more than two writers are present") = forAll: (initial: Marking[Place]) =>
    val atMostTwoWriters: SystemProperty[Marking[Place]] = m => m(P4) <= 2
    system.invariant(initial, depth)(atMostTwoWriters)
```


### 3. Artist
For this task I've designed an externsion of the Petri Net meta-model adding priorities to transitions. In order to 
do this I've first modified the `Trn` case class adding a new field called `priority`:

```scala
type Marking[P] = MSet[P]
case class Trn[P](cond: MSet[P], eff: MSet[P], inh: MSet[P], priority: Marking[P] => Int = (p:Marking[P]) => 0)
```
`priority` is a function from a marking to an integer, this allows us to define dynamic priorities that can change
based on the current marking of the system. The default priority is 0, so if not specified the transition will have no priority.
It's possible to define priorities with a new operator `withPriority` that is added to the current DSL.

```scala
extension [P](self: Trn[P])
    infix def ^^^(z: Marking[P]): Trn[P] = self.copy(inh = z)
    infix def withPriority(fn: Marking[P] => Int): Trn[P] = self.copy(priority = fn)
    infix def withPriority(p: Int): Trn[P] = self.copy(priority = _ => p)
```
So we can model a petri Net like this:

```scala
def adaptivePetriNet = PriorityPetriNet[Place](
  MSet(P1) ~~> MSet(P2),

  (MSet(P2) ~~> MSet(P3)).withPriority(m=> if m(P3) > 3 then 10 else 1),

  (MSet(P2) ~~> MSet(P4)).withPriority(m =>  if m(P4) > 2 then 0 else 5),

  (MSet(P3, P5) ~~> MSet(P6, P5)).withPriority(m => if m(P4) > 0 then 10 else 0)
  ,

  ((MSet(P4, P5) ~~> MSet(P7)) ^^^ MSet(P6)).withPriority: m =>
    val totalWaiting = m(P3) + m(P4)
    if totalWaiting > 5 then 0
    else if m(P3) > 0 then 20
    else 5,

  (MSet(P6) ~~> MSet(P1)) withPriority 10,
  (MSet(P7) ~~> MSet(P5, P1)) withPriority 10
).toSystem
```
### 4. PetriNet-LLM 
For this task I've used ChatGPT-4. As in the previous labs, It's important first to define a clear context for the model
in order to get a good result. Since we're working on a Meta-Model that handles rewrite systems, First I've upload single
files and asked to explain the code.

```text
Given the following code written in scala 3:

// The definition of a Rewrite System, as a function: S => Set[S]
trait System[S]:
  def next(a: S): Set[S]

// Our factory of Systems
object System:

  // The most general case, an intensional one
  def ofPartialFunction[S](f: PartialFunction[S, Set[S]]): System[S] = s =>
    f.applyOrElse(s, _ => Set[S]())

  // Extensional specification
  def ofRelation[S](rel: Set[(S, S)]): System[S] = ofPartialFunction: s =>
    rel collect:
      case (s, s2) => s2

  // Extensional with varargs
  def ofTransitions[S](rel: (S, S)*): System[S] =
    ofRelation(rel.toSet)

Could you explain me what it represents?
```

**ChatGPT**: _This Scala 3 code defines a flexible and composable framework for modeling rewrite systems or transition systems, 
which are abstract models for describing how elements of a set evolve over time through defined rules or transitions. 
Here’s a detailed breakdown:..._

I've done this for all the files in the `u06.modelling` package, so I had a good understanding of the code.
Then When I had to design the PetriNets in other tasks, I've asked ChatGPT to explain what I've tried to do, and
I've compared the results with my understanding. 

For example, for the Readers & Writers Petri Net I've asked:

```text
Could you explain me what the following Petri Net represents?

 PetriNet[Place](
    // t1
    MSet(P1) ~~> MSet(P2),

    // t2, t3
    MSet(P2) ~~> MSet(P3), MSet(P2) ~~> MSet(P4),

    // t4
    MSet(P3, P5) ~~> MSet(P6, P5),

    // t5
    MSet(P4, P5) ~~> MSet(P7) ^^^ MSet(P6),

    // t6
    MSet(P6) ~~> MSet(P1),

    // t7
    MSet(P7) ~~> MSet(P5, P1)
).toSystem
```

**ChatGPT**: _This Petri Net models a cyclic concurrent system with choices, synchronization, and inhibition, 
using a set of transitions over places P1 to P7. Let's walk through its structure and behavior in detail._

The model does not explicitly define the semantics of each place, it needed some more specific prompts to understand 
that it was the Readers & Writers problem.
just only specifying what kind of system it was, I've got a very good explanation of the model:

|Place| Description | 
|---|-------------|
|  P1 |      Idle or ready       | 
|P2|     Begin access decision        | 
| P3  |   Read section (reader active)          | 
| P4  |   Write section (writer active)          |
| P5  |   Resource available (read/write lock)          |
| P6  |   Reader completed          |
| P7  |   Writer completed          |


| Transition  | Description | 
|-------------|-------------|
| t1: P1 → P2 | Start accessing the shared resource |
| t2: P2 → P3 | Choose to read |
| t3: P2 → P4 | Choose to write |
| t4: P3 + P5 → P6 + P5 | Reader uses resource (non-exclusive), but doesn't consume it |
| t5: P4 + P5 → P7 (inhibited by P6) | Writer uses resource, only if no readers are active |
| t6: P6 → P1 | Reader done, returns to idle |
| t7: P7 → P5 + P1 | Writer done, restores resource and goes idle |

This shows that LLMs can be of help in understanding and explaining Petri Nets, especially when provided with
clear context and specific prompts. However, the accuracy and depth of understanding may vary based on the
complexity of the model and the clarity of the prompts given.