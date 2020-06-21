# Functional Programming in Luclin2

即使在业务描述层面，函数式编程的思路也能提供许多便利。例如不需要从一开始就架构一个庞大的类结构体系，也不用担心之前犯的错误在后期纠正时面临巨大的重构除错压力。你要面对的仅仅是针对类型构建一系列函数工具，通过拼装组合这些工具来实现业务目标。

> 当读完本章文档后可以回来品味这段话：在Luclin2的系统中`CaseClass`是可以用来实现类型和值的脱机转移的。而行为的脱机转移则需要用到下一章所说的领域业务模型`Context/Domain`机制来实现。

## 使用 CaseClass

`CaseClass`是Luclin2中函数式编程的核心，它就像一个壳，把需要处理的数据包装起来，赋与其额外的意义，提供对函数式编程的支持。例如你有个字串`"hello"`，把它变成一个CaseClass的过程看起来是这样：

```php
$case = casing('string', "hello");
```

`casing()`方法可以生成一个CaseClass，一般情况下一个Case会包括其类型和内部包含的具体值/变量引用。case的类型即`case type`，这是个需要记住的概念。部分情况下，可以获取一个没有包含值的case，这让它看起来更像其他语言中的符号`symbol`：

```php
$some = casing('some');
$none = casing('none');
```

> 需要注意的是，尽管Scala中的命名方式，并且有些接近Scala、OCaml、Haskell等中的概念，请仍就不要将他们完全划上等号，在之后的示例中你会看到不少区别。

一个特别的地方是，case是自带解包的，所有的case对象都有同样的解包方式：

```php
$case   = casing('string', "hello");
$value  = $case(); // $value = "hello"
```

默认的解包会返回原始值，可以创建一个带处理函数的case：

```php
$case = casing('double', 1, fn($v) => $v * 2);
$doubled = $case(); // $doubled = 2
```

处理函数可以支持多个参数，case内包含的值始终作为一个参数传入，比如改进一下：

```php
$case = casing('times', 1, fn($v, int $factor) => $v * $factor);
$tripled = $case(3); // $tripled = 3
```

**case不可以被嵌套**，因此当一个case作为值被放入另一个case中时，会解包原来的case中的value。这可以用于做便捷的case type转换：

```php
$caseA = case('a', 100, fn($v) => $v * 10);
$caseB = case('b', $caseA);
$newValue = $caseB(); // $newValue = 1000, not $caseA
```

> 当确实需要要一个case的处理函数中把值作为另一个case进行操作时，请在处理函数中自行调用`casing()`函数转换。

可以基于一个case进行继承创建，新的case会继承老case的type，计算后value以及处理函数：

```php
$case1 = casing('double', 1, fn($v) => $v * 2);
$case2 = casing($case1);
$doubled  = $case1(); // $doubled = 2
$again    = $case2(); // $again = 4
```

> 注意这里并没有真的去生成一个链式调用，当执行`$case2 = casing($case1)`的时候已经计算过一遍$case1中的值了。

### 原始类型 raw case

有时我们只是想把一些基础的类型封装起来以使用函数式编程能力，这时可以使用`raw()`函数对值/变量进行包装。

```php
$case = raw('hello');
```

这时得到的`case type`是`string`。这个类型分析机制从实用角度出发，与php原生的类型并不完全一致，下面列出当前的转换规则。

```php
function type($var): ?string
{
    if (is_string($var))    return "string";
    if (is_numeric($var))   return "numeric";
    if (is_bool($var))      return "boolean";
    if (is_null($var))      return "null";
    if (is_iterable($var))  return "iterable";
    if (is_resource($var))  return "resource";
    if (is_object($var))    return "instance";
    return null;
}
```

!> Luclin2正在开发中，这部分转换规则可能会有所变动，待1.0 release后才会正式固定。欲了解具体计算方式请查看项目中`casetype()`函数源码。

### case type 的获取和比较

使用以下方法获取和比较一个case的type

```php
$case1 = raw(1);
$case2 = casing('numeric', 2);

$case1->is($case2->type); // true
```

### case 的字串化

一个case可以转换为字串，转换后的约定结构如下：

```php
$some = casing('some', 100);
$none = casing('none');

echo "$some"; // output ":some(100)"
echo "$none"; // output ":none"
```

对转换后的字串可以直接通过`casing()`函数恢复为case，但不支持对处理函数做恢复：

```php
$some = casing('some(100)');
$none = casing('none');
```

