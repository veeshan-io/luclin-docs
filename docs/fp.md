# Functional Programming in Luclin2

即使在业务描述层面，函数式编程的思路也能提供许多便利。例如不需要从一开始就架构一个庞大的类结构体系，也不用担心之前犯的错误在后期纠正时面临巨大的重构除错压力。你要面对的仅仅是针对类型构建一系列函数工具，通过拼装组合这些工具来实现业务目标。

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

可以基于一个case进行迭代创建，新的case会继承老case的type，计算后value以及处理函数：

```php
$case1 = casing('double', 1, fn($v) => $v * 2);
$case2 = casing($case1);
$doubled  = $case1(); // $doubled = 2
$again    = $case2(); // $again = 4
```

> 注意这里并没有真的去生成一个链式调用，当执行`$case2 = casing($case1)`的时候已经计算过一遍$case1中的值了。

### 原始类型 case

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

### case type 获取和比较

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
$url = new class('https://www.php.net/') {
    public string $url;

    public function __construct(string $url) {
        $this->url = $url;
    }

    public function __toString() {
        return urlencode($this->url);
    }
};

// 隐式方法声明
implicit('url')
    ->_restore(fn(string $str) => urldecode($str));

// 创建case并序列化
$case = casing('url', $url);
$serialized = "$case"; // $serialized = ':url(https%3A%2F%2Fwww.php.net%2F)'

// 反序列化还原case
$case = casing($serialized);
$urlStr = $case(); // $urlStr = 'https://www.php.net/'
```

> 隐式方法相关操作见后文详述。

## 模式匹配

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

> 函数式编程中相对通用的约定，将`_`命名作为通配或忽略的变量使用。



### 封闭匹配

## 隐式方法

### Pipe

## 使用函子

result
thought
functor

## 其他支持

### 批量应用一组函数

### 柯里化
