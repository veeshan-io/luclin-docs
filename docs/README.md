# Luclin2 Documents

`Luclin2`是一个PHP的开发库，提供了企业级业务描述的必要组件，并对函数式编程和LSP协议提供支持。

## 安装

```bash
composer require veeshan/luclin2
```

!> 注意`composer.json`中需要开启`"minimum-stability": "dev"`，目前持续开发中不打版本号。也可在`composer.json`的`require`配置中加入`"veeshan/luclin2": "dev-master"`

<!-- ```scala
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
|111|2222|```333```| -->

<!-- tabs:start -->

#### ** English **

Hello!

#### ** French **

Bonjour!

#### ** Italian **

Ciao!

<!-- tabs:end -->