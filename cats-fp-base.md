## ammonite reple  
`ammonite reple` はリッチなscalaのREPLだ  
関数型及びcatsの話とは関係ないが、以降の説明で度々出てくるサンプルコードを動かして学んで欲しいため  
そのためのツールの導入と説明を初めに説明しておく  

repl本体については、このリポジトリにDockerfileを用意しているので、使って欲しい  

```sh
$ cd <repo root>/repl

$ docker build -t amm .

$ docker run -it amm
```

ammイメージにはデフォルトでcatsのライブラリが使えるように細工をしている、これはammoniteで提供されているmagic importと  
言う機能を使ったものだ  
立ち上がれば、即座にcatsのライブラリが使用できる  

```scala
@ import cats._
import cats._

@ import cats.implicits._
import cats.implicits._

@ Functor[Option].map(Option(3)) { i => i + i }
res3: Option[Int] = Some(6)
```

複数行のコードを一気に読み込む場合は `{ }` で囲む必要がある  
これはコンパニオンオブジェクトを定義したい場合などに有用だ  

```scala
@ {
    case class Foo(bar: Int)
    object Foo {
      def showBar(foo: Foo): String = s"bar is ${foo.bar}"
    }
  }
defined class Foo
defined object Foo
```

使い方は以上だ、これで足回りが整ったと思うので思う存分サンプルコードたちを実行して欲しい

## 型クラス(type class)  
いきなり「型クラスとは」と聞くと身構えるかもしれないが  
型クラスはcatsを理解する上で重要な概念だ  

型クラスは振る舞いのインタフェースを定義する、型クラスの実装は任意のデータ構造(データ型とも呼ばれる)に対して定義され  
scalaの世界では `implicit value` で表される  
以下は簡単な例だ(cats上ではもっと高級なものが定義されている、これは説明のためにシンプルに再定義したものと思って欲しい)  

* ammonite REPLを使用する場合は一番外側を `{ }` で囲む必要がある  
```scala
// type class Semigroup
// 実装した場合、データ構造が結合可能な振る舞いを持つことを表す
trait Semigroup[A] {
  def combine(lhs: A, rhs: A): A
}
object Semigroup {
  // Semigroup[Amount].combine(a, b) のように使うヘルパークラス、summonerとも呼ばれる
  def apply[A](implicit s: Semigroup[A]): Semigroup[A] = s
}

// 金額を表す データ構造
case class Amount(value: BigDecimal)

object Amount {
  // 型クラスインスタンスの定義はどこにしてもよいが、可能であればデータ構造のコンパニオンオブジェクトに設置すると
  // importなしに自動的に導出できる
  implicit val semigroupInstance: Semigroup[Amount] = new Semigroup[Amount] {
    def combine(lhs: Amount, rhs: Amount): Amount = Amount(lhs.value + rhs.value)
  }
}

object Test {
  // Semigroupを実装したデータ構造Aに対して、抽象的な関数を定義する場合は以下のように書く
  // def f[A: Foo] はContextBoundと呼ばれ、 def f[A](implicit x: Foo) のシンタックスシュガーだ
  def squash[A: Semigroup](xs: List[A])(default: A): A = {
    if (xs.isEmpty) default
    else xs.reduce { (l, r) => Semigroup[A].combine(l, r) }
  }

  def case1: Unit = {
    val amounts = List(
      Amount(BigDecimal(1.0)),
      Amount(BigDecimal(2.6)),
      Amount(BigDecimal(3.4))
    )

    // 関数 squash は暗黙的に Semigroup[Amount] を受け取る
    // 仮に型クラス実装(implicit vallu)をコンパニオンオブジェクトに置いていないなら、importしてスコープないに入れる必要がある
    println(squash(amounts)(Amount(10.0))) // => Amount(7.0)
  }
}
```

scalaの世界において、型クラスは表現したい振る舞いの関数インタフェースをもったただの `trait` だ  
データ構造は `case class` で表される(もちろん、通常のclassでもよいが、多くの場合case classのほうが使いやすいだろう)  
そして、型クラスの実装は `implicit value` だ(implicitな値であればよいので `implicit def` を使うこともできる、ただこれが有用になるのはもう少しあとだ)  
実装は関数型の世界では「型クラスインスタンス」とも呼ばれる  

見ての通り, 基本的な型クラスの実態はscalaのprimitiveな機能だけで十分実装できるものだ  
前述した通り、型クラスは振る舞いをインタフェースとして定義するもので `Semigroup` はデータ構造 `A` を2つ取り、それらを結合した新しい `A` を返す、という振る舞いを定義する  
(結合律を満たす必要がある、などの暗黙的な制約はあるが、今回は取り扱わない、初めのうちは関数のインタフェースが示しているものを素直に実装すればよい)  

scalaに置いて振る舞いを定義するもう一つの手段として継承がある, こちらのほうが皆に馴染みのある手法だろう  
以下はSemigroupを継承で実現した簡単な例だ  

