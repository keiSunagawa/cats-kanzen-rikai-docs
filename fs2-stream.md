fs2については実は筆者も最近触り始めたばかりだ  
本来であれば実装や元となった理論もベースに説明したいのだが、今時点ではあまりにも乏しい  
なのでこのページではひとまず、今時点で持っている「基本的な操作」と「安全にfs2を使う方法」をベースに説明しようと思う  
(fs2のソースコードを読み、改めて理解が深まったときに書き直そうと思っている)

## Streamの世界
Streamは流れ続けるデータとそれに対する処理フローを表した概念だ  
fs2.Streamも基本は同じで、その概念を表した型だと思っていい  
以下は小さな有限で純粋なStreamの構築と、加工処理のサンプルだ  

```scala
import fs2.Stream

object Test {
  def case1: Unit = {
    val upstream = Stream.emits(List(1, 2, 3))

    val res = upstream.map { i => i + i }.compile.toList

    println(res) // => List(2, 4, 6)
  }
}
```

小さな有限で純粋なStreamはcollectionと同じだ、ただし、「全体に対する処理を書ける」「何度でも走査できる」ことを考えるとcollectionのほうが強力だ  
Srreamに有効なのは無限に流れ続けるデータ(あるいは有限だがとても大きなデータ)に対する処理を書きたいときだ  

ただ幸いにもStreamは抽象的に書かれているため、上流のデータの実態がなんであろうと下流は同じように定義できる  
つまり、下流を先に定義しておいて、あとから上流を結合することができる、部品として個々の流れを定義できるのだ  

以降、上流には小さな有限のStreamを使うが、それはいつでも無限のStreamに置き換えれるものと思って読んで欲しい  

### fs2.Streamの型
fs2.Streamの型は以下のように定義されている

```scala
class Stream[+F[_], +O]
```

`O` は要素型だ、通常のcollectionの要素型と同じだ  
高階型 `F[_]` はそのStreamを組み立てた(compile)ときになる最終的な型だ  

### compile
fs2.Streamは `compile` メソッドを呼び出すことで「組み立てる」ことができる  
fs2.Streamの実態は各命令を表したデータ構造を再帰的にNestしたものだ  
(例によって簡略化したコードを掲載しようと思ったが、僕の力不足で作れなかった、後日再度挑戦しようと思う)  

fs2.Streamにおける命令(flatMapやevalや `++` )を各メソッドで積み上げ、その命令列を順次解釈し、別のデータ構造 `F[_]` を組み立てるのがcompileメソッドの実態だ  
抽象的でわかりづらい話かもしれないが、ここで伝えたいことは  
- fs2.Streamは各メソッドを呼び出した時点ではデータの流れは発生せず、命令列を作っているだけ  
- `compile` メソッドも命令列を解釈し、別のデータ構造(例えばIO)を組み立てているだけ(この時点でもstream処理は実施されない)  
  (さらに正確に言うとcompileメソッドもCompilerOpsに包んでいるだけで実際はdrainメソッドなどを呼び出した後にデータ構造の組み上げが始まる)  
- 完成したデータ構造 `F[_]` は実行(IOならばunsafeRunSync)するとstream処理を行うタスクであることを意味する  

### Compiler
`Compiler[F[_], G[_]]` はfs2で用意されている `Stream[F[_], O]` から `G[O]` への変換をする振る舞いを持つ型クラスだ  
変換というのは先に示したStreamを「組み立て」てデータ構造 `G[_]` を作成することと同義だ  
ここで2つのデータ構造 `F[_]` `G[_]` が登場していることを疑問に思われるかもしれないが、基本的には `F[_] == G[_]` で扱われることが多い  
そして、実際にimplicit導出はほぼ以下のものが使われると思ってよい  

```scala
import cats.effect.Sync

implicit def syncInstance[F[_]](implicit F: Sync[F]): Compiler[F, F]
```

これは実際にfs2内部でCompier型のコンパニオンオブジェクトに用意されている定義だ  
これはデータ構造 `F[_]` に対してSync型クラスさえあればCompilerを導出できることを意味している  
IOはもちろんSync型クラスを実装しているので、`Compiler[IO, IO]` はこの定義によって導出されている  

