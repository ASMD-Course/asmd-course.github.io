# Lab07

## Objectives

- get acquainted with CTMCs, their simulation, and meta-models
- play with SPN
- reason about probability, time, and testing

## Tasks

1. **SIMULATOR**: Take the communication channel CTMC example in StochasticChannelSimulation. Compute the average time at which
   communication is done across n runs. Compute the relative amount of time (0% to 100%) that the system is in fail state until
   communication is done across n runs. Extract an API for nicely performing similar checks.

2. **GURU**: Check the SPN module, that incorporates the ability of CTMC modelling on top of Petri Nets, leading to 
    Stochastic Petri Nets. Code and simulate Stochastic Readers & Writers shown in previous lesson. Try to study how key 
    parameters/rate influence average time the system is in read or write state.

3. **RANDOM-UNIT-TESTER**: How do we unit-test with randomness? And how we test at all with randomness? Think about this 
   in general. Try to create a repeatable unit test for Statistics as in utils.StochasticSpec.


### 1. Simulator
For computing the relative amount of time that the system is in fail state until communication is done, 
we can modify the existing simulation code to track the time spent in each state and obtain the relative percentage.

In order to achieve this, I've created a small API that allows us to perform these checks:

```scala
def statesTiming(s0: S, ts: S, rnd: Random): Option[Map[S, Double]] =
    val trace = self.newSimulationTrace(s0, rnd)
    trace.zip(trace.tail)
      .takeWhile:
    case (current, _) => current.state != ts
      .map:
    case (current, next) => (current.state, next.time - current.time)
      .toList match
        case Nil if trace.head.state == ts => Some(Map.empty)
        case Nil => None
        case pairs => Some(pairs.groupMapReduce(_._1)(_._2)(_ + _))
```
State timing function will return a map of states to the total time spent in each state until reaching 
the target state `ts`. It do this by first generating a simulation trace from the initial state `s0`, 
then zipping the trace with its tail to get pairs of consecutive states along with their timestamps and then calculating 
the time spent in each of those `(current.state, next.time - current.time)`
Finally, it groups the time spent by state and sums it up to get the total time spent in each state.

This is used in the following way:

```scala
def meanStatesTiming(s0: S, ts: S, rnd: Random)(n: Int): Option[Map[S, Double]] =
  val residenceMaps = for
    _ <- 1 to n
    resMap <- statesTiming(s0, ts, rnd)
  yield resMap

  if residenceMaps.isEmpty then
    None
  else
    Some:
      residenceMaps
        .flatMap(_.toSeq)
        .groupMapReduce(_._1)(_._2)(_ + _)
        .view
        .mapValues(_ / residenceMaps.size)
        .toMap
```
The `meanStatesTiming` function runs the `statesTiming` function `n` times and collects the resulting maps of state timings.
It then aggregates these maps by summing the times for each state across all runs and finally computes the average time 
spent in each state by dividing the total time by the number of runs.

Finally, since we can compute the mean time spent in each state, we could also provide the percentage of time spent:

```scala
def timingPercentage(s0: S, ts: S, rnd: Random)(n: Int): Option[Map[S, Double]] =
  meanStatesTiming(s0, ts, rnd)(n).map: m =>
    val total = m.values.sum
    m.transform((_, v) => v * 100 / total)
```

For example, using the above API on the StochasticChannel example:

```scala
def stocChannel: CTMC[State] = CTMC.ofTransitions(
    Transition(IDLE,1.0 --> SEND),
    Transition(SEND,100_000.0 --> SEND),
    Transition(SEND,200_000.0 --> DONE),
    Transition(SEND,100_000.0 --> FAIL),
    Transition(FAIL,100_000.0 --> IDLE),
    Transition(DONE,1.0 --> DONE)
  )

print:
  stocChannel.timingPercentage(IDLE, DONE, new Random)(30)

```

This will output something like:
`Some(Map(SEND -> 3.123242120876556E-4, IDLE -> 99.99937798514014, FAIL -> 3.0969064776552335E-4))`
So the system spends 3.0969064776552335E-4 % of the time in FAIL state until communication is done.

### 2. Guru
For this task we start from the Stochastic Readers & Writers example that is modelled as follow:

```scala
val transitions = SPN[Place](
    Trn(MSet(P1), m => 1.0,   MSet(P2),  MSet()),                         // t1
    Trn(MSet(P2), m => 200_000,  MSet(P3),  MSet()),                      // t2
    Trn(MSet(P2), m => 100_000,  MSet(P4),  MSet()),                      // t3
    Trn(MSet(P3, P5), m => 100_000,  MSet(P6, P5),  MSet()),              // t4
    Trn(MSet(P4, P5), m => 100_000,  MSet(P7),  MSet(P6)),                // t5
    Trn(MSet(P6), m => 0.1 * m(P6),  MSet(P1),  MSet() ),                 // t6
    Trn(MSet(P7), m => 0.2,  MSet(P1, P5),  MSet()),                      // t7
  )
```
For analyze the following key parameters/rate influence average time the system is in read or write state, I've created
a small API that allows us to create variations of the original SPN model by modifying the rates of specific transitions.

This framework, that is available in `SPNAnalyzer` allows to define variation of the original SPN model like this:

```scala
val analyzer = SPNAnalyzer[Place](
  Trn(MSet(P1), _ => 1.0, MSet(P2), MSet()),
  Trn(MSet(P2), _ => 200_000, MSet(P3), MSet()).rateValues(prefVariations *),
  Trn(MSet(P2), _ => 100_000, MSet(P4), MSet()).rateValues(prefVariations *),
  Trn(MSet(P3, P5), _ => 100_000, MSet(P6, P5), MSet()),
  Trn(MSet(P4, P5), _ => 100_000, MSet(P7), MSet(P6)),
  Trn(MSet(P6), m => 0.1 * m(P6), MSet(P8), MSet()),
  Trn(MSet(P7), _ => 0.2, MSet(P8, P5), MSet())
)
```