```scala
// 継承したクラスは結合することを可能とする
trait Semigroup[Self] {
  def combine(that: Self): Self
}

case class Amount(value: BigDecimal) extends Semigroup[Amount] {

  def combine(that: Amount): Amount = Amount(this.value + that.value)
}

object Test {
  // 継承を利用する場合の抽象化はおなじみの上限境界だ
  def squash[A <: Semigroup[A]](xs: List[A])(default: A): A = {
    if (xs.isEmpty) default
    else xs.reduce { (l, r) => l.combine(r) }
  }

  def case1: Unit = {
    val amounts = List(
      Amount(BigDecimal(1.0)),
      Amount(BigDecimal(2.6)),
      Amount(BigDecimal(3.4))
    )
    println(squash(amounts)(Amount(10.0))) // => Amount(7.0)
  }
}
```

すごくシンプルだ！  
型クラスは(sub class typingに置ける)class機能を持たない言語の文化のため、場合によってはclass継承を利用したほうが  
完結になる場合もあるだろう、例えばコンパクトでライフタイムの短いコードなどが当てはまる  

しかし、complexな振る舞いを定義する場合や、外部の及び既存のデータ構造に対して新しい振る舞いを適用したい場合に型クラスは有用になってくる  
型クラスにおけるinstance実装はデータ構造と分離されており、合成もしやすいからだ  

時に継承の表現は理解の助けになることもある  
振る舞いについて難しいと感じた型クラスに置いては、一旦class継承に置き換えてで再定義してみるのもひとつの手段だ  

## 高階型(higher kinded types)  
高階型は簡単に言うと型をひとつ引数にとる型 `F[_]` だ,  
馴染み深いもので言うとListやOptionが高階型の一例だ  
高階型自体は簡単な概念だか、型クラスを定義する上において高階型は通常の型と区別されるためここで説明しておく  
一例をあげよう  

```scala  
// Functor型クラスはデータ構造 F[A] に対し、関数 f: A => B を適用できる振る舞いを示す型クラスだ
// Functorは実装するデータ構造の型が F[_] である必要がある
trait Functor[F[_]] {
  // 実はscala collectionでお馴染みのmap関数だ
  def map[A, B](fa: F[A])(f: A => B): F[B]
}
object Functor {
  def apply[F[_]](implicit s: Functor[F]): Functor[F] = s
}

// Optionを別の名前で再実装してみよう
// Maybe[A] は一つ型をとるデータ構造だ, つまり高階型 F[A] だ
sealed trait Maybe[+A]
case class Just[A](value: A) extends Maybe[A]
case object Empty extends Maybe[Nothing]

object Maybe {
  // {Just|Empty} をそのまま利用すると Functor[Just] のような型クラスをコンパイラに期待されてしまうので
  // 型がMaybeに潰れるようにヘルパーを定義しよう
  def just[A](a: A): Maybe[A] = Just(a)
  def empty[A]: Maybe[A] = Empty

  implicit val functorInstance: Functor[Maybe] = new Functor[Maybe] {
    def map[A, B](fa: Maybe[A])(f: A => B): Maybe[B] = fa match {
      case Just(a) => Just(f(a))
      case Empty => Empty
    }
  }
}

object Test {
  def showDetail[F[_]: Functor](fa: F[Int]): F[String] = {
    // 言うまでもないが第二引数は Int => String の関数だ
    Functor[F].map(fa) { a => s"payload: ${a.toString}" }
  }

  def case1: Unit = {
    println(showDetail(Maybe.just(3))) // => Just("payload: 3")
    println(showDetail(Maybe.empty[Int])) // => Empty
  }
}
```

少し長いサンプルコードになってしまったが  
Functor型クラスはコメントにも書いてある通り、型引数をひとつとるデータ型 `F[_]` 対して  
内部の値 `A` を `B` に写す関数を適用できる振る舞いを定義する  
「内部の値」という書き方をしたが実際にデータ構造 `F[A]` が自身のメンバとして値 `A` を持つ必要はない(Functorインスタンスが守るべき制約, Functor則にもそのような制約はない)  
例えば `Int => A` を写す関数を持つデータ構造に対しFunctorを定義することもできる  

```scala
case class FromInt[A](run: Int => A)

object FromInt {
  // 関数 f: A => B を受け取り FromInt[B](rum: Int => B) を生成する
  implicit val functorInstance: Functor[FromInt] = new Functor[FromInt] {
    def map[A, B](fa: FromInt[A])(f: A => B): FromInt[B] = FromInt(fa.run.andThen(f))
  }
}

object Test {
  def case1: Unit = {
    val stringInt: FromInt[String] = FromInt(i => i.toString)

    val mapped: FromInt[Char] = Functor[FromInt].map(stringInt) { s => s.head }

    println(mapped.run(30)) // => 3
  }
}
```

## 直和(coproduct) と 直積(product)  
直和と直積は型クラスを実装するデータ構造を分類するための言葉だ  
よく出てくる言葉なので紹介しておく、scalaで定義するとすごく馴染み深いものになる  

```scala
// 直和
sealed trait Fruits
case object Apple extends Fruits
// パタンマッチでextractできれば中に値を別の持っていたって構わない
case class Orange(seed: Int) extends Fruits

// 直積
case class Bucket(apple: Apple, orage: Orange)
```

直和は複数の派生型をもち、それらがexplicitである(つまりパタンマッチができる)必要がある、聞き慣れた言葉を使うのであればEnumだ  
直積が複数の型の値を内包する、scalaでは単純なcase classとして表現される  
これらのデータ構造は型クラスの実装を定義する上で便利であり、自身の限定された型の範囲で振る舞いを変更することができる  
先に示したMaybe型のFunctorの実装がまさにそれだ  