### Sync
`Sync[F[_]]` はデータ構造 `F[_]` または `F[_]` を返す関数に対して、評価を遅延して `F[_]` を取得できる振る舞いを持つ型クラスだ  
これは `F[_]` にの文脈における副作用を一時停止(suspend)させている振る舞いを持つとも言える  
コードで示そう  

```scala
import cats.effect.Sync
import cats.effect.IO
import cats.Applicative

object Test {

  def unsafe[F[_]: Applicative](): F[Int] = {
    println("fetch by sideffect..")
    Applicative[F].pure { 3 }
  }
  def safe[F[_]: Applicative: Sync](): F[Int] = Sync[F].suspend {
    println("fetch by sideffect..")
    Applicative[F].pure { 3 }
  }

  def case1: Unit = {
    val task = unsafe[IO]() // fetch by sideffect..
    println("===")
    println(task.unsafeRunSync) // => 3
  }

  def case2: Unit = {
    val task = safe[IO]()
    println("===")
    println(task.unsafeRunSync) // fetch by sideffect.. => 3
  }
}
```

`case2` の方は `F[_]` に抽象化していた上で副作用を安全に遅延させられていることに気付くだろう  

---
理解を簡単にするために、以下データ構造 `F[_]` は必要でない限り `IO[_]` として書く  
つまり、 `IO[_]` で書いてあるものは全て `F[_]` に置き換えれるものとして読んで欲しい  

## Output
`Output` 型にユーザが触れることはないが、Stream処理の要素の粒度を理解するために説明しておこう  
Compilerによって組み上げられた `IO[_]` にはfetchタスクと変換タスクと畳み込み関数が含まれている  
このとき、畳み込み関数のみひとつ上位の存在として組み込まれており、一区切りの命令列の終端で毎回実施されるものだ  
この命令列の終端を表すのが `Output` 型だ  
つまり、実態のみの話をするとOutput型が組み込まれる粒度がStreamを流れる最大の粒度、すなわち一回のStreamでメモリにのるデータの粒度  
だと思って欲しい  

## 命令の列
「命令列」という用語を度々出しているが、これはStreamの構造を直感的に理解するために用いる抽象的な表現だ  
fs2.Streamの実態は前述した通り、「各命令を表したデータ構造を再帰的にNestしたものだ」これは連結リストのデータ構造によく似ている  

```scala
sealed trait List[+A]
object List {
  case class Cons[A](head: A, tail: List[A]) extends List[A]
  case object Nil extends List[Nothing]
}
```

Listは自身のデータ構造を再帰的にNestし、ひとつのデータ構造としてみたそれをそのまま列としてもみなせる  

```scala
// Cons(1, Cons(2, Cons(3, Nil))) ≒ [1, 2, 3]
```

fs2.Streamの命令列も、ひとまずこれと同じだと思って欲しい  

## Streamの操作
ここからは実装の話ではなく、使い方の話をしていこう  
どんなものがあるかの一覧はAPIを眺めれば理解できる話なので、ここでは実践でよく使うものをpickupして説明する  

### eval
`eval` は `F[A]` から `Stream[F[_], A]` を作り出すメソッドだ  
これは単一のIOタスクをStreamの文脈にliftしたいときに使える、また最上流(flatMapの外側という意味)で実施した場合はこれによって一つのOutputが作られる  
注意して欲しいのはevalでliftしたStreamの要素の粒度が `A` であるということだ  
(ちなみにflatMapの内側でもOutputは生成される、ただしStepという他のデータ構造にラップされており、畳み込み関数にはたどり着かず、後続の命令を実行する実装になっているのだ)  
たとえば `IO[List[X]]` のIOタスクをliftしたところで、`Stream[F[_], X]` にはならないということだ(何を当たり前のことを、と思う読者もいると思うが)  
型を合わせるだけならば  

```scala
import fs2.Stream
import cats.effect.IO

object Test {
  def case1: Unit = {
    val upstream: Stream[IO, List[Int]] = Stream.eval(IO { List(1, 2, 3) })

    val stream: Stream[IO, Int] = upstream.flatMap { xs => Stream.emits(xs) }
  }
}
```

