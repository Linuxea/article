```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {

	// create a context with `context.Background()`
	// create a derived context with `context.WithCancel`
	cancel, cancelFunc := context.WithCancel(context.Background())
	go doWork(cancel)
	// wait for seconds
	time.Sleep(3 * time.Second)
	// invoke cancel func
	cancelFunc()
	// keep main goroutine live
	time.Sleep(time.Hour)

}

func doWork(ctx context.Context) {

	ch := make(chan int)
	// it will never trigger `follow case <-ch:`, because a long time sleep longer than parent context cancel
	go sleep(time.Hour, ch)

	select {
	case <-ch:
		fmt.Println("the sleep is done")
		return
	case <-ctx.Done():
		fmt.Println("the parent context is cancel")
		// clean resource and balabala
		return
	}

}

func sleep(duration time.Duration, ch chan int) {
	time.Sleep(duration)
	ch <- 1
}

```
