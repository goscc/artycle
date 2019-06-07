# Interface In Go -Part1
[原文地址（ https://medium.com/golangspec/interfaces-in-go-part-i-4ae53a97479c）](https://medium.com/golangspec/interfaces-in-go-part-i-4ae53a97479c)

使用一些接口，会让你的编码具有更强的灵活性、可扩展性，并且interface是golang用于实现多态（polymorphism）的一种手段。interfaces允许在一些行为被需要的时候才去指定，而不需要特定的类型。行为由一些方法的集合来定义：

```golang
type I interface {
    f1(name string)
    f2(name string) (error, float32)
    f3() int64
}
```

不要求特殊的关键字表明实现接口，只要输入所需方法名和签名（输入输出参数表）的方法定义，就足以说明它实现（满足）了一个接口：

```golang
type T int64

func (T) f1(name string) {
    fmt.Println(name)
}

func (T) f2(name string) (error, float32) {
    return nil, 10.2
}

func (T) f3() int64 {
    return 10
}
```

T类型满足了定义在第一块代码里面的的接口I，类型T可以传递给任何接受I作为参数的方法（代码如下）：

```golang
type I interface {
    M() string
}

type T struct {
    name string
}

func (t T) M() string {
    return t.name
}

func Hello(i I) {
    fmt.Printf("Hi, my name is %s\n", i.M())
}

func main() {
    Hello(T{name: "Michał"}) // "Hi, my name is Michał"
}
```
在Hello方法中，方法I.m()调用是抽象的，可以调用各种类型的方法，只要这些方法是有满足接口I的类型实现的。

Golang突出的特点是接口是被隐式实现的。程序不必特意声明`type T implements interfact I`。Go的编译器会来做这件事（永远不要派人去做机器的工作）。这个行为非常好的含义是定义接口的可能性，可以被已经写好的类自动实现（无需做任何改动）。

接口提供的灵活性来自于一个类型可以实现多个接口（代码如下）：

```golang
type I1 interface {
    M1()
}

type I2 interface {
    M2()
}

type T struct{}

func (T) M1() { fmt.Println("T.M1") }
func (T) M2() { fmt.Println("T.M2") }

func f1(i I1) { i.M1() }
func f2(i I2) { i.M2() }

func main() {
    t := T{}
    f1(t) // "T.M1"
    f2(t) // "T.M2"
}
```

或者是相同的接口可以被许多类实现（代码如下）：

```golang
type I interface {
    M()
}

type T1 struct{}

func (T1) M() { fmt.Println("T1.M") }

type T2 struct{}

func (T2) M() { fmt.Println("T2.M") }

func f(i I) { i.M() }

func main() {
    f(T1{}) // "T1.M"
    f(T2{}) // "T2.M"
}
```
**除了一个或多个接口需要实现的方法外，类型还可以自由的实现其他的方法**

---------

在Golang，有两个和interfaces相关的概念
1. 接口--实现该接口需要的一组方法集合。使用关键字：interface来定义
2. 接口类型--接口类型的变量可以保存任何实现了特定接口的值
下面两节我们来讨论一下这些主题

## 定义一个接口

接口类型的声明指定了属于它的方法，方法由其名称和签名(输入的参数和返回的结果)定义

```golang
type I interface {
    m1()
    m2(int)
    m3(int) int
    m4() int
}
```

除了方法外还可以使用限定名嵌入其他的接口（包括定义在相同的包内、和imported进来的）。它会加入所有来自被嵌入接口的方法。

```golang
import "fmt"

type I interface {
     m1()
}

type J interface {
    m2()
    I
    fmt.Stringer
}
```

接口J的方法结合包括：
- m1() (来自被嵌入的接口I)
- m2()
- String string (来源于被嵌入的接口Stringer)

顺序不要紧，所以可以方法规范和嵌入的接口类型可以交叉着使用。

从被嵌入的接口添加导出方法（大写字母开头）和非导出方法（小写字母开头）
如果我嵌入了一个接口J，这个接口J又嵌入了另一个接口K，那么K的所有方法也会被添加到I：

```golang
type I interface {
    J
    i()
}

type J interface {
    K
    j()
}

type K interface {
    k()
}
```

I的方法包括i(),j()和K()

环形嵌入接口是不被允许的并且会在编译过程中被检测到（代码如下）：

```golang
type I interface {
    J
    i()
}

type J interface {
    K
    j()
}

type K interface {
    k()
    I
}
```

编译器会抛出一个error`interface type loop involving I`.

接口方法必须有唯一的名字（代码如下）：

```golang
type I interface {
    J
    i()
}

type J interface {
    j()
    i(int)
}
```

否则将抛出编译时错误（compile-time error）:`duplicate method i`.

接口的组合可以在标准库的各个地方被发现。其中一个例子就是`io.ReadWriter`

```golang
type ReadWriter interface {
    Reader
    Writer
}
```

我们已经知道了如何创建新的接口，现在我们来研究一下接口类型的值

## 接口类型的值
接口类型的变量I可以保存任意实现了接口I的值（代码如下）：

```golang
type I interface {
    method1()
}

type T struct{}

func (T) method1() {}

func main() {
    var i I = T{}
    fmt.Println(i)
}
```
这里我们使用变量i，他是接口I类型的变量

## 静态类型vs动态类型
变量在编译阶段具有已知的类型，他是在生命时指定的，并且不会改变我们认为该变量是静态类型。接口类型本身也有静态类型，同时也具有动态类型，因此他们的赋值（代码如下）：

```golang
type I interface {
    M()
}

type T1 struct {}

func (T1) M() {}

type T2 struct {}

func (T2) M() {}

func main() {
    var i I = T1{}
    i = T2{}
    _ = i
}
```

静态类型变量i是I类型的，它不会改变。另一方面，动态类型就是动态的。第一次赋值后，i的动态类型是T1.但他不是固定不变的，因此第二次赋值后，i的类型变为了T2。当接口类型变量的值是nil（接口的空值），动态类型不会被设定

## 如何获取接口类型值的动态类型？
反射`(reflect)`包可以被用于实现这个（代码如下）：

```golang
fmt.Println(reflect.TypeOf(i).PkgPath(), reflect.TypeOf(i).Name())
fmt.Println(reflect.TypeOf(i).String())
```

除此之外，format包也可以通过格式化动词`%T`来做到这一点：

```golang
fmt.Printf("%T\n", i)
```
尽管他也基于反射包，但即使i是nil的时候，这个方法也可以正常工作。

## 空接口值(nil interface value)
这里我们使用一个例子来开头：

```golang
type I interface {
    M()
}

type T struct {}

func (T) M() {}

func main() {
    var t *T
    if t == nil {
        fmt.Println("t is nil")
    } else {
        fmt.Println("t is not nil")
    }
    var i I = t
    if i == nil {
        fmt.Println("i is nil")
    } else {
        fmt.Println("i is not nil")
    }
}
```

输出：
```golang
t is nil
i is not nil
```

一开始可能会感到一些惊讶，我们将一个nil值赋值给变量i，但是i却不等于nil。接口类型由下面两部分构成：
- 动态类型
- 动态值

动态类型我们已经在前面讨论过了（”静态类型和动态类型“）。动态值是实际分配的值，在赋值语句`var i I = t`的代码后，i的动态值是nil但是动态类型是`*T`。在这个赋值后调用方法`fmt.Printf("%T\n", i)`，会打印出`*main.T`。只有在动态值和动态类型同时为nil时，接口类型值才为nil。其结果是，即使接口类型的变量持有一个空指针，该接口的值也不为nil。已知错误是返回未初始化，非接口类型值从函数返回接口类型（代码如下）：

```golang
type I interface {}

type T struct {}

func F() I {
    var t *T
    if false { // 这段代码不可达，但是实际设置了值
        t = &T{}
    }
    return t
}

func main() {
    fmt.Printf("F() = %v\n", F())
    fmt.Printf("F() is nil: %v\n", F() == nil)
    fmt.Printf("type of F(): %T", F())
}
```

打印出如下结果

```golang
F() = <nil>
F() is nil: false
type of F(): *main.T
```
因为函数返回的接口类型值具有动态类型集`*main.T`，所以它不等于nil。

## 空接口（Empty interface）
接口的方法集合没有任何方法，他可以完全是空的（代码如下）：
```golang
type I interface {}

type T struct {}

func (T) M() {}

func main() {
    var i I = T{}
    _ = i
}
```
任何类型都会自动满足空接口--因此任何类型的值都可以付给空接口类型的变量。动态类型或静态类型的行为可以和非空接口一样的方式应用于空接口。在可变参数函数`fmt.Println`中使用了大量空接口。

## 实现一个接口

实现所有接口方法的每个类型都自动满足这个接口。我们不需要再这些类型中用任何像java中`implements`去说明这个类实现了一个接口。他是由Go编译器自动检测的，是Go语言的强大特性。

```golang
import (
    "fmt"
    "regexp"
)

type I interface {
    Find(b []byte) []byte
}

func f(i I) {
    fmt.Printf("%s\n", i.Find([]byte("abc")))
}

func main() {
    var re = regexp.MustCompile(`b`)
    f(re)
}
```
这里我们定义了一个接口，由`regexp.Regexp`类型实现。不用对内置regexp模块做任何更改。

## 行为的抽象

接口类型的值只允许访问接口类型提供的方法，隐藏了所有关于确切值的所有细节，想数据结构、数组、标量等等。（代码如下）：

```golang
type I interface {
    M1()
}

type T int64

func (T) M1() {}
func (T) M2() {}

func main() {
    var i I = T(10)
    i.M1()
    i.M2() // i.M2 未定义（类型I没有名字为M2的字段或方法）
}
```
