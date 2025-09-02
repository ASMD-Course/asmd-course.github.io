# Lab08

## Objectives

- get acquainted with using multiple simulation to analyse CTMCs
- get acquainted with PRISM model checker
- play with SPN in PRISM
- play with DAP model presented in lesson 07


## Tasks
1. **PRISM** : Make the stochastic Readers & Writers Petri Net seen in lesson work: perform experiments to investigate the probability
   that something good happens within a bound. Play with PRISM configuration to inspect steady-state proabilities of reading and writing (may need to play with options
   anche choose “linear equations method”)

2. **PRISM-VS-SCALA**: Take the communication channel example, and perform comparison of results between PRISM and our Scala approach
   Write Scala support for performing additional experiments and comparisons (e.g., G formulas, steady-state computations)

3. **LLM-STOCHASTIC-ANALYSIS**: PRISM is rather well known, and perhaps LLMs know it. Can LLMs understand the meaning of a stochastic property?
   Can they solve (very) simple modelchecking? Can they preview what a simulation can produce?

### 1. Prism

Updted the original reader and writers model adding a new constant T for indicating the time bound for the experiments.

```text
ctmc

const int N = 20;
const double T;

module RW
p1 : [0..N] init N;
p2 : [0..N] init 0;
p3 : [0..N] init 0;
p4 : [0..N] init 0;
p5 : [0..N] init 1;
p6 : [0..N] init 0;
p7 : [0..N] init 0;

[t1] p1>0 & p2<N  -> 1 : (p1'=p1-1)&(p2'=p2+1);
[t2] p2>0 & p3<N ->  200000 : (p2'=p2-1) & (p3'=p3+1);
[t3] p2>0 & p4<N -> 100000 : (p2'=p2-1) & (p4'=p4+1);
[t4] p3>0 & p5>0 & p6<N -> 100000 : (p3'=p3-1) & (p6'=p6+1);
[t5] p4>0 & p5>0 & p6=0 & p7<N -> 100000 : (p4'=p4-1) & (p5'=p5-1) & (p7'=p7+1);
[t6] p6>0 & p1<N -> p6*1 : (p6'=p6-1) & (p1'=p1+1);
[t7] p7>0 & p5<N & p1<N -> 0.5 : (p7'=p7-1) & (p1'=p1+1) & (p5'=p5+1);

endmodule
```

*Experiments*:

1. Test the probability that the critical section is accessed within T time units. 

```text
P=? [ F<=T p6>0 | p7>0 ]

Defined Constants:
    N = 20, T = 10

Method:
    Verification

Results Probability:
    0.940070578
```

2. Test the probability that the critical section is never violated within T time units.

```text
P=? [ G<=T !(p6>0 & p7>0) ]

Defined Constants:
    N = 20, T = 10

Method:
    Verification

Results Probability:
    1.0
```


3. Test liveness inside 
```text
P=? [ G<=T ((p3+p4) <= p1)]

Defined Constants:
    N = 20, T = 20

Method:
    Verification

Results Probability:
    0.8648083404140592
```

4. Steady-state probability of being in writing state

```text
S=? [ p7>0 ]

Defined Constants:
    N = 30, T=20

Method:
    Verification

Result Probability:
    0.655203298
```

5. Steady-state probability

```text
S=? [ p6>0 ]

Defined Constants:
    N = 30, T=20

Method:
    Verification

Result Probability:
    0.3147706
```


### 2. PRISM-VS-SCALA
I've create an extension of the CTMC experiment framework to support more operations on properties and for model checking.
In particulare I've added support for more temporal operators other than eventually and, also, I've defined a small
DSL for joining properties together. The implementation is in `scala.lab.u08.CTMCExtensions`.

```scala
  extension [S](self: CTMC[S])
    /**
     * G operator expressed as G x= not F not x
     *
     * @param filter the filter to apply
     * @tparam A
     * @return
     */
    def always[A](filter: A => Boolean): Property[A] = ! self.eventually[A](p => !filter(p))

    /**
     * Never operator
     *
     * @param filter
     * @tparam A
     * @return
     */
    def never[A](filter: A => Boolean): Property[A] =  self.always(p => !filter(p))

    /**
     * Now operator, which checks the property on the first element of the trace
     *
     * @param pred the predicate to check
     * @tparam A
     * @return
     */
    def now[A](pred: A => Boolean): Property[A] = tr => tr.headOption.exists(ev => pred(ev.state))
```

With this support I've been able to express the same properties as in PRISM and compare the results.

```scala
val reachDone: Property[State] = stocChannel.eventually(_ == DONE)
val probReachDone = stocChannel.experiment(
  runs = 1000,
  prop = reachDone,
  s0 = IDLE,
  timeBound = 5.0
)
println(s"Probability of eventually reaching DONE from IDLE within 10 time units: $probReachDone")
```
Can be translated to PRISM as:
```text
P=? [ F<=5 "DONE" ]
```
Result in Prism: 0.928936
Result in Scala: 0.969

