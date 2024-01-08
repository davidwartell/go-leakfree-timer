# go-cancel-lock

A timer that helps with the pitfalls of using golang timers in loops.

For example, the following code leaks timers because time.After() does not garbage collect until the timer expires:
```go
for {
    select {
    case somePayload, ok := <-someChan:
        if !ok {
            // channel is closed exit
            return
        }
        doSomething()
    checkSomethingLoop::
        for _, foo := range someSlice {
            // It's possible we could block for a long time on writing to channel. If we cant write after Xms abort
            select {
            case writeChan <- "foo":
                doSomething
            case <-time.After(time.Duration(25) * time.Millisecond):
                break checkSomethingLoop
            }
        }
    }
}
````
See: https://pkg.go.dev/time#After

The obvious solution is to use timer := time.NewTimer(time.Duration(25) * time.Millisecond) before the loop and reset it, 
but it turns out this has complications:
https://pkg.go.dev/time#NewTimer
https://github.com/golang/go/issues/11513

CockroachDB has a nice solution to encapsulate the complexity.
This code is copied from CockroachDB when it was released under an Apache license in 2019:
https://github.com/cockroachdb/cockroach/blob/4d82429ba71d1d2a868ebff38f6d8f7ce3595d21/pkg/util/timeutil/timer.go

## Usage

The Timer type represents a single event. When the Timer expires, the current time will be sent on Timer.C.

This timer implementation is an abstraction around the standard library's time.Timer that provides a workaround for the 
issue described in https://github.com/golang/go/issues/14038. As such, this timer should only be used when Reset is planned to
be called continually in a loop. For this Reset pattern to work, Timer.Read must be set to true whenever a timestamp is read from
the Timer.C channel. If Timer.Read is not set to true when the channel is read from, the next call to Timer.Reset will deadlock.

Example:
```go
 var tmr timer.Timer
 defer tmr.Stop()
 for {
     tmr.Reset(wait)
     select {
     case <-tmr.C:
         tmr.Read = true
         ...
     }
 }
```

Note that unlike the standard library's Timer type, this Timer will
not begin counting down until Reset is called for the first time, as
there is no constructor function.

## Contributing

Happy to accept PRs.

# Author

**davidwartell**

* <http://github.com/davidwartell>
* <http://linkedin.com/in/wartell>

## License

Released under the [Apache License](https://github.com/davidwartell/go-commons-drw/blob/master/LICENSE).
