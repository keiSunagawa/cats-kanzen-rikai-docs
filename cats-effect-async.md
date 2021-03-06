## 遅延評価(Lazy Eval)
データ構造 IO の評価戦略は遅延評価だ  
これが何を言っているかと言うと  
`IO[A]` は実行可能で値 `A` を返すタスクを内包するデータ構造だ  
コードで書くとこんな感じになる  

```scala
class IO[A](val run: () => A) {
  def flatMap[B](f: A => IO[B]): IO[B] = IO {
    val a = run()
    f(a).run()
  }
  def map[B](f: A => B): IO[B] = IO {
    f(run())
  }
}

object IO {
  // 名前渡しで引数を評価させないようにしている
  def apply[A](a: => A): IO[A] = new IO[A](() => a)
}

object Test {
  def case1: Unit = {
    val task = IO { println("input: 3"); 3 } // この時点では内包したタスクは実行されない

    val program = for {
      i <- task
      _ <- IO { println("task running..") }
      i2 <- IO { i + i }
      _ <- IO { println("task stopping..") }
    } yield i2

    val res = program.run() // input: 3\ntask running..\ntask stopping..
    println(res) // => 6

    // 同じタスクをもう一度実行することもできる
    program.run() // input: 3\ntask running..\ntask stopping..
  }
}
```

もちろん、実際にcatsで定義されているIO型はもっと複雑だし多機能だ  
だが、遅延評価の説明をするにはシンプルな方が良い  

見ての通りIOに渡されたtaskはその場で評価されず、最終的な `run()` の呼び出しまで遅延されている  
この評価戦略は副作用と相性がよく、定義されたタイミングでは評価されないことが保証されるため, IOタスクを  
「使い回す」ことができるのだ  
例えば上記の変数 `program` に追加のIOを合成したり、再帰的に `program` のタスクを複数回呼び出すこともできる  

```scala
// ここからは本物のIOを使おう, 型クラスインスタンスが定義されているため、扱いやすいからだ
import cats.effect.IO

object Test {
  val program = for {
      i <- IO { println("input: 3"); 3 }
      _ <- IO { println("task running..") }
      i2 <- IO { i + i }
      _ <- IO { println("task stopping..") }
    } yield i2

  def case1: Unit = {
    for {
      // program taskの計算結果を受け取る
      i <- program
      _ <- IO { println("continuation...") }
      ix <- IO { i + i }
    } yield ix

    val res = program.unsafeRunSync() // input: 3\ntask running..\ntask stopping..\ncontinuation...
    println(res) // => 12
  }

  def case2: Unit = {
    import cats.syntax.traverse._
    import cats.instances.list._

    val `3times` = (1 to 3).toList.traverse { _ => program }

    `3times`.unsafeRunSync() // input: 3\ntask running..\ntask stopping..\ninput: 3\ntask running..\ntask stopping..\ninput: 3\ntask running..\ntask stopping..
  }
}
```

もしtaskを定義した時点で処理が評価されてしまっていたら、このようなことはできないだろ  
しかし、IOの遅延評価はthunkと呼ばれるような「一度評価されると結果値をメモ化し、二度目以降は評価されない」ものではないので  
扱いには気をつけよう、ユーザがうっかり一度しか実行してはいけない副作用を二度実行してしまうことだって起こり得る(もちろん、そういうコードを書かないよう気をつけようという話だ)  

話がそれるが、catsには遅延評価を実装するもうひとつのデータ型として `cats.Eval` が存在する  
こちらは上記で話したようなメモ化なども実装しているため、副作用のない純粋な遅延評価だけを求めているのであれば、こちらをつかといい  
(なんとこのデータ型は内部的にtraverse関数でも使われているのだ、他にも最適化や安全性のために様々な関数で使われている"縁の下の力持ち"だ)

## IOタスクの実行
今まで見てきたようにIOタスクは `unsafeRunSync` メソッドを呼び出すことでも実行できる  
これはその名前の通り、unsafeで同期的(sync)な実行だ、もしIOが失敗の文脈で終了すれば例外を投げるし  
timeoutを設定せずに無限ループなIOタスクを実行した場合にはそのプログラムは永遠に終了しないだろう(ユーザは泣く泣く `^C` を押下するだろう)  
その物々しい名前が示している通り、本来はユーザが直接実行するものでなく、そうしなくても手段をcatsは提供してくれている  

