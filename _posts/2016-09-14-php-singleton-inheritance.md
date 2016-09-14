---
layout: post
title: PHP 单例模式实现继承的坑
date:  2016-09-14
author: kuga
---

自 PHP 5.3.0 起，PHP 增加了一个叫做后期静态绑定的特性(Late Static Binding)。
有了这个特性，实现单例模式的继承就变得简单得多了。
虽然这里有一个坑，但不并阻碍 PHP 成为世界上最好的语言 :)

网上(某度)最常见的实现方式如下。

```php
<?php

class A
{
    public static $instance = null;

    public static function getInstance()
    {
        if (is_null(static::$instance)) {
            static::$instance = new static;
        }
        return static::$instance;
    }
}

class B extends A
{
    public static $instance = null;
}

class C extends A
{
    public static $instance = null;
}

$b = B::getInstance();
$c = C::getInstance();
var_dump($b === $c);  // return false
```

一眼看上去是没问题的，的确每个类拥有各自的一个静态实例。
但是我们一般使用的继承是不需要在子类里重新声明继承变量的。
如果把子类 B 和 C 中声明的静态变量去掉，再运行一下代码，你会发现结果是 true。
因为此时 $b 和 $c 都引用了同一个静态变量，这个变量就是 B 的一个实例。
而且如果修改了这个静态变量，就会影响到其它子类，这是很危险的。
想像一下如果这个静态变量是一些数据库配置，这是要出人命的啊！

有人可能会说，不就一个变量声明吗，记得写就是了。
前提是如果这个项目不只有你一个人，那就是留坑了，早晚有人会跳下去的。
我们要做的不是告诉别人这里有坑你要小心，而是直接把他干掉。

解决方法也不是什么高级技巧，Google 一下 PHP Singleton Inheritance，
第一页就会有答案了，我是在 Stack Overflow 里看到的，这里直接贴一下代码好了。

```php
<?php

abstract class Singleton
{
    protected function __construct()
    {
    }

    final public static function getInstance()
    {
        static $instances = array();

        $calledClass = get_called_class();

        if (!isset($instances[$calledClass]))
        {
            $instances[$calledClass] = new $calledClass();
        }

        return $instances[$calledClass];
    }

    final private function __clone()
    {
    }
}

class FileService extends Singleton
{
    // Lots of neat stuff in here
}

$fs = FileService::getInstance();
```

get_called_class 也是 5.3.0 之后的新函数。
这样处理的话就不用在子类声明变量了，他能根据不同子类的名称自动管理这些实例。

## 引用
------

1. <a href="http://stackoverflow.com/questions/3126130/extending-singletons-in-php" target="_blank">http://stackoverflow.com/questions/3126130/extending-singletons-in-php</a>
