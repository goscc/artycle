# Mocking dependencies in Go
[原文地址（https://medium.com/agrea-technogies/mocking-dependencies-in-go-bb9739fef008）](https://medium.com/agrea-technogies/mocking-dependencies-in-go-bb9739fef008)
## 概括
当我们在写一个程序的时候我们经常会用到很多框架，包括内部开发的框架和一些第三方的框架。我会在接下来为大家介绍[`mockery`](https://github.com/vektra/mockery)和[`stretchr/testify/mock`](https://github.com/stretchr/testify/tree/master/mock)这两个框架，让大家能更简单的进行单元测试的mock
## 让我们更加深入：举个例子
当我们为自己的代码写单元测试的时候，我们可能会发现外部的代码导入的错误。考虑这种情况：
```golang
import "github.com/mediocregopher/radix.v2/redis"
type Handler struct {
    db *redis.Client
}
func (h *Handler) Ping() (string, error) {
    res := h.db.Cmd("INCR", "ping:count")
    if res.Err != nil {
        return "", res.Err
    }
    return "pong", nil
}
```
我们定义了一个struct为`Handler`,其中包含Redis client变量。其中在`Handler`中有一个`Ping`方法，它依赖于Redis client的接口。现在当我们为`Ping`做单元测试的时候，我们想要确认两件事：
+ 什么时候Redis会抛出或者不抛出error
+ 什么时候会返回正确或者不正确的字符串

这些时候我们需要有一个真正的的Redis数据库为了去调用`Cmd`命令。然而当我们为了测试`Ping`方法时，我们不应该依赖于一个真实的Redis。我们真正关心的是`Ping`方法本身和在某些特定的情况下`Ping`方法的运行情况。
### 解决方法：构造一个接口
我们可以看出`db`是通过`*redis.Client`工作的，我们可以看出它是一个指针指向一个*struct*，mock *struct*是很困难的，我们不能用我们实现替换他。所以让我们重构我们的代码，用一个*interface*代替它。
```golang
import "github.com/mediocregopher/radix.v2/redis"
type storager interface {
    Cmd(string, ...interface{}) *redis.Resp
}
type Handler struct {
    db storager
}
func (h *Handler) Ping() (string, error) {
    res := h.db.Cmd("INCR", "ping:count")
    if res.Err != nil {
        return "", res.Err
    }
    return "pong", nil
}
```
现在我们又一个`storager`的*interface*，这个*interface*定义了我们访问Redis的入参和出参。
再让我们看我们需要怎么做这个单元测试
```golang
import (
    "errors"
    "testing"
    "github.com/mediocregopher/radix.v2/redis"
    "github.com/stretchr/testify/assert"
)
func TestPing(t *testing.T) {
    sampleErr := errors.New("sample error")
    tests := map[string]struct {
        storageErr error
        response   string
        err        error
    }{
        "successful": {
            storageErr: nil,
            response:   "pong",
            err:        nil,
        },
        "with db error": {
            storageErr: sampleErr,
            response:   "",
            err:        sampleErr,
        },
    }
    for name, test := range tests {
        t.Logf("Running test case: %s", name)
        storage := &mockStorager{}
        storage.
            On("Cmd", "INCR", []interface{}{"ping:count"}).
            Return(&redis.Resp{
                Err: test.storageErr,
            }).
            Once()
        h := &Handler{
            db: storage,
        }
        response, err := h.Ping()
        assert.Equal(t, test.err, err)
        assert.Equal(t, test.response, response)
        storage.AssertExpectations(t)
    }
}
```
让我们一步一步的看
+ 首先，我设置了在我的map中设置了两种测试情况，第一个测试数据库不返回错误，第二个返回错误。
+ 我定义了一个`storage`变量，类型为`*mockStorager`，在接下来我会解释为什么会这么做。同时可以注意到我定义了输入的参数和对应的返回值。
+ 然后我验证了方法的返回值
+ `storage.AssertExpectations(t)`判断所有通过`storage`mock的函数返回值，是否全部都调用了。在这个这段代码中`Cmd("INCR", "ping:count")`被调用调用，并且只被调用一次。

现在，`*mockStorager`是我通过(mockery)[https://github.com/vektra/mockery]产生出的一个struct，（mockery是一个自动生成mock struct的工具，如果你没有用过这个，请仔细阅读）

```
mockery -name storager -inpkg .
```