### IOApp
IOAppは言うならばIOタスクの「処理系」だ  
ユーザはIOタスクを組み上げ、IOAppを継承したsinglton objectのrun関数の戻り値として返すだけでいい  
以下のような感じだ  

```scala
import cats.effect.{ IO, IOApp, ExitCode }

object Main extends IOApp {
  val program = for {
    _ <- IO { println("app start..") }
    _ <- IO { println("app end..") }
  } yield ExitCode.Success

  def run(args: List[String]): IO[ExitCode] = {
    program
  }
}

// IOAppはjavaで言う所の `public static void main` を実装している、つまりエントリポイントだ
Main.main(Array())
```

IOAppの実装は本物の「処理系」に依存する  
つまり、通常のscalaならjvm, scalajsならブラウザだ  
これが意味していることは、つまり非同期処理の実装が処理系により異なるということだ  
以降、本ドキュメントではjvmを前提として話を進める  

### 同期実行/非同期実行
同期実行は `IO#unsafeRunSync`, 非同期実行は `TO#unsafeRunAsync` で実行できる, `IO#unsafeRunAsync` は例外のハンドリングを要求し、戻り値はUnitだ  
つまり、結果を取得したい場合は `IO#unsafeRunSync` を使い、現在のスレッドでawaitする必要がある(これは非同期処理に置いては当然のことだが)  
`IO#unsafeRunSync` は必ずしも実行スレッドと同じスレッドでIOタスクされるわけはなく、どこかのスレッドで実行され、現在のスレッドで同期処理を行なっているだけなのだ  
どのスレッドで実行されるかは、IOタスク自身に依存する(つまり、IOタスク自身でどのスレッドで実行させるかを決定できるのだ、その方法は後続で示そう)

### unsafeToFuture
必要ならば、scala標準のFutureに変換することができる, `IO#unsafeToFuture` を使えばいい  
これはAkkaなど他の非同期機構と連携したいときに有効だ  
Futureはscalaプログラマには扱いなれた形だろうので本ドキュメントでは説明しない  
(実は筆者はFutureをあまり使ったことがないので説明できないのだ)  

### timeout
`IO#timeout` はIOタスクにタイムアウトを設定できる、設定したtimeout値を超えた場合は `TimeoutException` も持った失敗の文脈に遷移する(つまり `ApplicativeError.raiseError` だ)  
この関数は `implicit parameter` `Timer` と `ContextShift` を要求する、TimerはスケジューラのようなものでContextShiftはスレッドプールのようなものだ(ContextShiftについては後続で詳しく説明する)  
いずれも `IOApp` 内で「デフォルト」を定義されているので、特にこだわりがなければそちらを使うといい(もちろん、最適化を考えたとき、いずれ向き合うことになるものだ)  

```scala
import cats.effect.{ IO, IOApp, ExitCode }
import scala.concurrent.duration._
import scala.concurrent.TimeoutException

object Main extends IOApp {
  // TimerとContextShiftはIOApp内でimplicit valueとして定義されてるため、暗黙的に渡される
  val program = (for {
    _ <- IO { Thread.sleep(100) }.timeout(1.second) // 自身に設定されているtimeout内で終了すれば、次のIOにそのtimeout値を引き継ぐことはない
    _ <- IO { println("complete step1.") }
    _ <- IO { Thread.sleep(1000) }.timeout(10.millis)
           .handleErrorWith {
             case e: TimeoutException => IO.unit
             case e: Throwable => IO.raiseError(e)
            } // もちろん、タイムアウトエラーから復帰することも可能だ
    _ <- IO { println("complete step2.") }
    _ <- IO { Thread.sleep(Int.MaxValue) } // ほぼ無限時間Threadをブロックする
    _ <- IO { println("complete step3.") }
  } yield ExitCode.Success).timeout(3.second) // このIO taskブロック"全体"のtimeout値として見ることができる

  def run(args: List[String]): IO[ExitCode] = {
    program
  }
}

// timeoutを超えるため例外がなげられる!とても危険だ!
Main.main(Array())
```