如果case中包含的数据是复杂结构，在序列化的时候该结构自身需要实现`__toString()`方法。而在恢复的时候则需要在该case type下事先注册约定的隐式方法`_restore`，这样当恢复时其解析出来的值会被正确还原：

```php
// 创建示范对象
class Url {
    public string $url;

    public function __construct(string $url) {
        $this->url = $url;
    }

    public static function restore(string $str): self {
        return new self(urldecode($str));
    }

    public function __toString() {
        return urlencode($this->url);
    }
}

$url = new Url('https://www.php.net/');

// 隐式方法声明
implicit('url')
    ->_restore(fn(string $str) => Url::restore($str));

// 创建case并序列化
$case = casing('url', $url);
$serialized = "$case"; // $serialized = ':url(https%3A%2F%2Fwww.php.net%2F)'

// 反序列化还原case
$case = casing($serialized);
$url = $case(); // same as new Url('https://www.php.net/')
```

> 隐式方法相关操作见后文详述。

## 模式匹配 Match

有了CaseClass就可以通过模式匹配来实现流程控制，在这里我直接举一个现实一些的例子，也可以看出和其他直接支持FP的语言的差异性。

?> 假设这里需要实现查询器生成，当用户传入`name=Lucy`时作为相等条件查询；传入`name=:none`时查询`name is null`；而传入`name=:some`的时候查询`name is not null`。演示所用查询器借用Laravel库的语法。

```php
$field  = 'name';
$value  = ':some'; // or ':none' or 'Lucy' ..etc

$query = Models\User::query();

$match = match()
    ->some(fn($_) => $query->whereNotNull($field))
    ->none(fn($_) => $query->whereNull($field))
    ->_(fn($case) => $query->where($field, $case()));
$query = $match($value);

$user = $query->first();
```

上面的代码定义了一个模式匹配，得到了一个闭包。在具体需要匹配的场合，可以根据case执行相应的代码分支。

> 函数式编程中相对通用的约定，将`_`命名作为通配或忽略的变量使用。这里的`$_`即明确表示该参数不在函数体中被使用。当然由于php的特性也可以直接不写该变量，但是在某些必填变量的情况下推荐使用此规则。

`match()`方法支持接收一个数组作为初始环境参数，这个参数会在匹配成功后作为末尾参数传给调用闭包：

```php
$match = match([
    'userId'  => 1,
])->male(fn($_, $context) => "User {$context['userId']} is male.")
    ->female(fn($_, $context) => "User {$context['userId']} is female.");

$sex = casing('female');
echo $match($sex); // output "User 1 is female."
```

`match()`生成的闭包支持多个参数，其中第一个参数是作为匹配对象传入，并在匹配成功后作为第一个参数传递给匹配上的函数。如果有更多参数，而这些参数会被带在匹配对象之后，context之前传入：

```php
$match = match(['userId' => 1])
    ->show(fn($case, $p1, $p2, $context) => [$case, $p1, $p2, $context]);
$result = match(casing('show'), 'p1', 'p2');
```

上面例程会把传入的case，"p1"和"p2"的字串以及context组合成一个数组一同返回。

### 封闭匹配

查询器生成的例子中是一个封闭匹配，在所有的选项最后加上`_()`下划线作为`case type`匹配，表示当上述所有type都未命中时，执行此分支的函数。

当一个match未处于封闭状态时，所有匹配都未命中时会抛出`\UnexpectedValueException`异常。

### 特殊类型 :nothing

使用`match()`方法匹配的对象往往都是case，但也允许使用非case的值/变量进行匹配。这种情况下会使用`casetype()`函数获取匹配的对象的分类，**而不会将其包装成一个case**。当casetype无法在规则内获取其类型返回null时，会将其作为一个名为`nothing`的特殊case type参与匹配。

### 函数列表 funcs 与 take()

事实上，`match()`之后通过overloading机制实现的动态调用方法指定的匹配参数是支持多个的。也就是说你可以传入一组函数列表，Luclin2中使用`funcs`表述，`funcs`内的函数将被依次调用。

```php
$match = match()->double(fn($case) => casing($case), fn($case) => casing($case));
$case = casing('double', 1, fn($v) => $v * 2);
echo $match($case)(); // output 4
```

这里要重点提到的是对`funcs`的执行支持是基于`take()`函数实现的，`take()`接收一个`funcs`，一个值及一堆参数。这个值将作为第一个参数传入函数，之后是同一套参数列表。在每个函数处理结束后，该函数的返回值会作为新的值代替旧值传递给下一个函数，而后面跟的参数列表不变，直到列表执行结束，返回最后的结果。