## データ構造と型クラス  
関数型のプログラミングを学ぶ上で「データ構造」と「型クラス」の区別はしっかりつけておいて欲しい  
型クラスはデータ構造に対して実装し、利用する側では「型クラスを実装していること」を条件に抽象的な関数を定義するが  
データ構造が型クラスそのものではないのだ  
もし読者が関数型について少しでも調べたことがあるのなら、すぐにでも「モナド(Monad)」「ステートモナド(StateMonad)」「リーダモナド(ReaderMonad)」などの言葉を目にしただろう  
「Monad」は型クラスだが、「State」「Rader」はデータ構造だ、もちろんそれらも「Monad」型クラスを実装しているため  
抽象化された「Monad」として扱うことができるし、「Monad」として扱われることのほうが一般的だ  
ただあくまでStateが(Readerが)Monadの振る舞いを実装しているのであって同レイヤに存在しているものではない、ということを覚えておいて欲しい  

## Catsの世界  
長々と関数型プログラミングの登場人物を紹介したが、ここからはcatsの話をしていく  
catsは上記で説明したような関数型のテクニックをscalaの世界に落とし込んだライブラリだ  
すでに説明したSemigroupやFunctorも定義されているし、他にも沢山のデータ型やそれに対する型クラス実装も定義されている  
この項では、catsの使い方と動かす仕組みについて説明していこう  

```scala
import cats.syntax._
import cats.implicits._

object Test {
  def case1: Unit = {
    val xs: List[Int] = List(2, 5, 4, 7)

    // 全ての要素が偶数であればRightを返し、要素の値を奇数に変換する
    // ひとつでも奇数が含まれていればLeftを返す
    val res = xs.traverse { n => Either.cond(n % 2 == 0, n+1, s"$n is odd.") }

    println(res) // Left("5 is odd")
  }
}
```

これは僕がもっともよく使うtraverse関数のサンプルだ  
もちろんscala標準のListにはtraverseなんて関数は生えていない、catsが魔法で(魔法ではないが)使えるようにしてくれているのだ  
まず、traverse関数の説明からしよう  

traverse関数は型クラス `Traverse[F[_]]` によって定義されている関数だ  

```scala
import cats._

trait Traverse[F[_]] extends Foldable[F] {
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]
}
```  
* 実際には他の継承もあるが、説明のため必要なものだけここでは継承させている  

catsでは型クラスの継承表現をscalaの継承で実装する  
「型クラスの継承」というのは、ある型クラスを実装するために、前提として必要とされている型クラスのことだ  
つまり `Traverse[F[_]]` をデータ構造 `F[_]` に実装するためには、まず `Foldable[F[_]]` の振る舞いを実装しなければならない  
もちろん一緒に実装したっていい  
また、traverse関数自体も、登場人物となる別のデータ構造 `G[_]` に対して `Applicative[G[_]]` を要求する  
( `Applicative[G[_]]` と書いたが、文脈上 `G` を使っただけで, 表記の上では `Applicative[F[_]]` だっていいし、 `Applicative[X[_]]` だっていい、つまり高階型を受け取ることを示せれば、なんだってよいのだ)  

ここで新しく2つの型クラスが登場したのでひとつひとつ説明しておこう  

### Foldable  
Foldableはデータ構造 `F[_]` に対して「畳み込める」振る舞いを要求する  
「畳み込める」振る舞いというのはListに当てはめてみるとわかりやすい  

```scala
import cats._

Foldable[List].foldLeft(List(1, 2, 3), 0) { case (acm, i) => acm + i } // => 6: Int
```

見ての通りListの畳み込みは自身の持つ要素列に対して Accumlate とひとつづつ要素を引数にとり、順繰りに適用していくことだ  
これは標準ライブラリの `List#foldLeft` の振る舞いと同じだと思って構わない  

他にも見てみよう  
Option や Either は要素数が1または0のリストのように振る舞う  

```scala
import cats._

Foldable[Option].foldLeft(Some(1), 0) { case (acm, i) => acm + i } // => 1: Int
// Noneの場合、Nilのように振る舞う
Foldable[Option].foldLeft[Int, Int](None, 0) { case (acm, i) => acm + i } // => 0: Int

type StringEither[A] = Either[String, A]
Foldable[StringEither].foldLeft(Right(1), 0) { case (acm, i) => acm + i } // => 1: Int
// Leftの場合、Nilのように振る舞う
Foldable[StringEither].foldLeft[Int, Int](LefNt("left."), 0) { case (acm, i) => acm + i } // => 0: Int
```

* 説明が遅れたがEitherのように型引数を二つとる高階型は片方を事前に適用することで型引数を一つだけとる高階型のように扱える  
  `kind-projecter` というsbt-pluginを使えば `F[_]` を要求する箇所に `Either[String, *]` という風に直接書くこともできる  
  e.g. `Foldable[Either[String, *]]`  


## Applicative  
Applicativeは今までの型クラスに比べると少しだけ難しい  
今までの型クラスになかった要素として、Applicativeは実装するデータ構造 `F[_]` について  
振る舞いの他に, ある文脈(context)を表現していることを期待する(期待する、と書いたのはプログラム上では縛れない制約だからだ)  
例えば `Option` はApplicativeの実装を持つため、何らかの「文脈」を表したデータ構造だということになる  
`Option` では(scalaユーザにはお馴染みだが)、内包された値 `A` が「存在するかもしれないし」「存在しないかもしれない」という文脈を直和で表現している  
つまり、文脈を表しているデータ構造とはこう言うことだ  

