## Debugging implicit resolution {#sec:generic:debugging}

```tut:book:invisible
import shapeless._

trait CsvEncoder[A] {
  val width: Int
  def encode(value: A): List[String]
}

object CsvEncoder {
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc
}

def createEncoder[A](w: Int)(func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    val width = w
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(1)(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(1)(num => List(num.toString))

implicit val doubleEncoder: CsvEncoder[Double] =
  createEncoder(1)(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(1)(bool => List(if(bool) "yes" else "no"))

implicit def optionEncoder[A](implicit encoder: CsvEncoder[A]): CsvEncoder[Option[A]] =
  createEncoder(encoder.width)(opt => opt.map(encoder.encode).getOrElse(List.fill(encoder.width)("")))

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(0)(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case h :: t =>
      hEncoder.value.encode(h) ++ tEncoder.encode(t)
  }

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(0)(cnil => ???)

implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case Inl(h) => hEncoder.value.encode(h) ++ List.fill(tEncoder.width)("")
    case Inr(t) => List.fill(hEncoder.value.width)("") ++ tEncoder.encode(t)
  }

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  enc: Lazy[CsvEncoder[R]]
): CsvEncoder[A] =
  createEncoder(enc.value.width)(a => enc.value.encode(gen.to(a)))
```
