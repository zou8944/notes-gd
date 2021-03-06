# Scala学习笔记

## 热身

来源于Scala官网提供的[热身训练](https://docs.scala-lang.org/overviews/scala-book/prelude-taste-of-scala.html)，专供Java程序员使用。

### 变量

变量声明也分为val和var，和Kotlin一样，语法也一样。

### 控制语句

- if else，一样

- 匹配语句不一样，这里的匹配语句无比强大

  使用 match case => 这样的方式

  没有else，else就是 case _ => ...

  ```scala
  def getClassAsString(x: Any):String = x match {
      case s: String => s + " is a String"
      case i: Int => "Int"
      case f: Float => "Float"
      case l: List[_] => "List"
      case p: Person => "Person"
      case _ => "Unknown"
  }
  ```

- try-catch

  ```scala
  try {
      writeToFile(text)
  } catch {
      case fnfe: FileNotFoundException => println(fnfe)
      case ioe: IOException => println(ioe)
  }
  ```

# 《Scala编程》学习笔记

# 前言

这本书是2010年第一次出版，因此需要选择性参考，其余的需要参考官方手册。

# 基础介绍

- 面向对象编程 + 函数式编程 + 静态 + JVM = Scala

- 向Scala中加入自定义内容时，看起来就像是Scala内建的一样，体现了延展性

- Sacala能够如此具有伸缩性，是因为将面向对象和函数式结合了起来

  - 面向对象

    很多语言的面向对象其实不仅仅包含了对象，还包含了注入原生类型、运算符之类的不属于任何对象的内容，这会使编程变得很复杂，限制了伸缩性。

    而Scala的面向对象很纯粹，每个值都是对象，每个操作都是方法调用，没有多余的东西。

  - 函数式

    函数式两个理念：函数是一等公民；函数不应产生副作用。

- Scala也能够与Java完全互操作。几乎所有Scala库都极度依赖Java库。比如类型Int对应的是Java的int类型，异常是java.lang.Throwable的实现类

- Scala优点总结

  - 兼容Java
  - 语法简洁
  - 抽象层级高
  - 静态类型，且类型推断做得很好

- Sacala怎么提现伸缩性

  小到可以写简短的脚本代码；达到可以写庞大的系统

  - 脚本：不经过编译，直接使用scala解释器执行，因此甚至一句代码都可以直接执行
  - 系统：需经过编译，必须遵从面向对象，不能直接执一行裸代码

# 基础的点

- Scala将索引放在括号中，而不是其它语言的方括号中
- scala将集合区分为可变和不可变两种类型，目的是让不可变应用于函数式、可变应用于一般的命令式。因此我们使用时应该注意

## List

List本身及其元素都是不可变的

一些常用的方法

- List() 或Nil，都代表空List

- ::: 叠加两个List，取并集

- :: 在列表前新增一个元素

  ```scala
  // 在空List的基础上，增加3 2 1。
  1 :: 2 :: 3 :: Nil
  ```

  :: 也是方法，不过它的主语在右边，比如上面等价于 `Nil.::(3).::(2).::(1)`

## Tuple

Tuple本身及其元素都是不可变的，但可同时存储多种类型数据。

且它们可在编译期判定元素类型，访问方法从_1, _2这样开始，因为对于拥有静态元组的其他语言如Haskell，都是从1开始的

scala目前最长支持到Tuple22

## Set和Map

Set和Map都只是trait，其实现类有可变和不可变两个分支。

![image-20200308153541428](Scala学习笔记/image-20200308153541428.png)

![image-20200308153559204](Scala学习笔记/image-20200308153559204.png)

- Map初始化时，使用->方法，该方法对所有对象都有。

- 允许我们对任何对象调用-> 的机制称为隐式转换

  ```scala
  val romanNumeral = Map(1->2, 2->3)
  ```

## 向函数式风格推进

- 最关键的点就是不要使用任何var编程
- scala鼓励函数式编程的一个原因，是可以写出更为简洁的代码，方便维护，方便测试，且不容易犯错
- 另一个关键点就是尽量去除副作用，或是将副作用限制在很小的范围内

# 类和对象

- public是scala的缺省访问级别

## 方法

- 与kotlin不同的是，scala方法的不必加return，直接将最后一行的结果作为结果

- scala推荐的定义方法的风格

  - 避免显式地使用return返回的方法定义
  - 避免有多个返回语句的方法定义
  - 鼓励将方法分解为多个小的方法，将方法当做创建返回值的表达式

- 如果方法不声明返回值，那么即使方法体有返回值，那它的返回值也是Unit

  ```scala
  def g() {"Hello World!!"}
  ```

## 分号

- 句末不用加分号

- 多行语句合并在一行时，之间要加分号

分号推断规则：除非以下的一种情况成立，否则行尾被认为是一个分号

- 行由一个非合法结尾字结尾，如中缀操作符
- 下一行的开始不能作为语句的开始
- 行结束语括号或方括号内部

## 单例对象

- Java不够面向对象，因为它有静态成员

- scala没有静态成员，类似功能由单例对象完成

  ```scala
  object Hello {
      . . . . . .
  }
  ```

- 单例对象可以单独定义，也可以定义成某个类统一名称，此时该单利对象叫做伴生对象；相应地，同名类称为单例对象的伴生类

- 单例对象会在第一次访问的时候初始化

- 没有伴生类的单例对象称为孤立对象，孤立对象常用的使用场景

  - 把相关的功能方法搜集在一起
  - 定义一个Scala应用的入口点

## Scala程序

- 要执行scala程序，必须提供一个有main方法的孤立单例对象
- scala隐式地引用了如下包，如println方法就是来自Predef包下的
  - java.lang
  - scala
  - Predef

- Scala的文件名和类名没有关系，但为了方便，还是建议和Java保持一样的风格
- scala文件可以被当做脚本执行，或者当做源码编译。
  - 文件以表达式结束的会被当做脚本执行
  - 文件以定义结束的当做源码编译
- scalac编译很慢是由于每次都重新扫描包。可以使用fsc命令，它启动一个后台服务，保存上次的编译结果，修改代码再编译时可以省略一些步骤，达到快速编译的效果

## Application特质

- 执行scala程序的另一个方法是实现Application特质
- 实现后，在鼓励对象中直接写想要执行的代码即可
- 这种方式有局限性，因此除非场景很简单，不推荐使用

# 基本类型和操作

- 基本类型和Java完全一样，只是开头大写，并且是来自scala包
- String类型直接使用了java.lang.String

## 文本

- 字符串文本支持原始字串，即`""" lalala """`

- 符号文本

  写成`'<标识符>`的形式，会被映射成scala.Symbol的实例

  ```scala
  # 如下两个等效
  'hello
  Symbol("hello")
  ```

  不过，前面那种方式已废弃了，推荐使用后者

## 操作符和方法

- 所有的操作符都是作用在普通方法调用上的华丽语法

- 不像kotlin必须使用operator、infix之类的声明关键字才能将一个方法声明为操作符，scala将所有方法都可以当做操作符

- 下面展示几种调用方式和等效方法调用

  ```scala
  # 中缀等效
  1 + 1
  1.+(1)
  # 前缀等效
  -1
  1.unary_-()
  # 后缀等效
  "" toLowerCase
  "".toLowerCase()
  ```

  前缀需要说一下，并不是所有方法都可以被当做前缀的，只有+ - ! ~四个才行

## 对象相等性

scala不像java，==比较的是值的相等性，无论是基本类型还是引用类型

## 富包装器

对于每个基本类型，scala都提供一个富包装类型，以提供功能更多的很多隐式方法。

Int对应的富包装器位置为scala.runtime.RichInt

# 函数式对象

- 类参数

  如下a和b被称为类参数，Scala编译器会收集这两个类参数并创建带有这两个参数的主构造器

  ```scala
  class Rational(n: Int, d: Int)
  ```

- Scala编译器将把类内部的非字段、非方法的代码段编译进主构造器，因此这些代码段会在类创建时执行

## 标识符约定

- 字母数字标识符

  - scala和java一样，对于一般变量遵循驼峰命名法。
  - 变量名最好不要包含$和__，因为可能和系统变量名冲突
  - magic number最好不要用多了
  - scala常量用const修饰，命名规则为首字母大写的驼峰命名法。如XOffset。Java的全大写也可以使用

- 操作符标识符

  - 由一个或多个操作符字符组成。即 + - * ?之类的

  - Scala编译后，会将操作符标示转换为合法的带有$的Java标识符

    举例。 :-> 将被转换为 \$colon\$minus\$greater

- 混合标识符

  - 由字母数字加上一个下划线加一个操作符标识符

    如 unary_+，被定义为一元加操作符的方法名

- 文本标识符

  - 用反引号包起来的任意字串

## 隐式转换

可以通过定义如下方法完成Int在合适的时候自动转换为Rational操作

```scala
implicit def intToRational(x: Int) = new Rational(x)
```

这个方法很强大，不过不要乱用。且Int能不能被隐式转换，还要看是否符合方法定义的作用域。

隐式转换代表了不容易被看出，因此除非很显而易见，不然别用它。

# 控制结构

scala仅存在一下几个内建的控制结构，并且所有控制结构都会产生一个值。

- if else

- while 返回Unit，这个返回值并没有什么意义。

- for

  for的下面这种语法被称为发生器语法

  for能够解析任何种类的集合类，包括集合，range等都有效

  ```scala
  for (item <- items)
    print(item)
  ```

  for还可以加if，变成具有过滤器功能的结构

  ```scala
  for (item ，<- items if item.size > 0)
    print(item)
  ```

  过滤器可以包含很多个，而不只是一个

  ```scala
  for (
  	item <- items
      if item.size > 0;
      if item.name == ''
  )
    print(item)
  ```

  一个for中还可以加入多个<-子句，从而得到嵌套循环。

  ```scala
  // items是外层循环，names是内层循环
  for {
  	item <- items
      if item.size > 0
      name <- names
  }
    print(item + " " + name)
  ```

  还可以同时进行变量绑定

  ```scala
  for {
    item <- items
    name = item.name
  }
    print(name)
  ```

  还可以将循环之后的结果创建一个新的集合 即 for yield

  ```scala
  // 将items中元素的名字拿出来，加上hello，创建一个新的数组
  def names = 
  for {
      item <- items
      name = item.name
  } yield {
      name + "hello"
  }
  ```

- try catch

  用法和java完全一样，不同的是

  - catch块支持模式匹配

- match

  模式匹配

  可以匹配的类型是任意的，非常强大。

  每个case语句默认包含了break的功能。因此不想java那样需要break。

- 函数调用

  当一个函数是尾递归时，scala编译器会自动将其编译成循环的形式

- 变量范围

  和Java一样，除了一点

  - scala可以在嵌套块中定义和外层名称重复的变量。

# 函数和闭包

- 作为一个对象的成员的函数，叫做方法
- 本地函数：定义在函数中的函数，很有用

## 偏应用函数

英文名为 partially applied function，这里的偏，是部分的意思。即被部分应用的函数。说得更明白一点，就是部分参数被固定住，当然，这个“部分”可以是0， 于是有了下面第一二个例子。第三个例子能够更形象地展示。

```scala
def sum(a: Int, b: Int, c: Int): Int = a + b + c

// 下面三种都属于偏应用函数
// a为有三个Int参数并返回Int的函数
val a = sum _
// 同上
val a = sum(_)
// a为有一个Int参数并返回Int的函数
val a = sum(1, _, 2)
```

偏函数的原理

​	执行`a=sum _`时，scala编译器会创建一个类，该类存在一个apply方法，该方法的参数为sum同样的参数类型和数量，apply方法内部调用了sum方法。因此，执行'`a(1,2,3)`时其实相当于调用了创建类的`apply(1,2,3)`方法，而该方法调用了`sum(1,2,3)`

## 闭包

闭包名称的来源

- 不带外部变量的函数，叫做closed term

  ```scala
  val addMore = (x: Int) => x + 1
  ```

- 带外部变量的函数，即会使用到外部变量的函数，叫做open term

  ```scala
  val more = 1
  val addMore = (x: Int) => x + more
  ```

- 一个open term在定义时是开放的，但在执行到这个函数时，more将被捕获，函数封闭，得到运算值。这个过程被称为闭包，我们也把这个open term叫做一个闭包，即closure。

- 闭包只有在运行时会捕获外部变量，并且对外部变量的修改在闭包外部也是可见的。因为scala操作的是该自由变量的引用，该自由变量是存在于堆中，而不是栈中（如果是栈中，闭包是一个方法调用，调用完成后对自由变量的修改是不会保存的）

## 变长参数

在参数末尾加上*表示可变参数，它实际上是Array类型。

在Array类型末尾加上*可以让编译器把数组中的每个元素当做参数

```scala
val a = (args: String*) => args.foreach(println)
a(Array("1", "2")*)
```

## 尾递归

- 尾递归就是一个动作调用自己，就称之为尾递归

- scala尾递归优化是自动打开的。

- 可以手动关闭：加参数 `-g: notailcalls`

- scala尾递归有局限性，只能优化直接形式的尾递归。不能优化间接形式，甚至在最后一步调用自身函数的别名都不行。

# 抽象控制

## curry化

是一种函数式编程技巧，将多个参数的函数编程只含有一个参数的多级函数（高阶函数）

```scala
def sum(x: Int, y: Int): Int = x + y
// 柯理化结果如下
def sum(x:Int)(y:Int) = x + y
```

解释：传入第一个参数x会产生一个(Int) => Int函数，再传入第二个参数，能够得到最终结果。

- 当一个函数只有一个参数时，对函数的调用可以用{}替换()。这样的好处是能够自定义结构，结合柯理化，更加能够发挥它的威力

## by-name参数

与by-value（传值参数）参数对应，传值参数会在传入前计算值，而by-name（传名参数）参数是在方法调用后再计算的。其原理是将传入的内容包装成不带参数匿名函数。

传值参数、传名参数、传函数的参数，写法和使用上有如下区别

```scala
// by value参数
scala> def test(param: Int) = println(param)
test: (param: Int)Unit

scala> test(1 + 1)
2
// by-name参数
scala> def test(param: => Int) = println(param)
test: (param: => Int)Unit

scala> test(1 + 1)
2

// 传入函数
scala> def test(param: () => Int) = println(param())
test: (param: () => Int)Unit

scala> test(() => 1+1)
2
```

可以看到， by-name 参数在函数体的调用上，也没有像函数那样调用，而是直接当做值一样使用，但在真是执行效果上，却是像传入函数一样的执行。这一点要注意。

可以看做使用高阶函数做的一个福利。

# 组合与继承

## 无参函数

无参函数可以将参数列表的括号去除

此时，定义一个午无参函数和一个变量的唯一区别仅在于val换成了def

```scala
scala> def test :Int = 1
test: Int
```

要不要用无参函数，有以下几个原则

- 当函数无参，且函数不产生任何副作用，即只读取值时，可以省略括号。并鼓励省略括号
- 如果函数会产生副作用，必须加上()

即遵循统一访问原则

- 用户调用一个看起来不可变的变量、属性或方法时，都可以以不带括号的形式调用，而不用管它到底是属性还是方法。
- 作为反例，Java的数组的length属性、字符串的length()方法，如果遵循统一访问原则，都能以length的方式访问，但实际不行。

## 继承

- AnyRef是所有类的超类

- 方法和字段是可以重载的

  scala中方法和字段处于一个命名空间，即他们的名称必须唯一，这带来了有意思的东西

  - 可以在子类中用字段覆盖父类的无参方法

    ```scala
    abstract class dadClass {
      def property: Int = 1 + 1
    }
    
    class sonClass extends dadClass {
      override val property: Int = 1
    }
    ```

  - 字段名和方法名不能相同

- 方法不允许子类重写，也是使用final修饰符

# Scala的层级

![image-20200314151956124](Scala学习笔记/image-20200314151956124.png)

上面是scala的层级，可以看到最顶级的是Any，与java.lang.Object相对应的是Any的子类AnyRef

## 值类

AnyVal称为值类，下属八种值类型和一个Unit类型，虚线说明了他们能够在计算时进行隐式转换。

隐式转换的原理：每个类都对应了scala.runtime.Richxxx的类，当需要和更高级别的值进行计算时，实际上应用的是RichInt中事先设置好的转换。

## 引用类

AnyRef称作引用类，在Java平台上，AnyRef实际上就是Object的别名。String就直接用了java.lang.String。对于scala特有的类，还继承了scala.ScalaObject特质，这是用来让scala的执行更有效。

## 底层类型

- Null

  Null是null的引用类型，是除了所有值类型外的子类型

- Nothing

  Nothing是所有类型的子类型

# 特质

特质，trait，与接口仅仅是类似，实则有很多不同

- 它可以定义已实现的方法、可以声明字段和位置状态值
- 它可以用extends字段混入类中，而不是被实现
- 在特质的方法中进行super调用，这个super的具体指向是动态绑定的。

## 用于堆叠改变

这是特质看起来最屌的一个特性，用例子说明

```scala
abstract class IntQueue {
    def get(): Int
    def put(x: Int)
}

class BasicIntQueue extends IntQueue {
    private val buf = new ArrayBuffer[Int]
    def get() = buf.remove(0)
    def put(x: Int) { buf += x }
}

trait Doubling extends IntQueue {
	abstract override def put(x: Int) { super.put(2 * x) }
}

trait Incrementing extends IntQueue {
    abstract override def put(x: Int) { super.put(x + 1) }
}

val queue = (new BasicIntQueue with Doubling with Incrementing)

queue.put(1)
queue.get() // 得到的结果是40
```

上面定义了抽象类IntQueue，定义实现类BasicIntQueue，同时定义两个特质，分别用于翻倍和加一，这里的关键在于特之中使用super调用父类的put方法，而父类是谁，只有执行时才知道。从而带来了非常大的灵活性。

当混入多个特质时，最右边那个最先生效。

## 堆叠的原理

堆叠的原理就是线性化组织继承关系。

Java中，如果一个类继承多个类或接口，则它和父类的关系是分叉结构，可能两个父类是完全平行且无交叉的，这样虽然好理解，但不够灵活。

Scala中，是将该类和所有父类以线性的结构连接在一起的。

举例如下

```scala
class Animal
trait Furry extends Animal
trait HasLegs extends Animal
trait FourLegged extends HasLegs
class Cat extends Animal with Furry with FourLegged
```

总体来看，它们的继承关系如下

![image-20200314164210324](Scala学习笔记/image-20200314164210324.png)

但是，对于类Cat，它的继承结构被scala整理为线性化之后如下

```scala
Cat -> FourLegged -> Furry -> Animal -> AnyRef -> Any
```

所以，如果我在特质`FourLegged`中调用super，调用到的肯定是第一个实现类`Animal`，`Furry`也一样。如果我把`Animal`换成别的实现类，这时的super就指的是其它类啦，这就构成了动态堆叠。

# 包和引用

## 包

- Scala的包是嵌套的，即允许以如下的方式命名。虽然可以写成Java的方式，但是注意，Java的包却并不是嵌套的。

  ```scala
  package bobsrockets.navigation {
      // 在bobsrockets.navigation包里
      class Navigator
      package tests {
          // 在bobsrockets.navigation.tests包里
          class NavigatorSuite
      }
  }
  ```

- scala中定义的所有顶层包都属于名为\_root\_的根包。即上面定义的bobsrockets其实是\_root\_.bobsrockets

## 引用

scala的引用有如下几种形式

- 常规的简单名x，即单独引用这个类
- x => y，引入x，并起别名为y
- x => _ 引入x外的所有
- _ 引入所有

scala隐式地包含了如下三个引用

- import java.lang._
- import scala._
- import Prefdef._

## 访问控制符

scala和Java的访问控制符基本上是一样的，主要体现在如下几个方面

- 默认公开，且公开成员没有修饰符

- 可以有范围地保护

  可以通过  `修饰符[包名/类名/对象]`的方式限制控制符的范文

  ![image-20200315174755049](Scala学习笔记/image-20200315174755049.png)

- 类和它的伴生对象共享访问权限，即定义在对方的私有成员都能够互相地访问到。

# 断言和测试

两个断言函数

- assert()
- ensuring()

几个单元测试框架

- JUnit
- TestNG
- ScalaTest
- specs
- ScalaCheck

# case class和模式匹配

## 样本类

case class的case修饰符是编译器自动为这样的类增加一些语法所做的便捷设定

编译器为我们做了如下事项

- 添加一个与类名一致的工厂方法
- 所有参数列表中的参数隐式地获得了val前缀
- 添加toString、hashCode、equals的自然实现

使用case class的最大优点，还是它能够在模式匹配中使用

## 模式匹配

首先解释模式，其是英文pattern的直译，如果觉得模式不好理解，那就记pattern就好了。因为我认为模式不方便理解。

模式匹配非常强大，看下面的例子

```scala
expr match {
    case 1 => "匹配常量"
    case hello => "匹配变量"
    case List(0, _, _) => "以0开头的含有三个元素的序列"
    case s: String => "匹配类型"
    . . . . . .
}
```

scala中手动判断类型是非常麻烦的。这是故意的，为了让我们尽量使用模式匹配实现。

```scala
// 类型判断
x.isInstanceOf(String)
// 类型转换
val str = x.asInstanceOf(String)
```

## 模式守卫

守卫是在一个模式匹配分支中加上bool表达式，仅在表达式返回true时，才会匹配成功

```scala
expr match {
    case n: Int if n > 0 => "仅匹配正数"
    case _ => "其它情况"
}
```

## 类型擦除

Scala和Java一样，除了数组，其它的泛型在运行时都会被擦除。

## 密封类

和Kotlin的定义一样，密封类是仅允许在该类定义的文件中定义其子类，文件外就不行。

密封类很适合用来做模式匹配，将所有情况都限制在一个文件定义中，就不会出现漏掉的情况了。

有点类似枚举

## Option

Option类也适合用模式匹配来解析

像这种存在有限情况的类，都很适合用模式匹配来解决

```scala
x match {
    case Some(s) => s
    case None => "?"
}
```

# 列表

- 列表相比数组来说，有以下几点不同
  - 列表是不可变的，即其元素不能变化
  - 列表具有递归结构，即head+tail的递归结构，每一个tail又是head+tail的结构

- 列表是协变的，即`List[String]`是`List[Object]`的子类型

# 集合类型

- Scala的集合类型的主要特质继承如下。所有集合类型都集成了Iterable特质。

  ![image-20200321105910772](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200321105910772.png)

- 集合对象可以通过elemetns方法产生Iterator
- Iterator和Iterable看起来很像，但他们不是同一个层级的东西，Iterator扩展了AnyRef，它是用来执行枚举操作的特质，一个Iterator只能被使用一次。

## Seq

### List

- 列表是一个递归定义的形式，即head+tail的递归结构，每一个tail又是head+tail的结构
- 因此列表只能做取head和tail的操作，不能直接获取某个元素

### ListBuffer

- 是可变对象，可以高效地通过添加元素的方式构建列表
- 使用toList可以转换成List对象，因此可以使用ListBuffer来动态构建List，再用toList()得到一个最终的LIst

### Array

- 数组是一组元素序列，可以使用index直接访问到元素

### ArrayBuffer

- 与数组类似，不过还允许我们在序列开始和结束的地方添加和删除元素

### Queue

- 即队列，分为可变和不可变的队列

### Stack

- 即栈，也分为可变和不可变两部分

### RichString

- 它是Seq[Char]
- 由于Predef包含了从String到RichString的隐式转换，因此可以把任何字符串当做Seq[Char]

## Set && Map

在Predef中有如下定义，即默认导入的Set和Map等都是不可变的，要使用可变的，需要我们手动指明包

```scala
  /**  @group aliases */
  type Map[K, +V] = immutable.Map[K, V]
  /**  @group aliases */
  type Set[A]     = immutable.Set[A]
  /**  @group aliases */
  val Map         = immutable.Map
  /**  @group aliases */
  val Set         = immutable.Set
```

- Set的常用操作如下

  ![image-20200321113335658](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200321113335658.png)

- Map的常用操作如下

  ![image-20200321113707392](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200321113707392.png)

  ![image-20200321113726197](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200321113726197.png)

### SortedSet和SortedMap

- 即有序集和有序Map
- 他们的实现是TreeSet和TreeMap

## SynchronizedSet和SynchronizedMap

- 即线程安全的Set和Map
- 它是一个特质，使用时需要通过混入它来实现自己的线程安全的Set或Map

## 元组

- 元组可以组合不同类型的对象，因此它不是Iterable的子类
- 元组如果易于使用，但会在语义上给人冲突，直接将两个不同类型对象组合起来在语义上多数时候是说不通的，因此还是要慎重使用。

# 有状态的对象

- 一个对象有无状态，和它是否持有var类型的属性没有必然联系

- 如果一个对象属性hour被定义为var，则其会产生名为hour的getter方法和名为hour_的setter方法

  因此可以显式声明getter和setter，覆盖原本就有的内容

# 参数化类型

以如果A是B的子类型为前提，说明参数化类型之间的关系

- 不变，Scala中默认的参数类型之间是无关的。

  即：如果定义类型Clazz[A]，则Clazz[A]和Clazz[B]互不为父子类型

- 协变，Clazz[+A]

  即：Clazz[B]是Clazz[A]的子类型

- 逆变，Clazz[-A]

  即：Clazz[B]是Clazz[A]的符类型

- 类型上界，Clazz[T <: A]

  即：能够应用于Clazz的参数类型必须是A的子类

- 类型下界，Clazz[T >: A]

  即：能够应用于Clazz的参数类型必须是A的父类

- 上下界和协变逆变可以组合起来

# 抽象类型

特质和抽象类可以包含一个抽象类型成员，实际类型可以由实现来确定，比如

```scala
trait Buffer {
    type T
    val element: T
}

class ConcreteBuffer extends Buffer {
    type T = List[String]
    val element = List("1", "2")
}
```

## val懒加载

在val前加上lazy修饰符，会在使用时才会去计算val的值

```scala
lazy val s = System.currentMillis()
```

## 枚举

Scala的枚举并没有特殊的语法，而是继承scala.Enumeration类型即可。

# 隐式转换和参数

隐式转换，指的是，当一个方法声明为implicit时，在其对应参数类型运算出现编译错误时，编译器将自动应用该implicit方法，使得程序能够正常执行

```scala
class Test {

  implicit def int2String(int: Int): String = int.toString

  def test: Int = 12.length
}
```

关于隐式转换，有如下几个点需要注意

- 只有标记为implicit的定义才会被隐式转换
- implicit定义只有在合法作用于内才会被调用
- 如果作用域内有多个合法匹配的隐式方法，将由于歧义直接报错
- 隐式转换只会调用一次，不会在隐式转换的基础上再次调用隐式转换，即不会嵌套调用
- 隐式方法命名随意

隐式转换将在如下三种情况下尝试

- 转换为目标类型

  当需要String时，但传入了一个int，此时会应用int转换String的隐式转换

- 调用者的转换

  当对int调用String的方法时，会应用int转换String的隐式转换

- 参数列表转换

  将implicit修饰的val隐式应用到声明为implicit的参数中

  需要注意的是，参数列表要么不用隐式转换，要么全用隐式转换，不存在部分应用的情况。

  隐式参数的类型必须被命名，不能是(Int) => String这样的过于通用的函数签名

```scala
class Test {

  class Inner[String]
    
  implicit def int2String(int: Int): String = int.toString

  // 调用者的转换
  def test: Int = 12.length

  def testTest(str: String): Unit = print(str)
    
  def testTest2(implicit str: String, int: Int): Unit = print(str)

  def main(args: Array[String]): Unit = {
    // 转换为目标类型
    testTest(12)
    // 转换为目标类型
    new Inner[12]()
    // 参数列表转换，s被隐式应用到testTest2中。
    implicit val s: String = "helo"
    implicit val i: Int = 2
    testTest2
  }

}
```

调试隐式转换代码的方法

- 不多的话，可以先显式调用，没有问题再写成隐式转换
- 可以scalac编译成隐式调用之后的代码，再查看是否符合预期

# 列表深入探索

## List实现原理

前面说过，scala中的List是递归的数据结构。

在类型构成上，它只由List抽象类和::、Nil两个子类构成，所有的列表都表示成::+Nil的形式

![image-20200322114519856](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200322114519856.png)

```scala
case object Nil extends List[Nothing] {
    override def head: Nothing = throw new NoSuchElementException("")
    override def tail: List[Nothing] = throw new NoSuchElementException("")
    override def isEmpty = true
}

final case class ::[+A] (override val head: A, private[scala] var next: List[A]) extends List[A] {
    override def headOption: Som[A] = head
    override def tail: List[A] = next
    override def isEmpty = false
}
```

平时使用::和:::操作符创建列表，他的签名如下

```scala
def ::[B >: A](elem : B): List[B] = ::(elem, this)
def :::[B >: A](prefix : List[B]): List[B] = prefix.head::prefix.tail:::this 
// 由于是右关联的操作符，因此相当于this.:::(prefix.tail).::prefix.head
```

一个构建好的List应该如下

![image-20200322115558255](/home/floyd/PersonalCode/notes-gd/notes/Scala学习笔记/image-20200322115558255.png)

其余的所有平常使用的函数式方法，如map，都可以使用这个递归定义实现

```scala
def map[B](f : A=>B) : List[B] = f(this.head)::this.tail.map(f)
```

然而，上面这些方式都不是尾递归，意味着效率低下且存在栈溢出的风险。为了解决这个问题，在实现这些方式时，scala采用了效率优先的ListBuffer

## ListBuffer

待添加



# for表达式详解

for表达式的yield会被解析成map、flatmap等高阶函数

详细来说，只要被for迭代的对象包含如下四个高阶函数，for都会被转义，便可以完美支持for表达式

- map
- flatMap
- filter
- forEach

实现上述四个高阶函数的部分，则可以部分支持for表达式

举两个例子

```scala
for (x <- books ) yield x + 1
转义为
books.map(x => x + 1)

for (book <- books if book.id > 20) book.name
转义为
books.filter(book=>book.id>20).map(book=>book.name)
```

# 抽取器

抽取器就是具有名为unapply成员方法的对象，unapply方法的目的是为了匹配并分解值。用在模式匹配中

```scala
object Email {
    unapply(str: String): Option[(String, String)] = {
        val parts = str split "@"
        if (parts.length == 2) Some(parts(0), parts(1)) else None
    }
}

// 有了上面的定义，就可以将一个邮件字符串在匹配时结构成变量
emailString match {
    case Email(name, domain) => print(name, domain)
    case None => print("Nothing")
}
```

case匹配时，将会引发unapply调用。

- apply和unapply成对出现，且逻辑对偶，但并非强制，推荐这样做

- unapply返回值一般为类型为元组的Option，但也可以是另外的类型

  - 单个元素时，为该元素类型的Opton
  - 为Boolean时，不再需要Option，直接是Option

- unapply的返回值可以是Option[Seq[T]]类型，此时结构的参数是变化的。

  集合就是实现了变参类型的unapply才可以结构随意多个参数

## 抽取器 VS case class

case class也可以用在case匹配中，但有一个缺点是结构出来的参数就是构造函数的参数，相当于暴露了类的结构，这是它不如抽取器的一点。

# 注解

## 元编程

就是处理程序的程序。

## 注解

注解方式也是@注解类名

- 应用位置

  可应用于任何类的声明和定义上，包括val、var、def、class、object、trait、type

  还可以应用于表达式

- 常用注解

  - @deprecated	废弃

  - @volatile

    告知编译器该变量将被多线程使用，相当于Java中的volatile修饰符

  - @serializable

    默认情况下，类是不可序列化的，增加该注解可使其可序列化

  - @SerialVersionUID(1234)

    为序列化增加版本号

  - @transient

    序列化时忽略该注解注解的字段

  - @scala.reflect.BeanProperty

    放在字段上，编译器自动生成get和set方法

  - @unchecked

    忽略检查

# Actor和并发