```scala
sealed trait Option[+A]
object Option {
  // 値が存在していることを表す型
  case class Some[A](value: A) extends Option[A]
  // 値が存在して"いない"ことを表す型
  case object None extends Option[Nothing]
}
```

Applicativeは以下のように定義されている、要求する振る舞いは2つで `ap` と `pure` だ  
さらにFunctorを継承している  

```scala
import cats._

trait Applicative[F[_]] extends Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

  def pure[A](a: A): F[A]
}
```

`pure` は文脈を持たない `A` を最小限の文脈を持つ値に持ち上げる(lift)関数だ  
例えば, Optionなら値を持っているという文脈を表現している `Some` に、Eitherなら成功を表す `Right` だ  
最小限の文脈、というと難しいが基本的には値 `A` を一つだけ持つラップしたデータ構造に持ち上げられる、とイメージしてもよいだろう  
間違っても受け取った引数を闇に消し去るようなApplicativeの実装は(おそらく)存在しない  

`ap` は文脈で包まれた関数(型シグネチャが一眼では理解しづらいが、単に値として関数をもっているだけだ)に文脈でつつまれた値を適用する関数だ  
これについては見た方が早いため、実際にOptionを例にを触ってみよう  

```scala
import cats.Applicative
import cats.syntax._
import cats.implicits._

object Test {
  def case1: Unit = {
    val fa: Option[Int] = Some(1)
    val ff: Option[Int => Int] = Some { a: Int => a + a }

    val res = Applicative[Option].ap(ff)(fa)
    println(res) // => Some(2)
  }

  def case2: Unit = {
    val fa: Option[Int] = None
    val ff: Option[Int => Int] = Some { a: Int => a + a }

    val res = Applicative[Option].ap(ff)(fa)
    println(res) // => None
  }

  def case3: Unit = {
    val fa: Option[Int] = Some(1)
    val ff: Option[Int => Int] = None

    val res = Applicative[Option].ap(ff)(fa)
    println(res) // => None
  }
}
```

`case1` は見ての通り、Optionで包まれた関数に対してOptionで包まれた値を適用してOptionで包まれた結果を得ている  
`case2` `case3` の結果はいずれもNoneだ、これは `ap` 関数が引数で渡されたデータ構造の文脈を結果に対しても「引き継ぐ」ことを意味している  
端的に文脈を「引き継ぐ」と書いたが、文脈の引き継ぎ方自体はApplicativeの実装に委ねられている、例えばOptionやEitherなど失敗(や値の存在しないこと)を表す文脈  
をもっているものであれば、大抵の場合は失敗の文脈が引き継がれる(つまり、引数の両方が成功を示していた場合のみ成功の文脈を持った結果が返る)  

`ap` 関数は一見使いづらく見えるが、これによって以下のような関数を定義することができる  

```scala
import cats.Applicative
import cats.syntax._
import cats.implicits._

object Test {
  def leftBias[A, B, F[_]: Applicative](fl: F[A], fr: F[B]): F[A] = {
    // 引数を二つうけとり、第一引数の値を返す関数
    val leftBias0: A => B => A = { l => r => l }

    // 関数 leftBias0 を F[_] の文脈へliftする, F[A => B => A] となる
    val leftBiasF = Applicative[F].pure(leftBias0)
    // 第一引数に fl を適用, 引数がひとつ消化され F[B => A] となる
    val step1 = Applicative[F].ap(leftBiasF)(fl)

    // 第二引数に fr を適用, 結果が返る
    Applicative[F].ap(step1)(fr)
  }
  def case1: Unit = {
    val fa: Option[Int] = Some(1)
    val fb: Option[Unit] = Some(())

    println(leftBias(fa, fb)) // Some(1)
    println(leftBias(fa, None)) // None
    println(leftBias(None, fb)) // None
  }
}
```

この関数 leftBias は、例えば副作用やvalidationなどを文脈としたデータ構造(catsではIOやValidateと呼ばれるものがそれだ)  
に対して使うと有効だ、左側の値を保ちつつ、右側の副作用やvalidation処理を実行し、failした場合に結果として失敗を返すことができる  

