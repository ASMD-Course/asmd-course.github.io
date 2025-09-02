# Laboratory 04

Code available [here](https://github.com/ASMD-Course/asmd23-public-04-adv-prg-patterns)

## Objectives

- be operative with ScalaCheck/Test
- play with ADTs and ScalaCheck
- play with monads
- engineer with monads

## Tasks

1. **TEST-OPERATE**
Download the repo and check everything works as expected. Play just a bit with ScalaCheck and see which parameters it has (e.g. number of
generated tests?). Play also with ScalaTest, and see if it can perform parameterized tests. What are key differences between the two?

2. **ADT-VERIFIER**
Define an ADT for sequences with some operations: map/filer/flatMap/fold/reduce/.... Turn into a Scala trait, and define ScalaCheck
properties capturing axioms 1-to-1. Develop two implementations (Cons/Nil and by Scala List). Engineer tests such that you can easily show they
both satisfy those properties.

3. **MONAD-VERIFIER**
Define ScalaCheck properties for Monad axioms, and prove that some of the monads given during the lesson actually satisfy them. Derive a
general methodology to structure those tests.

4. **ADVANCED-FP-LLM**
LLMs/ChatGPT can arguably help in write/improve/complete/implement/reverse-engineer ADT specifications, ADTs in Scala, and monadic
specifications. Check if/whether this is the case.


### 1. Test - Operate
> Note: The first part of the task is not required, but was carried out (i) to practice designing an ADT and (2) to work 
> with other examples of ScalaCheck and ScalaTest beyond those already provided in the repository

For the following task I've decided to try to define a basic model of genetic sequences using Scala in order to understand
how to define a basic ADT and how to test it using ScalaTest and ScalaCheck.
The model includes two types of nucleotides: DNA and RNA, each represented by a case class. The `GeneticSequence` type is
defined as a generic type that can hold either DNA or RNA nucleotides.


```text
type :
    GeneticSequence[A <: Nucleotide] =
    
constructors :
    dna: List[DNANucleotide] => GeneticSequence[DNANucleotide]
    rna: List[RNANucleotide] => GeneticSequence[RNANucleotide]

operations :
    elements: List[A]
    length: Int
    add: A => GeneticSequence[A]
    chain: GeneticSequence[A] => GeneticSequence[A]

axioms :
    elements(dna(xs)) = xs
    elements(rna(xs)) = xs
    elements(rna()) = E
    elements(dna()) = E
    
    length(dna(xs)) = xs.length
    length(rna(xs)) = xs.length
    
    add(x: Dn)(dna(xs)) = dna(xs :+ x)
    add(x)(rna(xs)) = rna(xs :+ x)
   
    chain(dna(xs))(dna(ys)) = dna(xs ++ ys)
    chain(rna(xs))(rna(ys)) = rna(xs ++ ys)>
```

Note that the axioms operation for adding a nucleotide should always return a sequence of the same type, 
i.e. adding a DNA nucleotide to an RNA sequence should not be allowed.

For this exercise was more convenient to define traits for the operations, first and then structure some tests
using ScalaTest and ScalaCheck.
I've started by defining the nucleotide types

```scala
trait Nucleotide:
  def complement: Nucleotide
```

Then I've implemented them as Enums (complete implementation at `scala/u04/task1/nucleotide`):

```scala
enum DNANucleotide extends Nucleotide:
  // implementation ...
enum RNANucleotide extends Nucleotide:
  // implementation ...
```

Then I've defined the `GeneticSequence` trait and its operations (complete implementation at `scala/u04/task1`) and 
also created a compnion object to act as a factory for creating DNA and RNA sequences.

```scala
trait GeneticSequence[A <: Nucleotide]:
  /**
   * Returns the elements of the genetic sequence.
   */
  def elements: List[A]

  /**
   * Returns the length of the genetic sequence.
   */
  def length: Int

  /**
   * Adds an element to the genetic sequence.
   * @param element the element to add
   * @return a new genetic sequence with the added element
   */
  infix def add(element: A): GeneticSequence[A]

  /**
   * Chains two genetic sequences together.
   * @param other the other genetic sequence to chain
   * @return a new genetic sequence that is the concatenation of this and the other
   */
  infix def chain(other: GeneticSequence[A]): GeneticSequence[A]

object GeneticSequence:
    type DNA = GeneticSequence[DNANucleotide]
    type RNA = GeneticSequence[RNANucleotide]

    /**
     * Creates a DNA genetic sequence from a list of DNA nucleotides.
     * @param elements the list of DNA nucleotides to create the genetic sequence from
     * @return
     */
    def dna(elements: List[DNANucleotide]): DNA = GeneticSequence(elements)
    
    /**
     * Creates an RNA genetic sequence from a list of RNA nucleotides.
     * @param elements the list of RNA nucleotides to create the genetic sequence from
     * @return
     */
    def rna(elements: List[RNANucleotide]): RNA = GeneticSequence(elements)
```
With both the nucleotide types and the `GeneticSequence` trait defined, It's possible to start structuring the tests, for now
using only ScalaTest and then, afterwards, integrating ScalaCheck for the axioms.
ScalaTests for Nucleotide can be found at `scala/u04/task1/nucleotide/NucleotideTest.scala` while the tests for GeneticSequence
are at `scala/u04/task1/GeneticSequenceTest.scala`.

Once I've defined the basic tests, I've started to implement the rest of the operations using a private case class
inside the `GeneticSequence` companion object to maintain the operation private and not expose the implementation details 
to the user.

```scala
private case class GeneticSequenceImpl[A <: Nucleotide](elements: List[A]) extends GeneticSequence[A]:
    override def length: Int = elements.length
    override def add(element: A): GeneticSequence[A] = GeneticSequenceImpl(elements :+ element)
    override def chain(other: GeneticSequence[A]): GeneticSequence[A] = GeneticSequence(
      this.elements ++ other.elements
    )
    override def contains(geneticSequence: GeneticSequence[A]): Boolean =
      this.elements.containsSlice(geneticSequence.elements)
```

After implementing the operations, I've started to integrate ScalaCheck to test the axioms defined in the specification.
Those tests can be found in `scala/u04/task1/properties` package.
We've got two different type of property tests: 
- The one defined using Scalacheck called `GeneticSequenceCheck`
- The one defined using ScalaTest called `GeneticSequenceScalaTestCheck`

Note that is actually possible to use ScalaTest for property testing as well, the key difference between the two is that
since ScalaCheck is a library specifically designed for property-based testing it provides more advanced control 
over the generation of test cases, such as custom generators, shrinking, and combinators for building complex properties.

> Shrinking is the process by which a PBT framework tries to reduce the failing random input
> to a property to its minimal value and it does this so that we as programmers don’t have to do much
> further investigation to find the “simplest” failing input.

ScalaTest supports property-based testing through AnyPropSpec with TableDrivenPropertyChecks:
- AnyPropSpec is a style of writing tests in ScalaTest that allows you to define properties that should hold true for a 
range of inputs.
- TableDrivenPropertyChecks is a trait that provides support for table-driven property checks, allowing you to define
a table of input values and expected results, and then automatically generate tests for each row in the table.


**DSL for GeneticSequence creation**
> Note: I developed this part out of pure personal curiosity, to try to understand how to effectively integrate a DSL 
> written in Scala into a context where there are axioms that must be respected. This is a bonus part of the task and is not
> required.


I've provided also a simple DSL to make the GeneticSequence creation easier in the tests. The definition of the DSL
is located in `scala/u04/task1/GeneticSequenceDSL.scala`

The DSL allows to create DNA and RNA sequences using a more natural syntax, for example:
```scala
DNANucleotide.G + DNANucleotide.T + DNANucleotide.A + DNANucleotide.C
RNASequence = RNANucleotide.A + RNANucleotide.U + RNANucleotide.C + RNANucleotide.G
```

Obviously the DSL should respect the axioms defined in the specification, so, for example the following should not compile:
```scala
val invalidSequence = DNANucleotide.G + RNANucleotide.U + DNANucleotide.A
```
For doing this I've exploited Type classes, context-bounds and implicit conversion mechanism of scala. In particular, 
I've defined a `SequenceFactory` trait used for helping generate a sequence of a specific type of nucleotide.

```scala
 trait SequenceFactory[A <: Nucleotide]:
    def createSequence(elements: List[A]): GeneticSequence[A]
    def single(element: A): GeneticSequence[A] = createSequence(List(element))
```

The following allow to create a sequence of a specific type of nucleotide,

```scala
  given SequenceFactory[DNANucleotide] with
    def createSequence(elements: List[DNANucleotide]): GeneticSequence[DNANucleotide] = GeneticSequence.dna(elements)

  given SequenceFactory[RNANucleotide] with
    def createSequence(elements: List[RNANucleotide]): GeneticSequence[RNANucleotide] = GeneticSequence.rna(elements)
```

Then I've defined an implicit conversion that will allow to transform a type of element `A` into a `GeneticSequence[A]` 
using the `single` method of the `SequenceFactory`.

```scala
  given [A <: Nucleotide: SequenceFactory]: Conversion[A, GeneticSequence[A]] with
    def apply(nucleotide: A): GeneticSequence[A] = summon[SequenceFactory[A]].single(nucleotide)
```
Finally it's possible to define the DSL using an extension method on both `Nucleotide` and `GeneticSequence[A]`:

```scala
extension [A <: Nucleotide](geneticSequence: GeneticSequence[A])
  infix def +(element: A): GeneticSequence[A] = geneticSequence.add(element)
  infix def ++(other: GeneticSequence[A]): GeneticSequence[A] = geneticSequence.chain(other)

extension [A <: Nucleotide: SequenceFactory](element: A)
  infix def +(other: A): GeneticSequence[A] = summon[SequenceFactory[A]].createSequence(List(element, other))
  infix def +(other: GeneticSequence[A]): GeneticSequence[A] = summon[SequenceFactory[A]].single(element) ++ other
```

### 2. Adt Verifier

The ADT defined for this task is the following:

```text
type :
    Sequence[A]
    
constructors :
    empty: Sequence[A]
    apply: A x A x ... x A => Sequence[A]
 
operations :
    add: A x Sequence[A] => Sequence[A]
    concat: Sequence[A] x Sequence[A] => Sequence[A]
    map [B]: Sequence [A] x (A => B) => Sequence [B]
    flatMap [B]: Sequence [A] x (A => Sequence [B]) => Sequence [B]
    filter: List [A] x (A => Boolean) => Sequence [A]
    foldLeft [B]: List [A] x B x (B, A) => B
    
axioms :

    add(empty, x) = apply(x)
    add(apply(xs), x) = apply(xs, x)
    
    concat(empty, ys) = ys
    concat(xs, ys) = apply(xs, ys)
    
    map(empty , f) = nil
    map(apply(xs), i => i) = apply(xs, f)
    map(apply(x, y), f) = apply(f(x), f(y))
    
    flatMap(empty, f) = nil
    flatMap(apply(x, y), f) = f(x) ++ f(y)
    
    filter(empty, f) = empty
    
    filter(apply(xs), f) = apply(filter(xs, f))
    
    foldLeft(empty, b, f) = b
    foldLeft(apply(x, xs), b, f) = foldLeft(xs, f(b, x), f)
```

The trait representing the ADT is the following (complete implementation at `scala/u04/task2/SequenceADT.scala`) and 
contains all the operations defined in the specification:

```scala
trait SequenceADT[A]:
  def add(element: A): SequenceADT[A]
  def concat(other: SequenceADT[A]): SequenceADT[A]
  def map[B](f: A => B): SequenceADT[B]
  def flatMap[B](f: A => SequenceADT[B]): SequenceADT[B]
  def filter(f: A => Boolean): SequenceADT[A]
  def foldLeft[B](init: B)(f: (B, A) => B): B
```

As regards testing the properties for both required implementations of the ADT, it is necessary to somehow separate the 
definition of these from the way in which the data for them is generated.
In order to achieve this, I've exploited the concept of composition of traits in Scala, defining `SequenceGenerator` that
contains the generator methods that should be implemented and injected into the property tests when needed.

```scala
trait SequenceGenerator:
  def generatorOf[A](gen: Gen[A]): Gen[SequenceADT[A]]
  
  def createEmpty[A]: SequenceADT[A]
  
  def of[A](elements: A*): SequenceADT[A]
```

The file that contains the property (`scala/u04/task2/properties/SequenceProperties.scala`) is defined as follows:

```scala
abstract class SequenceProperties(name: String) extends Properties(name) :
  self: SequenceGenerator =>

    property("add empty") = //...
    property("add cons") = //...
```

So it require a concrete implementation where also a trait that implements `SequenceGenerator` is mixed in.
So all the general properties are defined in a single file and can be reused for different implementations of the ADT.

```scala
object ListProperties extends SequenceProperties("ListSequence") with SequenceGenerator:

  import scala.u04.task2.SequenceList.*

  override def generatorOf[A](g: Gen[A]): Gen[SequenceADT[A]] = Gen.listOf(g).map(xs => apply(xs *))
  override def createEmpty[A]: SequenceADT[A] = empty[A]
  override def of[A](elements: A*): SequenceADT[A] = apply(elements*)

object ConsNilProperties extends SequenceProperties("ConsNil") with SequenceGenerator:
    import scala.u04.task2.SequenceConsNil.*
    
    override def generatorOf[A](g: Gen[A]): Gen[SequenceADT[A]] = Gen.listOf(g).map(xs => apply(xs*))
    override def createEmpty[A]: SequenceADT[A] = empty[A]
    override def of[A](elements: A*): SequenceADT[A] = apply(elements*)
```

### 3. Monad Verifier
ADT defined for this task is the following:

```text
```text
type: 
    Monad[M[_]]

constructors: 
        unit: A => M[A]
    
operations: 
    flatMap: M[A] x (A => M[B]) => M[B]
    map: M[A] x (A => B) => M[B]
    map2: M[A] x M[B] x (A, B) => M[C]
    seq: M[A] x M[B] => M[B]
    seqN: Stream[M[A]] => M[[A]]
    
axioms:
    flatMap(unit(a), f) = f(a)
    flatMap(m, a => unit(a)) = m
    flatMap(flatMap(m, f), g) = flatMap(m, a => flatMap(f(a), g))
    
    map(unit(a), f) = unit(f(a))
    map(m, x => x) = m   
    map(m, f) = flatMap(m, a => unit(f(a)))
    
    map2(unit(a), unit(b), f) = unit(f(a, b)) 
    map2(m1, m2, (a, b) => b) = seq(m1, m2)   
    
    seqN(Stream(m)) = m
    seqN(Stream(m, ms)) = seq(m, seqN(Stream(ms)))
```
I've used an approach similar to the one used in the previous task, trying to separate the definition of the properties from
the way in which the data is generated.
This time instead of mixing in a trait that generates the data, I've decided to use `given` generator instances that can be
injected when needed.

```scala
abstract class MonadProperties(name: String) extends Properties(name):
  import scala.u04.task3.generator.MonadGenerator.{given, *}
  
  property("flatMap unit") = //...
  // etc...
```

where `MonadGenerator` is defined as follows:

```scala
case object MonadGenerator:
  def monadGenerator[M[_]: Monad, A: Arbitrary]: Gen[M[A]] = for
    el <- arbitrary[A]
    m <- summon[Monad[M]].unit(el)
  yield m

  import u04.datastructures.Streams.*

  given[A: Arbitrary]: Arbitrary[Stream[A]] =
    Arbitrary:
      Gen.oneOf(
        Gen.const(Empty()),
        for 
          head <- arbitrary[A]
          tail <- arbitrary[Stream[A]]
        yield Stream.cons(head, tail)
      )

  given [M[_]: Monad, A: Arbitrary]: Arbitrary[Stream[M[A]]] =
    Arbitrary:
      Gen.oneOf(
        Gen.const(Empty()),
        for
          head <- monadGenerator[M, A]
          tail <- arbitrary[Stream[M[A]]]
        yield Stream.cons(head, tail)
      )
  
  given [M[_] : Monad, A: Arbitrary](using Cogen[A]): Arbitrary[A => M[A]] =
    Arbitrary(Gen.function1(monadGenerator[M, A]))
```
So the tests can be defined more easily:

```scala
object OptionProperties extends MonadicProperties[Optional]("Option Properties")
```


### 4. ADVANCED FP LLM
For this lab I've used LLMs for trying to enrich / improve ADT specifications. For doing this, obviously the first step
was to provide the LLM a way to understand the ADT specification language used both during the lesson and also adopted
in these tasks.
So I've provided a prompt that describes the ADT specification language and also provides some examples of ADTs
specified using that language.

>You are an expert in software engineering and formal specifications.
>I will provide you with a formal specification of an Abstract Data Type (ADT) using a specific language.
>The specification includes the type, constructors, operations, and axioms of the ADT.
>The language used for the specification is as follows:  

```text
type :
    <TypeName>[<TypeParameters>] = <Description of the type>
constructors :
    <ConstructorName>: <ParameterType1> x <ParameterType2> x ... x
    <ParameterTypeN> => <TypeName>[<TypeParameters>]
operations :
    <OperationName>: <ParameterType1> x <ParameterType2> x ... x
    <ParameterTypeN> => <ReturnType>

axioms :
    <Axiom1>
    <Axiom2>

```

```text
type :
    Sequence[A]
    
constructors :
    empty: Sequence[A]
    apply: A x A x ... x A => Sequence[A]
 
operations :
    add: A x Sequence[A] => Sequence[A]
    concat: Sequence[A] x Sequence[A] => Sequence[A]
    map [B]: Sequence [A] x (A => B) => Sequence [B]
    
axioms :

    add(empty, x) = apply(x)
    add(apply(xs), x) = apply(xs, x)
```

After providing the prompt, I've asked to enrich the ADT specification for sequences defined in task 2,
asking to add more operations and axioms to the specification starting from the original one.

This is an example of a result obtained for the sequence ADT:
```text
type :
    Sequence[A]
    
constructors :
    empty: Sequence[A]
    apply: A x A x ... x A => Sequence[A]
 
operations :
    add: A x Sequence[A] => Sequence[A]
    concat: Sequence[A] x Sequence[A] => Sequence[A]
    map [B]: Sequence [A] x (A => B) => Sequence [B]
    flatMap [B]: Sequence [A] x (A => Sequence [B]) => Sequence [B]
    filter: List [A] x (A => Boolean) => Sequence [A]
    foldLeft [B]: List [A] x B x (B, A) => B
    
axioms :

    add(empty, x) = apply(x)
    add(apply(xs), x) = apply(xs, x)
    
    concat(empty, ys) = ys
    concat(xs, ys) = apply(xs, ys)
    
    map(empty , f) = nil
    map(apply(xs), i => i) = apply(xs, f)
    map(apply(x, y), f) = apply(f(x), f(y))
    
    flatMap(empty, f) = nil
    flatMap(apply(x), f) = f(x)
    flatMap(apply(x1, x2, ..., xn), f) = concat(f(x1), flatMap(apply(x2, ..., xn), f))
    flatMap(apply(x, y), f) = f(x) ++ f(y)
    
    filter(empty, f) = empty
    filter(apply(x1, x2, ..., xn), p) = concat(filter(apply(x1), p), filter(apply(x2, ..., xn), p))
    filter(apply(xs), f) = apply(filter(xs, f))
    
    foldLeft(empty, b, f) = b
    foldLeft(apply(x, xs), b, f) = foldLeft(xs, f(b, x), f)
    foldLeft(concat(xs, ys), b, f) = foldLeft(ys, foldLeft(xs, b, f), f)
```