```scala
val isFail: Property[State] = stocChannel.now(_ == FAIL)
val isDone: Property[State] = stocChannel.now(_ == DONE)
val neverReachDoneUntilFail: Property[State] = !isDone U isFail
val probReachFail = stocChannel.experiment(
  runs = 1000,
  prop = neverReachDoneUntilFail,
  s0 = IDLE,
  timeBound = 10.0
)
println(s"Probability of not reaching DONE until reaching FAIL from IDLE within 10 time units: $probReachFail")
```

Can be translated to PRISM as:
```text
P=? [ !("DONE") U<=10 "FAIL" ]
```

Result in Prism: 0.317
Result in Scala: 0.335

```scala
val alwaysNotFail: Property[State] = stocChannel.never(_ == FAIL)
val probNotFail = stocChannel.experiment(
  runs = 1000,
  prop = alwaysNotFail,
  s0 = IDLE,
  timeBound = 10.0
)
println(s"Probability of never reaching FAIL from IDLE within 10 time units: $probNotFail")
```

Can be translated to PRISM as:
```text
P=? [ G<=10 !("FAIL") ]
```
Result in Prism: 0.682
Result in Scala: 0.675

```scala
 val stayInSend: Property[State] = stocChannel.always(_ == SEND)
val probNotFail = stocChannel.experiment(
  runs = 1000,
  prop = stayInSend,
  s0 = IDLE,
  timeBound = 5.0
)
println(s"Probability of always staying in SEND from IDLE within 5 time units: $probNotFail")
```

Can be translated to PRISM as:
```text
P=? [ G<=5 ("SEND") ]
```
Result in Prism: 0.0
Result in Scala: 0.0


### 3. LLM-STOCHASTIC-ANALYSIS
For experimenting the capacity of LLMs to perform task similar to the one we did with PRISM I'll use GPT-5 loading 
the prism model for reader and writers.
We'll ask then to perform some properties checking trying to infer the results.

First we load the model and try to make it understand what it represents:

```text
I'll give to you this prism model to analyze:

ctmc

const int N = 20;

module RW
p1 : [0..N] init N;
p2 : [0..N] init 0;
p3 : [0..N] init 0;
p4 : [0..N] init 0;
p5 : [0..N] init 1;
p6 : [0..N] init 0;
p7 : [0..N] init 0;

[t1] p1>0 & p2<N  -> 1 : (p1'=p1-1)&(p2'=p2+1);
[t2] p2>0 & p3<N ->  200000 : (p2'=p2-1) & (p3'=p3+1);
[t3] p2>0 & p4<N -> 100000 : (p2'=p2-1) & (p4'=p4+1);
[t4] p3>0 & p5>0 & p6<N -> 100000 : (p3'=p3-1) & (p6'=p6+1);
[t5] p4>0 & p5>0 & p6=0 & p7<N -> 100000 : (p4'=p4-1) & (p5'=p5-1) & (p7'=p7+1);
[t6] p6>0 & p1<N -> p6*1 : (p6'=p6-1) & (p1'=p1+1);
[t7] p7>0 & p5<N & p1<N -> 0.5 : (p7'=p7-1) & (p1'=p1+1) & (p5'=p5+1);

endmodule
```
As response the model correctly identifies the model as a representation of the readers and writers problem.

```text
Yep — I can read it. This is a PRISM CTMC for the classic Readers–Writers pattern, modeled as a closed population of 
N clients (here N=20) circulating through “idle → request → read/write → finish → idle”, with a single writer lock.
```
It's important to note that the memory of the model was cleared before loading the model, 
so it's not biased by previous knowledge.

Now we ask to perform the same verifications we did in task 1 and we compare the results:
The parameters used are the same both for GPT-5 and PRISM.


- `P=? [ F<=T p6>0 | p7>0 ]`
  - GPT-5: 0.99995460.
  - PRISM: 0.940070578

- `P=? [ G<=T !(p6>0 & p7>0) ]`
  - GPT-5: 1.0
  - PRISM: 1.0

- `P=? [ G<=T ((p3+p4) <= p1)]`
  - GPT-5: 0.865
  - PRISM: 0.86480

- `S=? [ p7>0 ]`
  - GPT-5: 0.667
  - PRISM: 0.655203298

- `S=? [ p6>0 ]`
  - GPT-5: 0.364
  - PRISM: 0.3147706

The results are very close to the ones obtained with PRISM, showing that LLMs can be used to perform basic stochastic analysis 
on CTMC models.
The problem is that these calculations takess a lot of time(~ 2 to 10 mins), and the model often fails to provide an answer within the time limit 
imposed by the free plans.

[Link to the full conversation with GPT-5](https://chatgpt.com/share/e/68b62801-fbf0-8004-a4a4-96d7d88aebda)