So it tries to emulate the orginal API but allowing new methods to define variations of the rates of specific transitions.
`rateValues`, in particular, takes a sequence of values and creates a variation of the original SPN model for each value,
modifying the rate of the transition.

> Note: It can be noted that, compared to the original model, a new place called P8 has been added. 
>The Petri net that is modeled will no longer carry the transitions from P6 and P7 to P1 but will instead cause the tokens 
>to flow into P8, representing a termination.
>This is due to the fact that, since I have to define an initial marking and a target marking, 
>if I defined them as identical, the system would not perform any simulation.

I will vary the rates of transitions t2 and t3, which represent the rates of process decides to read or write, respectively
and the rates of the velocities of the transitions t6 and t7, in which a process executes its action and releases the resource.

Rates for t2 and t3 -> [100000, 300000]
Rates for t6 and t7 -> [0.02, 0.08]

| t2 Rate | t3 Rate | t6 Rate | t7 Rate | % Time in Read State | % Time in Write State |
|---------|---------|---------|---------|----------------------|-----------------------|
| 100000  | 100000  | 0.02    | 0.02    | 49.32%               | 48.41%                |
| 100000  | 100000  | 0.02    | 0.08    | 69.91%               | 26.65%                |
| 100000  | 100000  | 0.08    | 0.02    | 14.61%               | 82.42%                |
| 100000  | 100000  | 0.08    | 0.08    | 41.81%               | 50.98%                |
| 300000  | 100000  | 0.02    | 0.02    | 33.00%               | 80.22%                |
| 300000  | 100000  | 0.02    | 0.08    | 44.35%               | 49.96%                |
| 300000  | 100000  | 0.08    | 0.02    | 8.76%                | 90.68%                |
| 300000  | 100000  | 0.08    | 0.08    | 20.62%               | 72.58%                |
| 100000  | 300000  | 0.02    | 0.02    | 18.31%               | 35.93%                |
| 100000  | 300000  | 0.02    | 0.08    | 91.02%               | 6.54%                 |
| 100000  | 300000  | 0.08    | 0.02    | 4.56%                | 58.74%                |
| 100000  | 300000  | 0.08    | 0.08    | 77.27%               | 15.44%                |
| 300000  | 300000  | 0.02    | 0.02    | 54.19%               | 43.58%                |
| 300000  | 300000  | 0.02    | 0.08    | 72.02%               | 23.10%                |
| 300000  | 300000  | 0.08    | 0.02    | 12.93%               | 83.74%                |
| 300000  | 300000  | 0.08    | 0.08    | 44.90%               | 46.77%                |


Increasing the write completion rate (t7) strongly reduces the time the system spends in write states.
The t6 rate amplifies this: if t7 is slow, a higher t6 pushes more tokens into writes and increases write bottlenecks; 
if t7 is fast, higher t6 instead helps reads dominate. 
The balance between t2 and t3 decides how much work flows into read vs. write: more flow into writes (high t3) means 
the system is write-heavy unless t7 is large enough to drain it quickly.

### 3. Random-Unit-Tester

Working with randomness, generally, involves dealing with random generators which usually accepts a seed for determine 
the sequence of random values that will be generated.
So, in order to create repeatable unit tests with randomness, we can use a fixed seed for the random generator.

```scala
class StochasticsTest extends AnyFunSuite with Matchers:

  given Random = new Random(42)

  test("statistics should return correct total count"):
    val choices = Set((0.3, "A"), (0.5, "B"), (0.2, "C"))
    val size = 1000
    val result = statistics(choices, size)
    result.values.sum shouldBe size


  test("statistics should only return values from original choices"):

    val choices = Set((0.4, "X"), (0.6, "Y"))
    val size = 500

    val result = statistics(choices, size)
    val expectedValues = choices.map(_._2)

    result.keys.toSet shouldBe expectedValues
  

  test("statistics should approximate expected probabilities"):
    val choices = Set(
      (0.1, "rare"),
      (0.3, "common"),
      (0.6, "frequent")
    )
    val size = 10000
    val result = statistics(choices, size)
    val tolerance = 0.05

    val rareCount = result.getOrElse("rare", 0)
    val commonCount = result.getOrElse("common", 0)
    val frequentCount = result.getOrElse("frequent", 0)

    (rareCount.toDouble / size) should be (0.1 +- tolerance)
    (commonCount.toDouble / size) should be (0.3 +- tolerance)
    (frequentCount.toDouble / size) should be (0.6 +- tolerance)


  test("statistics should handle single choice"):
    val choices = Set((1.0, "only"))
    val size = 50
    val result = statistics(choices, size)
    result shouldBe Map("only" -> 50)


  test("statistics should handle zero size"):
    val choices = Set(
      (0.5, "A"),
      (0.5, "B")
    )
    val size = 0
    val result = statistics(choices, size)
    result shouldBe Map.empty[String, Int]


  test("statistics should work with different data types"):
    case class Item(name: String, value: Int)
    val choices = Set(
      (0.4, Item("sword", 100)),
      (0.6, Item("shield", 80))
    )
    val size = 200

    val result = statistics(choices, size)
    result.values.sum shouldBe size
    result.keys.foreach:
      item =>
        choices.map(_._2) should contain(item)
```

### 4. Probability-LLM
