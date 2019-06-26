# Interface In Go Part2
[原文地址（https://medium.com/golangspec/interfaces-in-go-part-ii-d5057ffdb0a6）](https://medium.com/golangspec/interfaces-in-go-part-ii-d5057ffdb0a6)

有一些时候，一些值需要被转换为另一个不同的类型。转换操作是一个编译时受检的操作，并且整个机制在更早的[另一篇文章](https://medium.com/golangspec/conversions-in-go-4301e8d84067)里面有详细说明。简单来说，看起来就好像是这个样子的：

```golang
type T1 struct {
    name string
}

type T2 struct {
    name string
}

func main() {
    vs := []interface{}{T2(T1{"foo"}), string(322), []byte("abł")}
    for _, v := range vs {
        fmt.Printf("%v %T\n", v, v)
    }
}
```

输出

```golang
{foo} main.T2
ł string
[97 98 197 130] []uint8
```

Golang的可赋值性规则，在一些情况下允许分配不同类型的值给变量（代码如下）：

```golang
type T struct {
    name string
}

func main() {
    v1 := struct{ name string }{"foo"}
    fmt.Printf("%T\n", v1) // struct { name string }
    var v2 T
    v2 = v1
    fmt.Printf("%T\n", v2) // main.T
}
```

这篇文章主要的精力会放在当接口类型加入进来时golang的转换策略。另外，会介绍几个新结构--类型断言和类型切换。

---

首先假设我们有两个接口类型的变量，并且我们想将一个赋值给另一个（代码如下）：

```golang
type I1 interface {
    M1()
}

type I2 interface {
    M1()
}

type T struct{}func (T) M1() {}func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```

这种方式是简单的，程序也可以正常的工作。第三种赋值情形适用于这里：

**T is an interface type and x implements T.**

这是因为变量*v1*的类型实现了*I2*的接口。不管这些类型的结构如何。（源码如下）

```golang
type I1 interface {
    M1()
    M2()
}

type I2 interface {
    M1()
    I3
}

type I3 interface {
    M2()
}

type T struct{}

func (T) M1() {}
func (T) M2() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```

即使`I2`已经包含的其他接口，但是`I1`没有，这些接口仍然互相实现（即只要包含的方法集相同，则可以互相转换）。方法的顺序不重要，应该记住方法集不一定要相等（源码如下）：

```golang
type I1 interface {
    M1()
    M2()
}

type I2 interface {
    M1()
}

type T struct{}

func (T) M1() {}
func (T) M2() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```

这段代码可以正常运行仅仅是因为第三种赋值情形。类型`I2`的值实现了`I1`，因为`I2`的方法集是`I1`的子集。如果不满足这种情况，编译器将实时做出反应（源码如下）：

```golang
type I1 interface {
    M1()
}

type I2 interface {
    M1()
    M2()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```

上面这段代码无法正确编译，因为会抛出下面这个错误：

```golang
main.go:18: cannot use v1 (type I1) as type I2 in assignment:
	I1 does not implement I2 (missing M2 method)
```

我们已经看到了涉及到两种接口的情形。前面列出的第三种可赋值情形同样适用于右侧为具体类型值（non-interface type）实现了一个接口（源码如下）：

```golang
type I1 interface {
    M1()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    _ = v1
}
```

当需要将一个接口类型的变量赋值给一个具体类型的变量，到底是怎么工作的呢？（源码如下）：

```golang
type I1 interface {
    M1()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    var v2 T = v1
    _ = v2
}
```

这段代码不会工作，并且会抛出下面的异常`cannot use v1 (type I1) as type T in assignment: need type assertion`。这里就是类型断言介入的地方。

只有当go编译器可以检查其正确性的时候，转换才可以被完成。无法在编译时验证的场景如下：
1. 接口类型 → 具体类型 (源码如下)：

```golang
type I interface {
    M()
}

type T struct {}
func (T) M() {}

func main() {
    var v I = T{}
    fmt.Println(T(v))
}
```

这段代码会给出一个编译错误，`cannot convert v(type I) to type T: need type assertion`。因为编译器不知道这种隐式转换是否有效，因为任何值实现了接口`I`都可以赋值给变量`v`。

2. 接口类型 → 接口类型，其中右边的方法集不是左边类型的方法集的子集（源码如下）：
```golang
type I1 interface {
    M()
}

type I2 interface {
    M()
    N()
}

func main() {
    var v I1
    fmt.Println(I2(v))
}
```

编译的输出：

```golang
main.go:16: cannot convert v (type I1) to type I2:
	I1 does not implement I2 (missing N method)
```

原因和之前一样，如果`I2`的方法集是`I1`方法集的子集，编译器将会在编译阶段知道这个。但是不同的是，这种转换只有在运行时才可以进行。

> 这不是严格意义上的类型转换，而是类型断言和类型切换，允许检查/检索接口类型值的动态值或者将接口类型的值转换为不同接口类型的值。

## 类型断言

类型断言的语法格式如下：

```golang
v.(T)
```

其中v为接口类型，T为抽象类型或具体值类型

### 具体类型
首先来让我们看一下，非接口类型（non-interface）是如何工作的（代码如下）：

```golang
type I interface {
    M()
}
type T struct{}

func (T) M() {}

func main() {
    var v1 I = T{}
    v2 := v1.(T)
    fmt.Printf("%T\n", v2) // main.T
}
```

类型断言指定的类型必须实现变量`v1`的接口类型`I`，这将在编译阶段被验证（代码如下）：

```golang
type I interface {
    M()
}

type T1 struct{}

func (T1) M() {}

type T2 struct{}

func main() {
    var v1 I = T1{}
    v2 := v1.(T2)
    fmt.Printf("%T\n", v2)
}
```

成功编译这样的代码是不可能的，因为会抛出这个错误`impossible type assertion`，变量`v1`不能持有任何类型`T2`的值，因为`T2`不满足接口`I`同时变量`v1`只能存储实现了接口`I`的类型的值。

在程序运行的过程中，编译器不知道变量`v1`存储了什么类型的值。类型断言是一种可以检索接口类型值的动态值的方法。但是如果`v1`的动态类型和`T`不匹配的时候会发生什么呢？（代码如下）：

```golang
type I interface {
    M()
}
type T1 struct{}

func (T1) M() {}

type T2 struct{}

func (T2) M() {}

func main() {
    var v1 I = T1{}
    v2 := v1.(T2)
    fmt.Printf("%T\n", v2)
}
```

程序将会不知所措（panic）：

```golang
panic: interface conversion: main.I is main.T1, not main.T2
```

### 多值转化（请不要惊慌）

类型断言可以以多值的形式使用，附加的第二个值是一个boolean值，表明断言是否成立。如果不成立，第一个值是一个`T`类型的零值（zero-value）。（代码如下）：

```golang
type I interface {
    M()
}

type T1 struct{}

func (T1) M() {}

type T2 struct{}

func (T2) M() {}

func main() {
    var v1 I = T1{}
    v2, ok := v1.(T2)
    if !ok {
        fmt.Printf("ok: %v\n", ok) // ok: false
        fmt.Printf("%v,  %T\n", v2, v2) // {},  main.T2
    }
}
```

这种形式不会引起恐慌，因为返回的第二个布尔类型的值可以用来检查断言是否成立。

### 接口类型

在大多数情况下使用类型断言是没问题的。golang还允许传递接口类型。它会检查动态类型是否满足接口的要求，并且返回该接口类型值的值。在转换条约中，传递给类型断言的接口方法集合不用必须是变量`v`类型的子集（代码如下）：

```golang
type I1 interface {
    M()
}

type I2 interface {
    I1
    N()
}

type T struct{
    name string
}

func (T) M() {}

func (T) N() {}

func main() {
    var v1 I1 = T{"foo"}
    var v2 I2
    v2, ok := v1.(I2)
    fmt.Printf("%T %v %v\n", v2, v2, ok) // main.T {foo} true
}
```

如果接口不满足，那么会返回接口的`零值（zero-value）`，也就是nil（代码如下）：

```golang
type I1 interface {
    M()
}

type I2 interface {
    N()
}

type T struct {}

func (T) M() {}

func main() {
    var v1 I1 = T{}
    var v2 I2
    v2, ok := v1.(I2)
    fmt.Printf("%T %v %v\n", v2, v2, ok) // <nil> <nil> false
}
```

> 当处理接口类型时，还支持类型断言的单值转换。

### nil

当变量`v`是nil，那么类型断言将总是false。不论`T`是一个接口类型还是一个具体类型（代码如下）：

```golang
type I interface {
    M()
}

type T struct{}

func (T) M() {}

func main() {
    var v1 I
    v2 := v1.(T)
    fmt.Printf("%T\n", v2)
}
```

当开始运行这段程序时，程序将会恐慌。

```golang
panic: interface conversion: main.I is nil, not main.T
```

前面介绍的多值类型则可以当`v`是nil的时候，避免出现恐慌(panic)。

## 类型区别（Type switch）

类型断言是一种方法，用于检查接口类型值的动态类型是否实现了所需的接口，或者是否与传递的具体类型的值相同。如果代码需要对一个变量做多次这样的测试，那么golang有一种结构比多个类型断言更加紧凑，和传统的switch语句类似：

```golang
type I1 interface {
    M1()
}

type T1 struct{}

func (T1) M1() {}

type I2 interface {
    I1
    M2()
}

type T2 struct{}

func (T2) M1() {}

func (T2) M2() {}

func main() {
    var v I1
    switch v.(type) {
    case T1:
            fmt.Println("T1")
    case T2:
            fmt.Println("T2")
    case nil:
            fmt.Println("nil")
    default:
            fmt.Println("default")
    }
}
```

这个语法和类型断言非常相似，但是用到了关键字`type`。输出是nil，因为接口类型的值是nil，但是如果我们把变量`v`的值设为：

```golang
var v I1 = T2{}
```

那么程序将会输出`T2`。type switch同样可以作用于接口类型（代码如下）：

```golang
var v I1 = T2{}
switch v.(type) {
case I2:
        fmt.Println("I2")
case T1:
        fmt.Println("T1")
case T2:
        fmt.Println("T2")
case nil:
        fmt.Println("nil")
default:
        fmt.Println("default")
}
```

上面的这段代码会输出`I2`，如果可以匹配到多个接口类型，那么第一个将会被使用（从上到下），如果没有匹配到任何一个类型，那么什么都不会发生。

```golang
type I interface {
    M()
}

func main() {
    var v I
    switch v.(type) {
    }
}
```

这段程序不会恐慌，他会成功的执行完。

### 每个case多种类型

switch语句的每个case可以指定不止一个类型，多个类型用逗号分隔。可以避免匹配到不同类型，执行相同代码块的重复代码。（代码如下）：

```golang
type I1 interface {
    M1()
}

type T1 struct{}

func (T1) M1() {}

type T2 struct{}

func (T2) M1() {}

func main() {
    var v I1 = T2{}
    switch v.(type) {
    case nil:
            fmt.Println("nil")
    case T1, T2:
            fmt.Println("T1 or T2")
    }
}
```

这段代码的输出是`T1 or T2`，因为在断言时，`v`的动态类型是`T2`。

### default case

这种情况类似于之前switch的声明，它会在没有一个case被匹配到的时候执行。（代码如下）：

```golang
var v I
switch v.(type) {
default:
        fmt.Println("fallback")
}
```

### 短变量声明（short variable declaration）

到目前为止，我们已经看到了type switch有如下的语法`v.(type)`，其中`v`是一个类似于变量标识的表达符，除此之外，短变量声明可以在这里使用（代码如下）：

```golang
var p *T2
var v I1 = p
switch t := v.(type) {
case nil:
         fmt.Println("nil")
case *T1:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
case *T2:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
}
```

这段代码会打印`*main.T2 is nil: true`，所以`t`的类型是case的子句。如果在一个单一的case子句有，包含不止一种类型，那么`t`的类型将会和`v`一样（代码如下）：

```golang
var p *T2
var v I1 = p
switch t := v.(type) {
case nil:
         fmt.Println("nil")
case *T1, *T2:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
}
```

这段代码输出是`*main.T2 is nil: false`，变量`t`是接口类型，因为它不是nil，而是指向nil的指针。(InterfaceInGo-part I 解释了接口类型为nil的情况).

### 重复（duplicates）

在case子句中指定的类型一定要是唯一的（代码如下）：

```golang
switch v.(type) {
case nil:
    fmt.Println("nil")
case T1, T2:
    fmt.Println("T1 or T2")
case T1:
    fmt.Println("T1")
}
```

尝试去编译这段代码，将会抛出下面的错误`duplicate case T1 in type switch`。

### 可选简单的语句（optional simple statement）

guard可以用一个简单的语句，类似短变量声明。（代码如下）：

```golang
var v I1 = T1{}
switch aux := 1; v.(type) {
case nil:
    fmt.Println("nil")
case T1:
    fmt.Println("T1", aux)
case T2:
    fmt.Println("T2", aux)
}
```

这段程序将会打印`T1 1`。可以使用附加的语句，无论guard是否以短变量声明的形式出现。