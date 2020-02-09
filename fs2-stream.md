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
fs2.Streamの実態は各状態を表したデータ構造を再帰的にNestしたものだ  
実際に定義されているものはとても複雑なので、この直感を掴むためにも簡略化されたものを定義しよう  

// だいぶきびしいかも。

```scala
sealed trait Command[F[_], O, O2]
case class Pure[F[_], O]() extends Command[F, O, O]
case class FMap[F[_], O, OM, O2](f: O => OM, next: Command[F, OM, O2]) extends Command[F, O, O2]

import cats.Monad
import cats.syntax.flatMap._
import cats.syntax.functor._
import cats.syntax.applicative._

case class Stream[F[_], O, O2](source: F[Option[O]], cmds: Command[F, O, O2])

object Stream {
//  def eval()

  def compile[O, O2, L, F[_]: Monad](stream: Stream[F, O, O2], fold: (L, O2) => L, acc: L): F[L] = {
    stream.source.flatMap {
      case Some(s) => compileLoop(s, stream.cmds).flatMap { result =>
        compile(stream, fold, fold(acc, result))
      }
      case None => acc.pure[F]
    }
  }
  def compileLoop[O, O2, F[_]: Monad](s: O, cmds: Command[F, O, O2]): F[O2] = {
    cmds match {
      case Pure() => s.asInstanceOf[O2].pure[F]
      case FMap(f, next) => compileLoop(f(s), next)
    }
  }
}

object Test {
  def case1: Unit = {
    var source0: List[Int] = List(1, 2, 3)
    val source: IO[Option[Int]] = IO {
      val h = source0.headOption
      source0 = if (source0.isEmpty) Nil else source0.tail
      h
    }

    val stream = Stream[IO, Int, Int](source, FMap((o: Int) => o + 1, Pure()))
    val res = Stream.compile(stream, (xs: List[Int], b: Int) => b :: xs, Nil)

    println(res.unsafeRunSync)
  }
}
```

