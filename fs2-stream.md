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

## Compiler
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
