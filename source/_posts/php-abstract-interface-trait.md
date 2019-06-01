title: PHP中的abstract、interface和trait
date: 2016-12-30 17:04:13
tags: 
    - php
categories:
    - 知道
---

> 面向对象编程，有几个重要特征：继承性，抽象性，多态性等等。这里针对PHP语言，讨论一下`abstract`、`interface`和`trait`，其他语言可能有偏差。

## abstract

### 抽象类

在面向对象的概念中，所有的对象都是通过类来描绘的。具体类是实例对象的抽象化，而 **抽象类** 则是具体类的抽象化。
定义一个具体类，使用关键字`class`；定义一个抽象类，要带上关键字`abstract`，即`abstract class`。PHP语言的机制比较宽松，抽象类可以继承于其他抽象类，也可以继承于具体类，但是抽象类没有直接的实例化对象。

```php
<?php
    abstract class AbstractClass1
    {
     // 强制要求子类定义这些方法
        abstract protected function getValue();
        abstract protected function prefixValue($prefix);

        // 普通方法（非抽象方法）
        public function printOut() {
            print $this->getValue() . "\n";
        }
    }

    class ConcreteClass1 extends AbstractClass1
    {
        protected function getValue() {
            return "ConcreteClass1";
        }

        public function prefixValue($prefix) {
            return "{$prefix}ConcreteClass1";
        }
    }

    abstract class AbstractClass2 extends ConcreteClass1
    {
        
    }
```

<!--more-->

### 抽象方法
抽象类中可以设置属性，定义具体方法，特别，还可以定义 **抽象方法**。定义定义一个抽象方法，需要使用关键字`abstract`，并且只能在抽象类中定义。抽象类中，只能声明抽象方法其调用方式（参数），不能定义其具体的功能实现；而在抽象类的具体子类中，**一定要**将父类中的所有抽象方法具体化：
- 具体方法不再带`abstract`声明；
- 具体方法的访问控制必须和父类中一样（或者更为宽松）；
- 具体方法定义的调用方式必须与抽象方法的声明匹配，即所需参数的数量、类型必须一致。如果要增加参数，只能在末尾追加，并且只能是可选参数。



## interface

首先，接口不是类，既不是具体类，也不是抽象类，接口的定义就没有用到关键字`class`。定义一个接口，使用关键字`interface`：

```php
<?php
    interface a
    {
        const b = 'Interface constant';
        public function foo();
    }
    interface b
    {
        public function bar();
    }

    interface c extends a, b
    {
        public function baz();
    }

    class ConcreteClass implements c {
        public function foo(){}
        public function baz(){}
        public function bar(){}
    }
```

接口有以下几个特点：
- 接口不能被实例化，因为接口不是类；
- 接口可以继承于其他接口，并且可以多重继承，但不能继承于类；
- 接口中不能设置属性，但是可以设置常量；
- 接口的方法只能是公有的，并且是只声明不定义；
- 具体类、抽象类都可以使用关键字`implements`来实现（多个）接口；
- 具体类中一定要具体化所有要实现的接口方法，并且定义的调用方式必须与接口中所声明的匹配，即所需参数的数量、类型必须一致。如果要增加参数，只能在末尾追加，并且只能是可选参数。


## trait

`trait`，中文意思是 **特点**，算是PHP的一个新语法（自PHP5.4加入）。PHP本身是不支持多重继承的，`trait`就是为了减少单继承语言的限制而提出，开发人员可以通过`trait`自由地在不同层次结构内独立的类中复用代码。
`trait`示例：

```php
<?php
    trait A {
        public $a = 1;
        public function say() {
            echo 'a';
        }
        abstract public function selfSay();
    }
    trait B {
        use A;
    }

    class C {
        use B;
        public function selfSay() {
            echo 'c';
        }
    }
```

`trait`有以下几个特点：
- `trait`不是类，无法通过自身来实例化，也不能继承或者被继承；
- 但是`trait`中，可以定义属性，定义具体方法，还可以声明抽象方法；
- 正如`class`能够使用`trait`一样，`trait`也能够使用其他`trait`。在`trait`定义时可以使用一个或多个`trait`。

使用`trait`的时候还要注意：
1. 优先级。从基类继承的成员会被 `trait` 插入的成员所覆盖。优先顺序是来自当前类的成员覆盖了`trait`的方法，而`trait`则覆盖了被继承的方法。
2. 多个`trait`。通过逗号分隔，在`use`声明列出多个`trait`，可以都插入到一个类中。
3. 冲突问题。如果引入的两个`trait`都定义了一个同名方法，且没有明确的解决冲突，那么代码将会产生一个致命错误。为了解决多个 trait 在同一个类中的命名冲突，可以使用`insteadof`操作符来明确指定使用冲突方法中的哪一个，或者使用`as` 操作符将其中一个冲突的方法以另一个名称来引入：

    ```php
    <?php
        trait A {
            public function smallTalk() {
                echo 'a';
            }
            public function bigTalk() {
                echo 'A';
            }
        }
        trait B {
            public function smallTalk() {
                echo 'b';
            }
            public function bigTalk() {
                echo 'B';
            }
        }
        class Talker {
            use A, B {
                B::smallTalk insteadof A;
                A::bigTalk insteadof B;
            }
        }
        class Aliased_Talker {
            use A, B {
                B::smallTalk insteadof A;
                A::bigTalk insteadof B;
                B::bigTalk as talk;
            }
        }
    ```

4. `as`操作符还可以用来调整方法的访问控制：
    
    ```php
    <?php
        trait HelloWorld {
            public function sayHello() {
                echo 'Hello World!';
            }
        }
        // 修改 sayHello 的访问控制
        class MyClass1 {
            use HelloWorld { sayHello as protected; }
        }
        // 给方法一个改变了访问控制的别名，原版 sayHello 的访问控制则没有发生变化
        class MyClass2 {
            use HelloWorld { sayHello as private myPrivateHello; }
        }
    ```

## 最后

PHP面向对象开发时，要灵活利用`abstract`、`interface`和`trait`，不要生搬硬套。

## 参考
-[类与对象-抽象类](http://php.net/manual/zh/language.oop5.abstract.php)
-[类与对象-对象接口](http://php.net/manual/zh/language.oop5.interfaces.php)
-[类与对象-Trait](http://php.net/manual/zh/language.oop5.traits.php)