実際にcatsの中でも `<*` と `*>` という記号の関数で定義されているので、機会があれば使って欲しい(ちなみに `ap` はエイリアスとして `<*>` が定義されている  

冒頭に「今までの型クラスになかった要素」と行ったが、実はFunctorもApplicativeと同じで、データ構造 `F[_]` に対して文脈をもつことを期待する  
Applicativeが「文脈を持った関数」に対して「文脈を持った値」を適用するのに対して、Functorは「純粋な関数」(文脈を持たない関数)に対して「文脈を持った値」を適用できるのが特徴だ  

### Traverse  
さて、部品は揃ったので今度こそTraverseの説明だ  
ここまで説明してきた内容があれば、traverse関数を「言葉」で説明することができる  
traverse関数は「畳み込みの性質を持っているデータ構造 `F[_]` に対し、すべての要素 `A` に関数 `A => G[B]` を適用し、全ての結果を畳み込みながら `G[B]` の文脈を引き継いだ結果を返す」という関数だ、むずかしい  
今度は `Traverse[List]` を例にコードで表してみよう  
(以下は僕が上の言葉からコードを起こしたものだ、振る舞いはだいたい同じだが、実装はcatsのものと異なることを気をつけて欲しい)  

```scala
import cats.{
  Applicative,
  Foldable,
  Functor
}
import cats.Eval

trait Traverse[F[_]] extends Foldable[F] {
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]
}
object Traverse {
  def apply[F[_]](implicit t: Traverse[F]): Traverse[F] = t
}

object ListImplicits {
  implicit val traverseInstance: Traverse[List] = new Traverse[List] {
    // Foldableを実装する必要がある
    def foldLeft[A, B](fa: List[A], b: B)(f: (B, A) => B): B = fa.foldLeft(b)(f)
    // 今回は使わないので実装しない
    def foldRight[A, B](fa: List[A], lb: Eval[B])(f: (A, Eval[B]) => Eval[B]): Eval[B] = ???

    def traverse[G[_]: Applicative, A, B](fa: List[A])(f: A => G[B]): G[List[B]] = {
      fa match {
        // 空リストの場合は、G[_] における最小限の文脈に包んで結果を返す
        case Nil => Applicative[G].pure(Nil)
        case h :: tail =>
          // ApplicativeはFunctorを継承しているのでmapも使用できる
          val init = Applicative[G].map(f(h)) { ah => ah :: Nil }
          val gxs = foldLeft[A, G[List[B]]](tail, init) { case (acm, x) =>
            val gh = f(x)
            val gcons = Applicative[G].map(acm) { t: List[B] => ((xh: B) => xh :: t)}
            Applicative[G].ap(gcons)(gh)
          }
          // consで組み立てたため、reverseする
          Applicative[G].map(gxs) { _.reverse }
      }
    }
  }
}

object Test {
  import ListImplicits._

  // Applicative[Either[String, *]] を定義する
  type StringEither[A] = Either[String, A]
  implicit val eitherApplicativeInstance = new Applicative[StringEither] {
    def pure[A](a: A) = Right(a)
    def ap[A, B](ff: StringEither[A => B])(fa: StringEither[A]): StringEither[B] = (ff, fa) match {
      case (Left(e), _) => Left(e)
      case (_, Left(e)) => Left(e)
      case (Right(f), Right(a)) => Right(f(a))
    }
    override def map[A, B](fa: StringEither[A])(f: A => B): StringEither[B] = fa.map(f)
  }

  def case1: Unit = {
    val xs: List[Int] = List(2, 5, 4, 7)
    val res = Traverse[List].traverse(xs) { n => Either.cond(n % 2 == 0, n+1, s"$n is odd.") }

    println(res) // => Left(5 is odd.)
  }
  def case2: Unit = {
    val xs: List[Int] = List(2, 4, 6, 8)
    val res = Traverse[List].traverse(xs) { n => Either.cond(n % 2 == 0, n+1, s"$n is odd.") }

    println(res) // => Right(List(3, 5, 7, 9))
  }
}
```

Applicativeを使うことで要素ひとつに対する関数適用後の `G[_]` の文脈を引き継げるので  
ひとつが失敗すれば、全体を失敗とする、結果を返せるのだ  

## cats.instances._  
前項ではListに対するTraverse型クラス、そしてEither[String, *]に対するApplicative型クラスを自力で実装した  
しかし、catsを使う上では自力実装は必要最低限に抑え、catsが提供してくれている標準の型クラスインスタンスを使うべきだろう  
catsで定義されているインスタンスは全て `cats.instances.*` または、catsで定義されているデータ構造のコンパニオンオブジェクトに定義されている  
scala標準で用意されているデータ構造に対する型クラスインスタンスが欲しければ `cats.instances.*` を探せばいい、例えば `cats.instances.option._` や `cats.instances.list._` だ  

```scala
// 型クラスインスタンスはimplicit valueで提供されているため、package object配下をimportする必要がある
import cats.instances.list._
import cats.instances.try_._
import cats.Traverse

import scala.util.Try

object Test {
  def case1: Unit = {
    val res = Traverse[List].traverse(List(2, 1, 0)) { x => Try { 10 / x  } }

    println(res) // => Failure(java.lang.ArithmeticException: / by zero)
  }
}
```

catsで定義されているデータ構造の型クラスインスタンスは、コンパニオンオブジェクトに定義されているため、`cats.instances` からは何もimportしなくとも使うことができる  

```scala
// 必ず要素の数が一つ以上のListを表す型だ
import cats.data.NonEmptyList
// Applicativeによってエラーを累積できる失敗する可能性を文脈としてもつ型だ
import cats.data.{ Validated, ValidatedNel }

object Test {
  def case1: Unit = {
    val res = Traverse[NonEmptyList].traverse(
      NonEmptyList.of(2, 0, 1, 0)
    ) { x => if (x == 0) Validated.invalidNel("0 found.") else Validated.valid(10 / x) }

    println(res) // => Invalid(NonEmptyList(0 found., 0 found.))
  }
}
```

`cats.instances.all._` はcatsで定義されているすべてのデータ構造に対する型クラスインスタンスをimportする  


## cats.syntax._  
catsは型クラスに定義されている関数をメソッド呼び出しのように呼び出せるヘルパーを用意している  
それらは `cats.syntax.<type class name>` packageに定義されており, importすることで使うことができる  

```scala
// xs.traverse 形式で呼び出せる
import cats.syntax.traverse._
// もちろん、使用するときはsummonerを使用したときと同じ型クラスインスタンスも必要だ
import cats.instances.list._
import cats.instances.try_._

import scala.util.Try

object Test {
  def case1: Unit = {
    // レシーバ(第一引数)から直接型クラスのメソッドを呼び出すことができる
    val res = List(2, 1, 0).traverse { x => Try { 10 / x  } }

    println(res) // => Failure(java.lang.ArithmeticException: / by zero)
  }
}
```

とてもスッキリした!やっとコードにscalaらしさを取り戻せた気さえする!  
(scalaらしさとは、ただの哲学だ)  

少し前に書いたtraverse関数も書き直してみよう  

```scala
import cats.Applicative
import cats.syntax.applicative._
// みんなには黙っていたが, 実はcatsではap関数は Apply 型クラスの持ちものなのだ
import cats.syntax.apply._
import cats.syntax.functor._

def traverse[G[_]: Applicative, A, B](fa: List[A])(f: A => G[B]): G[List[B]] = {
  fa match {
    case Nil => List.empty[B].pure[G]
    case h :: tail =>
      val init = f(h).map { ah => ah :: Nil }
      val gxs = tail.foldLeft(init) { case (acm, x) =>
                  val gh = f(x)
                  val gcons = acm.map { t: List[B] => ((xh: B) => xh :: t)}
                  // syntaxは同時に、中置演算子も提供している、これはapの中置演算子だ
                  gcons <*> gh
                }
      // consで組み立てたため、reverseする
      gxs.map { _.reverse }
  }
}
```

スバラシイ!

imortできるsyntax関数は厳格だ、継承関係にあるsyntaxのものはimportされない(たとえば, `syntax.applicative` ではmap関数はimportされない)よく使うものを羅列しておこう
### cats.syntax.functor._
- `F[A].map[B](f: A => B): F[B]` : データ構造 `F[A]` に関数 `A => B` を適用する
- `F[A].void: F[Unit]` : 文脈を引き継ぎデータ構造 `F[Unit]` を返す (つまり `fa.map(_ => ())` だ)
- `F[A].as[B](b: B): F[B]` : 文脈を引き継ぎ, 受け取った純粋な値Bを埋め込んだデータ構造 `F[B]` を返す (つまり `fa.map(_ => b)` だ)

### cats.syntax.applicative._
- `A.pure[F]` : 純粋な値を最小限の文脈をもった `F[_]` へliftする

### cats.syntax.apply._
- `F[A] <* F[B]` : 両者の文脈を引き継ぎ、左側の値を持った `F[A]` を返す
- `F[A] *> F[B]` : 両者の文脈を引き継ぎ、右側の値を持った `F[B]` を返す

### cats.syntax.bifunctor._
- `X[A, B].bimap(f: A => C, g: B => D): X[C, D]` : 型引数を「二つ取る」高階型 `X[_, _]` の値 `A` `B` にそれぞれ関数を適用する(`X[_, _]` の例には `Either` や `cats.data.Ior` がある)

### cats.syntax.traverse,_
- `F[A].traverse[G[_]: Applicative, B](f: A => G[B]): G[F[B]]` : 先に上げた例の通り
- `F[A].flatTraverse[G[_]: Applicative, A, B](f: A => G[F[B]])(implicit F: FlatMap[F]): G[F[B]]` : `fa.traverse(..).map(_.flatten)` と同様、通常のListのflatMapをベースに考えるとわかりやすい
- `F[G[A]].sequence[G[_]: Applicative]: G[F[A]]` : `F[_]` と `G[_]` を転地させる

### cats.syntax.flatMap._ * flatMap族はMonadではなく、こちらにある
- `F[A].flatMap[B](f: A => F[B]): F[B]` : 値 `A` に関数 `A => F[B]` を適用したあと、返り値の `F[B]` の文脈を自分の文脈と合成する
- `F[A] >>= (f: A => F[B])` : flatMapのシンタックスシュガー
- `F[A] >> (f: => F[B])` : 両者の文脈を引き継ぎ、右側の値を持った `F[B]` を返す

`cats.syntax.all._` はcatsで定義されているすべての型クラスに対するシンタックスをimportする  

後は `cats.syntax` の仕組みについて、少しだけ触れよう  
動作からお気づきかもしれないが、`cats.syntax` は `implicit class` で実装されている、だいたいこんな感じだ  

```scala
import cats.Functor

implicit class FunctorOps[A, F[_]: Functor](fa: F[A]) {
  def map[B](f: A => B): F[B] = Functor[F].map(fa)(f)
}
```

`implicit class` `FunctorOps` は型クラスインスタンスを implicit 引数として受け取り、各関数で利用しているだけだ  
残念ながら、このコードはcats上ではマクロで生成されているため、リポジトリまで見に行っても実態を見ることはできないだろう  
(ただ、興味があれば [simulacrum](https://github.com/typelevel/simulacrum) について調べてみるといい、どういうヘルパーimplicit classが生成されているかの理解を助けることができるだろう)  

## 副作用(Side Effect)
入力が同じであれば、常に同じ値を返すような関数を純粋(pure)な関数と呼ぶ  
そして、入力が同じでも、外部の状態によって影響を受け、返り値が変わったり、外部に作用を起こすような関数を不純(impure)な関数と呼ぶ  
不純な関数が起こす外部への作用は一般的に副作用と呼ばれる  
副作用の例で言えば、ファイル入出力やDBアクセス、そして意外かもしれないが乱数生成機へのアクセスなども副作用だ、他にも沢山ある  
副作用は危険だ、副作用のある関数は動的に返り値を変えるし、外部の状態を変更するし、プログラムをクラッシュさせる可能性がある  
不純な(つまり副作用を持つ)関数は扱いづらい、たとえ内部の実装を知っていても入力値から返る値や変更される状態がを完全には推測できないからだ  
不純な関数を呼び出している関数は同じく不純な関数として扱われる、そう、不純は伝染し、結果的にプログラムの半分以上(もっとかもしれない)は  
不純な関数ということになるだろう  
プログラムが大きくなってきたときに、この不純な関数と純粋な関数を常に切り分けておくことは大事だ、明らかな外部要因によるエラーが発生した時(つまり、入力値は正しいと保証されている時)  
調査対象を不純な関数に絞り込むことができる  
さらに、不純な関数の中で起こす計算を最小限に抑えていれば(つまり、不純な関数は単純なデータの移動やハードウェアへの命令だけを書く)計算処理のバグがあった場合に  
調査対象を純粋な関数に絞り込むことができる  

実際に不純な関数を切り分ける手段としてCatsは `cats-effect` というモジュールを提供してくれている  

## Cats Effectの世界
### IO
cats-effectにおいて代表的に使われる「データ構造」だ  
失敗するかもしれないし、副作用を起こしており、非同期処理(jvm的に言うとmainスレッド以外で動いている処理だと思えばいい)を行なっているかもしれないという  
「文脈」を持つ(ちょっと詰め込みすぎだとは思う)  
その他にも副作用を扱いやすくするための関数や他の「データ構造」から変換することができる関数などのヘルパーも用意されている  
cats-effectの世界では副作用を持つ関数(もしくは副作用を持つ関数を呼び出す関数)はすべてIO型になっているべきであり  
純粋な関数と明確に「型で」区別される  

### Monad
データ構造 IO は「Monad」型クラスを実装している  
Monadは端的にいうと `fa: F[A]` と 関数 `f: A => F[B]` を受け取り、`fa` の文脈を引き継ぎつつ関数 `f` を適用できる、という振る舞い(flatMap)を持つ型クラスだ
実際にコードを見よう  

```scala
import cats.effect.IO

object Test {
  def case1: Unit = {
    val fa: IO[Int] = IO(3)
    // 本当であれば、純粋な関数をわざわざIOで包むようなことはしないが、例なので我慢して欲しい
    val f: Int => IO[String] = { i => IO(i.toString) }

    fa.flatMap(f) // => IO("3") * 実際には具象型のインスタンスが表示されただろう
  }

  def case2: Unit = {
    val fa: IO[Int] = IO(3)

    // 言うまでもないが、forシンタックスはflatMapのシンタックスシュガーだ
    val program = for {
      i <- fa
      _ <- IO { println(s"in: $i") }
      i2 = i +2
      _ <- IO { println(s"in: $i2") }
    } yield i2

    // unsafeRunSyncについてはあとで説明する、今はIOを評価するための手段と思えばいい
    val res = program.unsafeRunSync() // in: 3\n in: 5
    println(res) // => 5
  }

  def case3: Unit = {
    val fa: IO[Int] = IO(3)

    val program = for {
      i <- fa
      // 処理を失敗の文脈に推移させて見よう
      _ <- IO.raiseError { new RuntimeException("ops.") }
      i2 = i +2
      _ <- IO { println(s"in: $i2") }
    } yield i2

    // printlnへ到達する前に失敗の文脈に移ったため、標準出力は実行されない
    program
      .handleErrorWith { e => IO { println(e) } } // プログラムがクラッシュしないように、失敗の文脈から復帰しよう
      .unsafeRunSync() // => java.lang.RuntimeException: ops. (実行すれば、例外がthrowされたわけではないことに気づくだろう)
  }
}
```

ある程度scalaに触れた事のあるユーザならば、flatMapはもう身近な存在だろう  
flatMap自体はデータ構造 IOのメソッドしても定義されているので `cats.syntax` のimportは不要だ  
`case3` では失敗の文脈に推移させる方法を示した, これはjvmの例外機構によく似た動きを再現できるが、実態は  
失敗の文脈の引き継ぎによる処理のショートだ、OptionやEitherで言うNoneやLeftを返した時の挙動と同じだ  
`handleErrorWith` で成功の文脈へ復帰しているため特殊な処理に見えるが、これは `ApplicativeError` 型クラスで提供されている振る舞い  
を利用しているだけだ、つまりIOは `ApplicativeError` を実装している, ちょうどいいので紹介しておこう  

### ApplicativeError
`ApplicativeError` はデータ構造 `F[_]` に対して、失敗の文脈への推移や、失敗の文脈からの復帰の振る舞いを定義している  
そして名前が示す通り、`Applicative` を継承している  
使ってみよう  

```scala
import cats.ApplicativeError
import cats.syntax.applicativeError._
import cats.instances.either._

object Test {
  type StringEither[A] = Either[String, A]

  def case1: Unit = {
    val x: StringEither[Int] = Right(3)

    val m = for {
      px <- x
      // 奇数ならばエラー
      pxx <- if (px % 2 == 0) Right(px+1)
           // ApplicativeErrorはデータ構造 F[_] の他に失敗の型 E を要求する
           else ApplicativeError[StringEither, String].raiseError(s"$px is odd")
    } yield pxx

    val res = m.handleErrorWith { e =>
      println(e)
      Right(999)
    } // 3 is odd

    println(res) // => Right(999)
  }
}
```

先ほどIOで示したような例外機構の再現がEitherでもできた  
ApplicativeErrorはデータ構造 `F[_]` の他にもう一つ型 `E` を要求し、これが失敗の型に当てはめられる  
`ApplicativeError[Either[E, *], E]` の場合は `E` にユーザが任意の型を使用できるが
IOでは `Throws` しか使えないし、Optionでは `Unit` しか使えない、つまり、これらは特化されたエラー型に対してのみhandleとraiseの振る舞いを提供していることを示している  

### MonadError
MonadErrorはApplicativeErrorを進化させたものが、ApplicativeErrorとMonadを継承している  
新しく追加された振る舞いは `ensure` と `rethrow` だ、どちらも `F[A]` の値 `A` にアクセスし、判定処理をした上で失敗しているかもしれない文脈を持った `F[A]` を  
返す振る舞いだ  
なぜこれがApplicativeErrorで定義されていないのかを知るために、まずはMonadとApplicativeの違いについて説明しよう  

### MonadとApplicative
MonadはApplicativeを継承している、つまり、ApplicativeでできることはすべてMonadにも行えて、Monadにしか行えない振る舞いをしれば  
両者の違いを知ることができるだろう  
Monadは(またはflatMap関数は)データ構造 `F[A]` の値 `A` にアクセスし、値 `A` を使い、新しい文脈を生成するための判定処理を行うことができる  
これは値 `A` に依存して新しい文脈を生成できるということだ、コードで示そう  

```scala
import cats.Monad
import cats.ApplicativeError
import cats.instances.option._

object Test {
  // エラー型をUnitに固定
  type UnitError[F[_]] = ApplicativeError[F, Unit]

  // すでに奇数であれば失敗
  def toOdd[F[_]: Monad: UnitError](n: Int): F[Int] = if (n % 2 == 0 ) Monad[F].pure(n+1) else ApplicativeError[F, Unit].raiseError(())

  def case1: Unit = {
    val x: Option[Int] = Some(3)
    println(x.flatMap(toOdd[Option])) // => None
  }
}
```

関数 `toOdd` は純粋な値nに依存し、失敗するかもしれない文脈の値を返す  
これを `F[A]` に適用するためには内部の値 `n` を取り出したかのように振る舞えなければならない、そして、その値から生成された新しい文脈  
を引き継ぐことも必要だ  
対して、Applicativeは適用前にすでに文脈が定まってしまっている関数にしか値を適用できないので、値に依存して文脈を変更するような振る舞いはできないのだ  
これがMonadにしかできないことの説明だ (もし納得が行かないのであれば、上記テストをApplicativeで実装することを試みて欲しい)  

## 型シグネチャを読む
ここまで示してきた通り、型クラスは抽象的なデータ構造 `F[_]` に様々な振る舞いを要求する  
逆に言えば型クラスを実装しているデータ構造 `F[_]` はその振る舞いを行える、ということを示している  
これは関数の型上で表れ、それ以上の振る舞いができないことを表現することができる  
たとえば以下の二つの関数シグネチャでは片方は失敗するかもしれない処理、もう片方は決して失敗するこがない処理であることがわかる  

```scala
// 失敗することのない関数 f
def f[A, B, F[_]: Functor](fa: F[A])(f: A => B): F[B]

// 失敗するかもしれない関数 g
def nonFial[A, B, F[_]: MonadError](fa: F[A])(f: A => B): F[B]
```
* MonadErrorの実装を受け取って、全く使わないことも可能だが、それはunusedなvalueを受け取っているという意味でもそもそも避けるべきだ  

また、関数型プログラミングではデータ構造は文脈を表すことが多い  
あえて具象的な関数として定義することで、そのデータ構造が持つ文脈を明示できるだろう  

```scala
// 結果の値が存在しないかもしれない文脈を示す
def f(n: Int): Option[Int]

// 副作用が発生する文脈をしめす
def g(n: Int): IO[Int]
```

これら二つを正しく扱うことで、実装を読まずとも関数の型シグネチャから得られる情報は多くなる  
コードが大きくなればなるほど、全ての実装を読むのが困難になる  
そんな時に型情報が正しく整理されていれば、必要な情報を得るために読まなければいけなコードを絞り込むことができるだろう  
そういった理由から、常に型情報を正しく保つことは大事で、安易に広い型やハイコンテキストな型を使わない方がいいのだ  

---

関数型の話とcatsの基本的な話についてはここまでだ、ここまでの知識があれば  
cats自体は問題なく扱えるだろう、ただ、本ページではcats-effectの非同期性については一切話していない  
次ページではそれについて説明しようと思う

[Cats Effectの非同期/並列処理](./cats-effect-async.md)
