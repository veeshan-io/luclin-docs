# 领域业务模型

- 首选通过`implicit`注册各个基础方法
- 提供的专用`casetype`
  - `context`

## Context

操作业务，以一个带`__invoke()`方法的类来实现。`context`casetype是统一的，只有规范的几个操作接口。（隐式方法）

```php
$context = context(new Context(), [
  'rolename' => $rolevalue,
]);
```

## Role

定义`role`下命名空间下的各个`casetype`。把参数/模型、交互都给包了。

## Domain

## Item

这块是负责抽象出存储行为的，用于以后方便做缓存。
