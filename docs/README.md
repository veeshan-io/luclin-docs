# Luclin2 Documents

```scala
def countChange(money: Int, coins: List[Int]): Int = {
  if (money == 0)
    1
  else if (coins.size == 0 || money < 0)
    0
  else
    countChange(money, coins.tail) + countChange(money - coins.head, coins)
}
```

|一|二|三|
|---|---|---|
|111|2222|```333```|

<!-- tabs:start -->

#### ** English **

Hello!

#### ** French **

Bonjour!

#### ** Italian **

Ciao!

<!-- tabs:end -->