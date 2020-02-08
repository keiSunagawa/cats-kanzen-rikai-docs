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
同期実行は `unsafeRunSync`, 非同期実行は `unsafeRunAsync` で実行できる, `unsafeRunAsync` は例外のハンドリングを要求し、戻り値はUnitだ  
つまり、結果を取得したい場合は `unsafeRunSync` を使い、現在のスレッドでawaitする必要がある(これは非同期処理に置いては当然のことだが)  
`unsafeRunSync` は必ずしも実行スレッドと同じスレッドでIOタスクされるわけはなく、どこかのスレッドで実行され、現在のスレッドで同期処理を行なっているだけなのだ  
どのスレッドで実行されるかは、IOタスク自身に依存する(つまり、IOタスク自身でどのスレッドで実行させるかを決定できるのだ、その方法は後続で示そう)

### unsafeToFuture
必要ならば、scala標準のFutureに変換することができる, `unsafeToFuture` を使えばいい  
これはAkkaなど他の非同期機構と連携したいときに有効だ  
Futureはscalaプログラマには扱いなれた形だろうので本ドキュメントでは説明しない  
(実は筆者はFutureをあまり使ったことがないので説明できないのだ)  

### Timeout


### ひとつのIOでひとつのアプリケーション

## 他のデータ構造への変換 - LiftIO

## 安全なリソース使用 - Resource/Bracket

## スレッドシフト - ContextShift

## 並列処理
