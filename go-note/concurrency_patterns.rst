.. _go_concurrency_patterns:

Go 并发设计模式
=====================================================================

屏障模式(Barrier Mode)
--------------------------------------------------
屏障模式(Barrier Mode)，用来阻塞goroutine直到聚合所有goroutine返回结果，可以用通道实现。使用场景：

1. 多个网络请求并发，聚合结果
2. 粗粒度任务拆分并发执行，聚合结果

.. code-block:: go

    package main

    import (
        "fmt"
        "io/ioutil"
        "net/http"
        "time"
    )

    // 封装屏障模式响应的结构体(一般就是返回值和错误)
    type BarrierResponse struct {
        Err    error
        Resp   string
        Status int
    }

    // 执行单个请求，结果放入 channel
    func doRequest(out chan<- BarrierResponse, url string) {
        res := BarrierResponse{}
        client := http.Client{Timeout: time.Duration(3 * time.Second)}
        resp, err := client.Get(url)
        if resp != nil {
            res.Status = resp.StatusCode
        }
        if err != nil {
            res.Err = err
            out <- res
            return
        }

        b, err := ioutil.ReadAll(resp.Body)
        defer resp.Body.Close()
        if err != nil {
            res.Err = err
            out <- res
            return
        }
        res.Resp = string(b)
        out <- res //  结果放入通道
    }

    // 并发请求并聚合结果
    func Barrier(urls ...string) {
        n := len(urls)
        in := make(chan BarrierResponse, n)
        response := make([]BarrierResponse, n)
        defer close(in)

        for _, url := range urls {
            go doRequest(in, url)
        }

        var hasError bool
        for i := 0; i < n; i++ {
            resp := <-in
            if resp.Err != nil {
                fmt.Println("Error: ", resp.Err, resp.Status)
                hasError = true
            }
            response[i] = resp
        }
        if !hasError {
            for _, resp := range response {
                fmt.Println(resp.Status)
            }
        }
    }

    func main() {
        urls := []string{
            "https://www.baidu.com",
            "https://www.weibo.com",
            "https://www.zhihu.com",
        }
        Barrier(urls...)
    }

未来模式(Future Mode)
--------------------------------------------------
Future模式(也称为Promise Mode)。使用 `fire-and-forget` 方式，主进程不等子进程执行完就直接返回，然后等到未来执行完的时候再去获取结果。
未来模式中主goroutine不用等待子goroutine返回的结果，可以先去做其他事情，等未来需要子goroutine结果的时候再来取。
如果子goroutine还没有返回结果，则一直等待。以下简单的代码示例说明了该模式的原理：

.. code-block:: go

    c := make(chan int)      // future
    go func() { c <- f() }() // async
    value := <-c             // await

可以针对 future 模式做一个统一的封装，方便后续使用，代码示例如下：

.. code-block:: go

    /* https://github.com/golang-collections/go-datastructures/blob/59788d5eb259/futures/futures.go
    Package futures is useful for broadcasting an identical message to a multitude
    of listeners as opposed to channels which will choose a listener at random
    if multiple listeners are listening to the same channel.  The future will
    also cache the result so any future interest will be immediately returned
    to the consumer.
    */
    package main

    import (
        "fmt"
        "sync"
        "time"
    )

    // Completer is a channel that the future expects to receive
    // a result on.  The future only receives on this channel.
    type Completer <-chan interface{}

    // Future represents an object that can be used to perform asynchronous
    // tasks.  The constructor of the future will complete it, and listeners
    // will block on getresult until a result is received.  This is different
    // from a channel in that the future is only completed once, and anyone
    // listening on the future will get the result, regardless of the number
    // of listeners.
    type Future struct {
        triggered bool // because item can technically be nil and still be valid
        item      interface{}
        err       error
        lock      sync.Mutex
        wg        sync.WaitGroup
    }

    // GetResult will immediately fetch the result if it exists
    // or wait on the result until it is ready.
    func (f *Future) GetResult() (interface{}, error) {
        f.lock.Lock()
        if f.triggered {
            f.lock.Unlock()
            return f.item, f.err
        }
        f.lock.Unlock()

        f.wg.Wait()
        return f.item, f.err
    }

    func (f *Future) setItem(item interface{}, err error) {
        f.lock.Lock()
        f.triggered = true
        f.item = item
        f.err = err
        f.lock.Unlock()
        f.wg.Done()
    }

    func listenForResult(f *Future, ch Completer, timeout time.Duration, wg *sync.WaitGroup) {
        wg.Done()
        select {
        case item := <-ch:
            f.setItem(item, nil)
        case <-time.After(timeout):
            f.setItem(nil, fmt.Errorf(`Timeout after %f seconds.`, timeout.Seconds()))
        }
    }

    // New is the constructor to generate a new future.  Pass the completed
    // item to the toComplete channel and any listeners will get
    // notified.  If timeout is hit before toComplete is called,
    // any listeners will get passed an error.
    func New(completer Completer, timeout time.Duration) *Future {
        f := &Future{}
        f.wg.Add(1)
        var wg sync.WaitGroup
        wg.Add(1)
        go listenForResult(f, completer, timeout, &wg)
        wg.Wait()
        return f
    }

    // 使用示例
    func main() {
        c := make(chan interface{})

        go func() {
            time.Sleep(time.Second)
            c <- "hehe"
        }()

        f := New(c, time.Second*3)
        res, err := f.GetResult()
        fmt.Println(res, err)
    }