意識して欲しいのはタイムアウトの設定は「型には現れない」ということだ  
アプリケーションの性質によるが、多くの場合においてタイムアウトは設定しておいたほうがいい  
ただし、呼び出し側からはみえない値なのでむやみに設定すればいいというものではない  
本当に必要な部分を除けば、原則 `unsafeRunSync` を呼び出す直前やIOAppの `run` メソッド内でのみ記述するといいだろう  

### ひとつのIOでひとつのアプリケーション
IOAppのrunメソッドのシグネチャが示す通り、大小様々なIOタスクを定義したとしても、最終的にはそれらを  
ひとつのIOタスクに合成することが大切だ、他のフレームワークとの同期などの止むを得ない状況を除き、不用意に  
`unsafeRunSync` を呼び出しているアプリケーションを書いているのであれば、それは間違った使い方をしている可能性があるので  
気をつけるべきだ。cats-effectはIOを組み合わせたり、互いに連携したりするために必要な関数をちゃんと提供しているので  
なにかやりたいことがあるときはまずはAPIを眺めて考えてみるのも有効な手だ  
https://typelevel.org/cats-effect/api/cats/effect/IO$.html
https://typelevel.org/cats-effect/api/cats/effect/IO.html

## 他のデータ構造への変換 - LiftIO
前項でひとつのIOタスクに合成することが重要だといったものの、ライブラリや他のフレームワークの兼ね合いでIOを  
他のデータ構造に埋め込む必要が度々発生するのは事実だ、そのための手段としてcatsは `LiftIO` という型クラスを提供している  

### LiftIO

```scala
// 例としてdoobieのデータ構造を使おう
import doobie._
import doobie.implicits._

import cats.effect.{ IO, LiftIO }

object Test {
  val ioProgram: IO[Unit] = for {
    _ <- IO { println("start.") }
  } yield ()

  val conIOProgram: ConnectionIO[Unit] = ???

  def case1: Unit = {
    val mergedProgram: ConnectionIO[Unit] = for {
      _ <- conIOProgram
      _ <- ioProgram.to[ConnectionIO] // IO を ConnectionIO に組み込む(lift)する
    } yield ()
  }
}
```

doobieはcats friendlyなDBアクセスライブラリだ  
`ConnectionIO` は通常のIOに加え、DBへのトランザクションの文脈も持ったデータ構造になる  
(さらに言うならFreeMonadによって実装されている、この辺はdoobieのページで説明しようと思う)  
ただし、IOとの継承関係が存在しているわけではないのでIOで行える操作は一切行えない  
この関係性の橋渡しをするのが `LiftIO` の振る舞いだ  
LiftIOは任意のデータ構造 `F[_]` に対してIOの文脈を組み込める振る舞いを提供する  
これはつまり、型の上では `IO[A] => F[A]` ができるという表現になる、事実LiftIOの型シグネチャはその通りだ  

```scala
import cats.effect.IO

trait LiftIO[F[_]] {
  def liftIO[A](ioa: IO[A]): F[A]
}
```

IOがどのようにデータ構造 `F[_]` に取り込まれるかは実装に委ねられている  
また `F[A] => IO[A]` の逆方向の変換は「提供しない」  
`ConnectionIO` の場合は `transact` 関数で再び `IO` に戻すことはできるが、他のデータ構造に関してはその限りではない  
つまり、組み込む方法も組み込まれた後の話もすべてデータ構造 `F[_]` の実装に委ねられているのだ  

LiftIOそれ自体をsummoner経由で使うこともできるが、`IO#to[F[_]: LiftIO]` というメソッドが用意されているため、そちらを使うのが一般的だろう  

## 安全なリソース使用 - Resource/Bracket
TODO

## スレッドシフト - ContextShift
TODO // 必要な知識を優先するため先にStreamの説明を進める

## 並列処理
TODO

[fs2.StreamによるStreaming処理](./fs2-stream.md)
