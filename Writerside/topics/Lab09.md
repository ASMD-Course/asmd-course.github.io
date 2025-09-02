# Lab09

Start typing here...


## Objectives
 
- play with basic RL and Q-learning
- play with Deep Q-Learning
- play with MARL

## Tasks

1. **BASIC-Q-LEARNING**
    - Get acquainted with the basic tool of Q-learning, focussing on examples/TryQLearningMatrix
    - Check how variation of key parameters (, γ, α, episode length) affects learning
    - Check how learning gets more difficult as the grid size increases

2. **DESIGN-BY-Q-LEARNING**
    - changing environment, state, rewards (and adding jumps, holes, items, enemies, walls, moving obstacles)
      you can really make your “robot” learn virtually anything, e.g.:
      - define an environment as a sort of corridor with obstacles: your robot has to “zigzag walk” to avoid them
      - “program by learning” a robot to collect items one at a time and go back to initial position
      - “program by learning” a robot to move obstacles to hide from enemies


### 1. Basic Q-Learning
For this task I've analyzed how the variation of key parameters (epsilon, gamma, alpha, episode length), other then 
the grid size, affects the learning process. As did in Lab07, I've developed a small API for generating different environments and
tracking the results of these based on the parameters used.

Each experiment is represented by an initial `Configuration` and will produce a `Result`. We can define them as follows:

```scala
type Configuration
type Result = (Configuration, Q)

extension (configuration: Configuration)
    def toLearningProcess: LearningProcess
```

Similar to ScalaCheck, I've defined a trait called `Generator` which will be used to generate data for the configuration
passed to the experiments. 

```scala
  trait Generator[A] extends (() => LazyList[A]):

    def map[B](f: A => B): Generator[B] = () => this().map(f)

    def flatMap[B](f: A => Generator[B]): Generator[B] = () => this().flatMap(a => f(a)())

  object Generator:

    def fromRange(start: Double, end: Double, step: Double): Generator[Double] =
      () => LazyList
        .iterate(start)(_ + step)
        .takeWhile(_ <= end)

    def fixed[A](elements: List[A]): Generator[A] = () => LazyList.from(elements)

type ConfigurationGenerator = Generator[Configuration]
```
We can define a Generator like a function that produces a LazyList of values of type A and, also, it can be used in a 
monadic way.

`LearningProcess` was also extended in order to be able to track the results of the experiments:

```scala
  extension (lp: LearningProcess)
    def learnWithDisplay(episodes: Int, episodeLength: Int, qf: Q, displayInterval: Int, show: Q => String): Q =

      @tailrec
      def learnRec(e: Int, currentQ: Q): Q =
        if e != episodes && e % displayInterval == 0 then
          println(s"Episode ${episodes - e}:")
          println(show(currentQ))

        e match
          case 0 => currentQ
          case _ =>
            val newQ = lp.runSingleEpisode((lp.system.initial, currentQ), episodeLength)._2
            learnRec(e - 1, newQ)

      println("Episode 0:")
      println(show(qf))
      learnRec(episodes, qf)
```

`learnWithDisplay` will run the learning episodes and display, with the show function passed as parameter,
the evolution of the Q-function every `displayInterval` episodes.
Finally, for running the various experiments we can define the generator like this:

```scala
 val configGen: ConfigurationGenerator =
    for
      s <- Generator.fixed(List(5, 7, 10))
      g <- Generator.fromRange(0.1, 0.9, 0.2)
      a <- Generator.fromRange(0.1, 0.9, 0.2)
      e <- Generator.fixed(List(0.0, 0.5, 0.78, 0.99))
    yield {
      Facade(
        width = s,
        height = s,
        initial = (0, 0),
        terminal = { case _ => false },
        reward = { case ((1,0),DOWN) => 10; case ((3,0),DOWN) => 5; case _ => 0},
        jumps = { case ((1,0),DOWN) => (1,4); case ((3,0),DOWN) => (3,2) },
        gamma = g,
        alpha = a,
        epsilon = e,
        v0 = 1
      )
    }
```

and running them using this function:

```scala
def analysis(
                generator: ConfigurationGenerator,
                show: Result => String
              )(episodes: Int, length: Int, qf: Q, interval: Int = 2500): Unit =
    generator().foreach:
      config =>
        println(f"\n${"=" * 60}")
        println(s"Configuration: $config")
        println(f"${"=" * 60}")


        val lp = config.toLearningProcess
        lp.learnWithDisplay(
          episodes,
          length,
          qf.copy(),
          interval,
          q => show(config, q)
        )
        
  def showGrid(config: Configuration, q: Q): String =
    config match
      case f: Facade =>
        f.show(q.vFunction, "%5.2f")
        f.show(s => q.bestPolicy(s).toString, "%7s")

  val base = QFunction(Move.values.toSet, 1.0, _ => false)
  analysis(configGen, showGrid)(10_000, 100, base)
```
With lower values of α and γ, the learning process tends to be slower. A low α (learning rate) means the agent updates 
its Q-values only slightly—new information changes estimates slowly rather than making it “forget” past learning—while a
low γ (discount factor) reduces the weight of future rewards, encouraging short-sighted behavior.

ε (epsilon) has a more nuanced effect. With a low ε, the agent exploits more than it explores, which can cause it to get
stuck in a local optimum. With a higher ε, the agent explores more, but it may take longer to converge to the optimal 
policy.

In general, the learning process becomes slower as the grid size increases, since the state space is larger and more
complex.

### 2. Design by Q-Learning
For the following task I've created an extension of the starting Environment where the agent has to collect items and 
avoid obstacles (walls) in order to reach the goal.
An agent spawn from a starting position and explore the environment in order to collect all the items and then return
to the starting position.
Whenever the agent collects an item, it receives a reward of +100 and this is then removed from the environment. If the 
agent return to the starting position without having collected all the items, it receives a penalty of -50.
Every time a wall is hit the agent receives a penalty of -1.5.

The environment used in the example is the following:

```text
A _ _ # _ _ _
_ # _ # _ * _
_ # * _ _ _ _
_ # _ # # # _
_ _ _ _ _ _ _
_ _ _ _ * _ _
_ _ _ _ _ _ _
```

- `A` is the starting position of the agent
- `#` are walls
- `*` are items to collect

I've experimented with different values of the parameters and the best results were obtained with:
- `gamma = 0.6`,
- `alpha = 0.5`,
- `epsilon = 0.5`,

The learning process was run for 10,000 episodes with a maximum length of 100 steps each.

The results obtained are shown below:

```text
Initial Q-Function:
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00
1,00	1,00	1,00	1,00	1,00	1,00	1,00

Final Q-Function:
1250,00	1250,00	750,00	1,00	1,02	1,62	6,78
1250,00	1,00	450,00	1,00	56,52	33,20	16,16
750,00	1,00	270,00	162,00	97,20	58,32	34,36
450,00	1,00	162,00	1,00	1,00	1,00	17,31
270,00	162,00	97,20	58,32	34,99	21,00	6,58
162,00	97,20	58,32	34,99	21,00	12,60	3,92
97,20	58,32	34,99	21,00	12,60	3,17	0,31

Move Policy:
<	      <	      <	      <	      >	      v	      v
^	      <	      ^	      <	      v	      v	      <
^	      <	      ^	      <	      <	      <	      <
^	      <	      ^	      <	      <	      <	      ^
^	      <	      <	      <	      <	      <	      <
^	      <	      <	      ^	      ^	      <	      <
^	      <	      ^	      ^	      <	      ^	      ^
```

It's possible to see how well the agent has learned to navigate inside the environment, in particular to avoid walls in 
order to collect all the items before returning to the starting position.