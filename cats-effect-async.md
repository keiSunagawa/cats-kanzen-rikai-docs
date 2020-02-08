## 遅延評価(Lazy Eval)
データ構造 IO の評価戦略は遅延評価だ  
これが何を言っているかと言うと  
`IO[A]` は実行可能で値 `A` を返すタスクを内包するデータ構造だ  
コードで書くとこんな感じになる

```scala
case class IO[A](run: () => A)

object Test {
  def case1: Unit = {
    val task = IO { () => println(3) } // この時点では内包したタスクは実行されない

    task.run() // => 3
  }
}
```
