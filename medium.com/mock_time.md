# Mocking time with Go
[原文地址（https://medium.com/agrea-technogies/mocking-time-with-go-a89e66553e79）](https://medium.com/agrea-technogies/mocking-time-with-go-a89e66553e79)

我经常发现一些开发人员在处理程序中“时间”问题付出较多的努力，所以我想在下面提出一些关于我的建议。
譬如我们需要测试下面这段代码
```go
import (
    "fmt"
    "time"
)
func WhatTimeIsIt() string {
    return fmt.Sprintf("It's %d", time.Now().Unix())
}
```
因为我们时间一直在变化中，所以当我们在不同时间运行单元测试时会获得两个不同的时间变量。这对我们单元测试带来一些困难。

[jonboulle/clockwork](https://github.com/jonboulle/clockwork)为我们解决了这个问题，他允许我们mock时间，这样我们就对我们测试代码又一个可预期的结果

你需要把你的代码重构成下面这个样子
```go
import (
    "fmt"
    "github.com/jonboulle/clockwork"
)
func WhatTimeIsIt(clock clockwork.Clock) string {
    return fmt.Sprintf("It's %d", clock.Now().Unix())
}
```
 可能看到这你会说“这多么愚蠢的写法”。其实我想要告诉大家的非常简单，`time`需要被mock，而且`jonboulle/clockwork`提供了这个功能。

 现在我们可以把我们的单元测试代码写成下面这个样子
 ```go
 import (
    "fmt"
    "testing"
    "github.com/jonboulle/clockwork"
    "github.com/stretchr/testify/assert"
)
func TestWhatTimeIsIt(t *testing.T) {
    tests := map[string]struct {
        clock clockwork.Clock
    }{
        "basic test": {
            clock: clockwork.NewFakeClock(),
        },
    }
    for name, test := range tests {
        t.Logf("Running test case: %s", name)
        output := WhatTimeIsIt(test.clock)
        assert.Equal(
            t,
            fmt.Sprintf("It's %d", test.clock.Now().Unix()),
            output,
        )
    }
}
 ```
 好了，这段单元测试代码非常的简单，我们总结下
 1. 在测试时，我们传入了一个mock的时间给`WhatTimeIsIt()`
 2. 在生产环境，我们需要传入一个真正的时间如`WhatTimeIsIt(clockwork.NewRealClock())`