と書けるが、これは `Stream.eval(IO { List(1, 2, 3) })` の時点でOutputが `List[Int]` で確定し、メモリに乗る粒度は `List[Int]` (つまりは `List(1, 2, 3)` )  
なので錯覚しないように気をつけよう(何を当たり前のことを、と思う読者もいると思うが)  

### evalMap
`evalMap` は単一の要素ごとに副作用を発生させつつ、変換をするメソッドだ  
これは `stream.flatMap(a => Stream.eval(???))` と同様だ(そして実装的にも同様だ)  

### ++ * append
`++` はふたつのStreamを結合するメソッドだ  
結合後のStreamに対して行った変換関数は両者のStreamに対して適用される  

```scala
import fs2.Stream

object Test {
  def case1: Unit = {
    val upstream1 = Stream.emits(List(1, 2, 3))
    val upstream2 = Stream.emits(List(4, 5, 6))

    val res = (upstream1 ++ upstream2).map { _ * 2}.compile.toList
    println(res) // => List(2, 4, 6, 8, 10, 12)

    // 以下は命令列/結果として同じになる
    assert((upstream1 ++ upstream2).map { _ * 2}.compile.toList == (upstream1.map { _ * 2} ++ upstream2.map { _ * 2}).compile.toList)
  }
}
```

### chunks
`chunkN` は名前の通りchunkを作り出すメソッドだ  
型で示すと `Stream[IO[_], A] => Stream[IO[_], Chunk[A]]` で、Chunkは引数の数値分の要素を保持する  
`Chunk[A]` は要素数固定の配列がさらに最適化されたようなものだ  
当たり前だが `chunkN` メソッドを呼び出した場合は、`Chunk[A]` がメモリに乗る粒度になり  
これを `stream.chunkN(100).flatMap { xs => Stream.chunk(xs) }` としたところでn要素がメモリに乗ることは変わらないので気をつけよう(当たり前だが)  
(むしろ情報をロストしているので僕は避けたほうがよいと思う)

### unfoldEval
`unfoldEval` は初期値 `S` と `S => IO[Option[(O, S)]]` を受け取り、`Stream[IO, O]` を作り出すメソッドだ  
これは僕がStreamの上流を独自で用意したいときによく使う  
`S => F[Option[(O, S)]]` は `S` からevalすべきIOを作り出し、次の `S` とセットで返す  
`F[Option[(O, S)]]` がNoneの場合はStreamの終端を表す  

```scala
import fs2.Stream
import cats.effect.IO

object Test {
  val source0 = List("a", "b", "c")
  def source(i: Int): IO[Option[String]] = IO { source0.lift(i) }
  def case1: Unit = {
    Stream.unfoldEval[IO, Int, String](0) { i => source(i).map { opt: Option[String] => opt.map { _ -> (i+1) } } }
      .evalMap { x =>
        IO { println("processed: " + x) }
      }.compile.drain.unsafeRunSync // processed: a\nprocessed: b\nprocessed: c
  }
}
```

これはlistの終端までindexアクセスを行い、終端までたどり着いたら終了をする例だ  
これは外部から継続してソースを取ってくる処理を書くために使える、メモリに乗る粒度はもちろんsourceの取得関数次第だ  
(試しにこれを `scala.io.Source.fromFile` などでIteratorを取得する処理に書き換えてみれば、よくわかるだろう)  

## Pure[_]型
Pure型は型をひとつとる高階型におけるボトム型だ(という言い方が正しいかはわからないが、そのように振る舞う)  
これは通常の型におけるNothingと同じだ  

```scala
import fs2.{ Pure, Stream }
import cats.effect.IO

object Test {
  def case1: Unit = {
    // Pureはボトム型(全ての型のサブクラス)なので Stream[IO, _] に型強制できる
    val s: Stream[IO, Int] = Stream.emit[Pure, Int](1)
    val i: Int = ??? //これはNothingの振る舞いと同じだ, `???` はNothingを返す関数だ
  }
}
```