```php
$funcs = [
    fn($v, $factor)     => $v + $factor,
    fn($v, $_, $factor) => $v - $factor,
    fn($v, $factor)     => $v * $factor,
    fn($v, $_, $factor) => $v / $factor,
];

echo take($funcs, 10, [5, 3]); // output 20
```

!> `take()`这一机制在Luclin2的很多系统中都会用到，需要重点理解。

## 隐式方法 Implicit Method

使用`implicit()`函数可以在某一`case type`上声明一组隐式方法，当隐式方法声明起到程序结束会一直有效。通过该机制结合case，可以方便地给各类对象/变量/值赋于其原不存在的方法。

```php
implicit('string')
    ->length(fn($case) => strlen($case()));

$case = raw('hello');
echo $case->length(); // output 5
```

隐式方法的参数传递机制和`match()`一样，$case本身会作为第一个参数传入，而调用时赋与的参数则紧跟其后。另外，隐式方法声明的其实也是一个函数列表`funcs`，所以你可以作为多个参数为一个隐式方法注册一组函数。

> 注意，如果在一个case type下连续注册同名方法，后面的注册函数会覆盖之前注册的，而不会在`funcs`中追加。你可以一次调用注册一组函数，但不能分批追加。

### 管道方法调用 Pipeline

如果你试着使用case作为返回值，那么你已经摸到了实现管道调用的门道。首先根据case type声明各自的隐式方法，之后在方法中明确返回的是什么case type，这样可以自然而然地实现pipeline机制。我们把上面的方法改进一下，增加一个获取字串长度后将字串长度乘以5的需求。

```php
implicit('string')
    ->length(fn($case) => raw(strlen($case())));
implicit('numeric')
    ->times(fn($case, int $factor) => raw($case() * $factor));

echo raw('hello')
    ->length()
    ->times(5)(); // output 25
```

看起来非常自然。在这里可以看到`raw()`为自然地创建case带来的便利，以及最后`times()`返回的其实也是一个case，因此如果要输出实际结果的话，还需要多加一个`()`。不用觉得奇怪，这个表示在一些语言中也有使用作为`Unit`的概念，而在`reason`中甚至使用`()`来作为自动柯里化终止的占位符。

### 封闭隐式方法

当尝试调用一个未注册的隐式方法时会抛出`\OutOfBoundsException`异常，除非注册了`_()`方法。这通常用来基于现有系统进行兜底。拿Laravel举例，该框架本身就提供了许多辅助方法，如果case也能利用起来就好了，不需要全部都封装一遍：

```php
use Illuminate\Support\Str;

implicit('string')
    ->_(fn($case, $method, ...$params) => raw(Str::$method($case(), ...$params)));
```

完美搞定，这样当调用不存在的隐式方法时就会去找Str处理，再报错就交给框架了。当然这部分也可以写得很细，比如同时支持多个基础库方法，或是自定义报错类型，这都是可以在一个函数内处理的事——一个不行就两个😎

## 使用函子 Functor

Luclin2的函数库提供的函子支持相对狭义，其主要目标是能把一个函数`fn(x) => y`应用到一个群上。那么这个群通常来说可以理解为一个集合，或是一个字典，等等等等。因此在Luclin2中，将Functor拆解成两个部分：函数`function`与解包器`handler`

?> 还是举个粟子，试想上面我们已经有了计算出一个字串长度的函数，也有了把数字乘倍数的函数，并且做过了一次组合运用。现在需要对数据库中取出的用户模型集合中每个用户名应用这套逻辑并返回结果集。如果我们继续用Laravel框架，会发现这里至少需要穿透两层不是我们自己缔造的封装：第一层查询结果集Collection，第二层Model对象。我们当然可以手动在**事发地**写循环Collection的代码，也可以在**Model类内**增加一个方法来实现类似的功能。但是这样具有明显侵入性和低复用性的套路明显不够函数式。

来看看Luclin2中的函子组合之前所述特性来如何应对吧！假设模型对象`$user->name`可以取得用户名，那么定义一个叫`user`的case type，并注入隐式方法：

```php
implicit('user')
    ->nameLengthTimes(fn($case, $factor) => raw($case()->name)
      ->length()
      ->times($factor));

$user = new class{public string $name;}; // 模拟取出的Model User
$user->name = 'Triss';
echo casing('user', $user)->nameLengthTimes(5)(); // output 25
```



result
thought
functor

## 柯里化 Currying

