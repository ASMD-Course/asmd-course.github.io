# Laboratory 11

Code available [here](https://github.com/ASMD-Course/asmd23-public-scafi)

## Objectives
- play with examples
- synthesize new behaviour

## Tasks

1. **Variations to programs**: Slides shows small variations to the programs seen in lesson 09. Those should be implemented and experimented with. 
2. **Partition**: Implement partition with scafi
3. **Channel**; Implement channel with scafi

### 1. Variations to programs

- **Case 9**:
For the following case we can have different variations:

```scala
class Main9 extends AggregateProgramSkeleton:
  override def main() = if sense1 then rep(0){value => if (value == 1000) value else value + 1} else 0
  // or  
  override def main() = mux[Integer](sense1)(rep(0){value => if (value == 1000) value else value + 1})(0)
  // or
  override def main() = branch(sense1)(rep(0){value => if (value == 1000) value else value + 1})(0)

object Demo9 extends Simulation[Main9]
```

- **Case 12**:

```scala
class Main12 extends AggregateProgramSkeleton:
  import Builtins.Bounded.of_i

  override def main() = foldhoodPlus[Set[Int]](Set())(_++_)(Set(nbr(mid())))

object Demo12 extends Simulation[Main12]
```

- **Case 8**:

```scala
class Main8 extends AggregateProgramSkeleton:
  override def main() = minHoodPlus((nbrRange, nbr{mid()}))

object Demo8 extends Simulation[Main8]
```

- **Case 14**:

```scala
class Main14 extends AggregateProgramSkeleton:
  import Builtins.Bounded.of_i

  override def main(): ID = rep(mid()){ curr => curr max maxHoodPlus( nbr{curr}) }

object Demo14 extends Simulation[Main14]
```

- **Case 16**:

```scala
class Main16 extends AggregateProgramSkeleton:
  override def main() = rep(Double.MaxValue):
    d => mux[Double](sense1){0.0}{minHoodPlus(nbr{d} + nbrRange * (if(sense2) 5.0 else 1.0))}

object Demo16 extends Simulation[Main16]
```

### 2. Implementation of partition

```scala

class PartitionExercise extends AggregateProgramSkeleton:
  import Builtins.Bounded.*
  def partition[T](where: => Boolean)(label: Int = mid()): Int =
    rep((Double.MaxValue, label)):
      (d, id) =>
        mux[(Double, Int)](where) {(0.0, label)} {minHoodPlus((nbr {d} + nbrRange(), nbr {id}))}
    ._2

  override def main() = partition(where = sense1)(label = mid())
 
 object PartitionExecution extends Simulation[PartitionExercise]
```


### 3. Implementation of channel

```scala
class ChannelExercise extends AggregateProgramSkeleton:
  import Builtins.Bounded.*
  import Builtins.Bounded

  def gradient(condition: => Boolean): Double = rep(Double.MaxValue):
    d => mux[Double](condition){ 0.0 }{ minHoodPlus(nbr{d} + nbrRange) }

  def broadcast[T: Bounded](src: => Boolean, value: T): T =
    rep((Double.MaxValue, value)):
      (d, v) =>mux[(Double, T)](src){(0.0, value)}{minHoodPlus((nbr{d} + nbrRange, nbr{v}))}
    ._2

  def distance(src: => Boolean, dst: => Boolean): Double = broadcast(src, gradient(dst))

  def dilate(region: => Boolean, width: Double): Boolean = gradient(region) < width

  def channel(src: => Boolean, dst: => Boolean)(width: Double) =
    dilate(gradient(src) + gradient(dst) <= distance(src, dst), width)

  override def main() = channel(src = sense1, dst = sense2)(width = 100.0)

object ChannelExecution extends Simulation[ChannelExercise]
```