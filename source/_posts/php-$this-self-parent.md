title: PHP中的$this、self和parent
date: 2016-11-12 21:24:13
tags: 
    - php
categories:
    - 知道
---

> 可继承性是面向对象编程的重要特征。在开发过程中，由于类与类之间继承关系，我们很容易弄混对象属性的真实归属，到底自己代码所调用的是父类属性还是子类属性呢？这里针对PHP语言，讨论一下`$this`、`self`和`parent`这三个关键字的作用。

## 基本概念

> 声明一下，以下的所有描述都是基于PHP的`E_STRICT`模式，也希望大家的PHP
> 代码运行于`E_STRICT`模式下，否则代码很容易产生二义。

首先，`$this`、`self`和`parent`只能在类定义内部使用。

`$this`是一个伪变量，不能也不需要手动赋值。在类内部定义方法时，`$this`自动指向了主叫对象，即是主叫对象的引用。如果没有实例化对象，`$this`就是空值；在静态方法中`$this`一直是空值，不管是被类名调用，还是被实例化对象调用。

与`$this`不同，`self`不是一个变量，不能作为参数传递予函数。在类定义内部，`self`指向了当前的类。无关有没有实例化对象，`self`是当前类的引用。在代码上，使用`self`的地方，都可以用当前类名代替，且运行结果不会改变；然而，使用`$this->`的地方，如果是在调用类方法（不是类属性），也可以用`self::`代替，虽然语法上没有错误，但是逻辑上会有差异，详见下面 **$this与self**。

`parent`也不是一个变量，也不能作为参数传递予函数。在类定义内部，`parent`指向了当前类的直接父类，如果当前类没有父类，使用`parent`则会报错。

<!--more-->

## $this与self

对于`$this`与`self`，一般认为：`$this`引用类的实例化对象，可以调用类的所有属性和方法；而`self`引用类本身，与实例化对象无关，只能调用静态资源（如类常量、静态属性和静态方法）。
但是实际上`self`可以“踩过界”——`self`也可以调用非静态资源。

### 分析

在类定义内部， `$this->`后面可以连接当前类的所以类属性和类方法，但不能连接类常量；而`self::`后面可以连接类常量、静态属性和 _所有类方法_。
`self`可以调用非静态方法，实际上是要借用当前类的实例化对象来调用的。
在静态方法中，`self`不能调用非静态方法，正如`$this`不能被使用；而在非静态方法中，`self`就可以调用其他非静态方法了，正如`$this`可以正常被使用。那么，在非静态方法中调用其他非静态方法，使用`$this->`和使用`self::`有什么区别呢？

```php
<?php
    class A {
        public function name() {
            echo 'A';
        }
        public function thisName() {
            $this->name();
        }
        public function selfName() {
            self::name();
        }
    }

    class B extends A {
        public function name() {
            echo 'B';
        }
    }

    $b = new B();
    $b->name();     // echo B;
    $b->thisName(); // echo B;
    $b->selfName(); // echo A;
```

对于调用非静态方法，可以这样来总结`$this`和`self`的区别：`$this`是直接指向 _运行时_ 的实例化对象的，而`self`是借用 _定义时_ 的实例化对象。（`self`是不能调用非静态属性的，所以这一点不需要讨论`$this`与`self`的区别）

### 使用

假设父类定义了一个非静态方法`a`，里面调用了另一个非静态方法`b`：
- 如果希望子类的实例化对象调用`a`时可以使用子类自身的`b`，那么，父类的`a`定义时应该使用`$this->b();`；
- 如果希望子类的实例化对象调用`a`时可以使用父类的`b`，那么，父类的`a`定义时就应该使用`self::b();`。

## $this与parent

在类定义内部，`parent`指向了当前类的直接父类。`parent`和`self`的意义不同，但是性质上是几乎一样的。

### 分析
`parent::`后面可以连接直接父类的常量、静态属性和 _所有类方法_。
`parent`可以调用非静态方法，实际上是借用直接父类的实例化对象来调用的。
直接看代码：

```php
<?php
    class A {
        public function name() {
            echo 'A';
        }
        public function thisName() {
            $this->name();
        }
        public function selfName() {
            self::name();
        }
    }

    class B extends A {
        public function name() {
            echo 'B';
        }

        public function parentName() {
            parent::name();
        }
        public function bparentName() {
            parent::name();
        }
    }

    class C extends B {
        public function name() {
            echo 'C';
        }

        public function parentName() {
            parent::name();
        }
    }
    
    $c = new C();
    $c->name();        // echo C;
    $c->parentName();  // echo B;
    $c->bparentName(); // echo A;
```

对于调用非静态方法，`parent`是借用 _定义时_ 的直接父类的实例化对象。（`parent`也是不能调用非静态属性的）

### 使用

## 后期静态绑定

性质上，`self`和`parent`可以归为一类，其实它们还有一个同类——`static`。范围解析操作符`::`，除了类名，就只有这三个关键字可以使用。

### 分析
上面 **$this与self** 的讨论是基于对非静态方法的调用，如果是对静态方法的调用呢？还是直接看代码：

```php
<?php
    class A {
        public static function name() {
            echo 'A';
        }
        public static function staticName() {
            static::name();
        }
        public static function selfName() {
            self::name();
        }
    }

    class B extends A {
        public static function name() {
            echo 'B';
        }
    }

    B::name();       // echo B;
    B::staticName(); // echo B;
    B::selfName();   // echo A;
```

这里`static::`的作用，叫做 **后期静态绑定**，用于在继承范围内引用静态调用的类。从语言内部角度看，“后期绑定”的意思是说：`static::`不被解析为 _定义时_ 当前类，而是在实际 _运行时_ 的使用类。`static::`后面只能连接静态属性和静态方法，弥补了`$this`与`self`的不足。

### 使用

假设父类定义了一个静态方法`a`，里面调用了另一个静态方法`b`：
- 如果希望子类调用`a`时可以使用子类自身的`b`，那么，父类的`a`定义时应该使用`static::b();`；
- 如果希望子类调用`a`时可以使用父类的`b`，那么，父类的`a`定义时就应该使用`self::b();`。

## 最后

所以在定义类的时候，应该使用`$this`的地方就使用`$this`，应该使用`self`的地方就使用`self`，`parent`估计会很少用到，总之不要混着用了。

## 参考
-[类与对象-基本概念](http://php.net/manual/zh/language.oop5.basic.php)
-[类与对象-范围解析操作符](http://php.net/manual/zh/language.oop5.paamayim-nekudotayim.php)
-[类与对象-后期静态绑定](http://php.net/manual/zh/language.oop5.late-static-bindings.php)
-[类与对象-基本概念](http://php.net/manual/zh/language.oop5.basic.php)

