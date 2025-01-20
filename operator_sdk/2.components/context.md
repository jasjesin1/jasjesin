
- The [context](https://golang.org/pkg/context/) is used to 
	- allow cancellation of requests, and potentially 
	- things like tracing
	- It’s the first argument to all client methods
		- The `Background` context is just a basic context without any extra data or timing restrictions
			- `Background` is the root of any `Context` tree; it is never canceled:
		- The `Done` method returns a channel that acts as a cancellation signal to functions running on behalf of the `Context`: 
			- when the channel is closed, the functions should abandon their work and return
		- The `Err` method returns an error indicating why the `Context` was canceled
		- A `Context` does _not_ have a `Cancel` method for the same reason the `Done` channel is receive-only: 
			- the function receiving a cancellation signal is usually not the one that sends the signal
	- In particular, when a parent operation starts goroutines for sub-operations, 
		- those sub-operations should not be able to cancel the parent
	- Package context defines the 
		- Context type, which carries deadlines, 
		- cancellation signals, and 
		- other request-scoped values across API boundaries and between processes
	- Incoming requests to a server should create a [Context](https://pkg.go.dev/context#Context), and 
		- outgoing calls to servers should accept a Context
	- Programs that use Contexts should follow these rules to 
		- keep interfaces consistent across packages and 
		- enable static analysis tools to check context propagation
	- DO NOT store Contexts inside a struct type; 
		- instead, pass a Context explicitly to each function that needs it. 
	- The Context should be the first parameter, typically named ctx
	- Do not pass a nil [Context](https://pkg.go.dev/context#Context), even if a function permits it. 
		- Pass [context.TODO](https://pkg.go.dev/context#TODO) if you are unsure about which Context to use.
	- Use context Values only for request-scoped data that transits processes and APIs, 
		- NOT for passing optional parameters to functions.
	- The same Context may be passed to functions running in different goroutines; 
		- Contexts are safe for simultaneous use by multiple goroutines.
```go
func DoSomething(ctx context.Context, arg Arg) error {
	// ... use ctx ...
}

type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline [time].[Time], ok [bool])
	// Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```
- The logging handle lets us log 
	- controller-runtime uses structured logging through a library called [logr](https://github.com/go-logr/logr)
	- logging works by attaching key-value pairs to a static message
		- We can pre-assign some pairs at the top of our reconcile method to have those attached to all log lines in this reconciler
```go
func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```
- Finally, we add this reconciler to the manager, so that it gets started when the manager is started
- For now, we just note that this reconciler operates on `CronJob`s
	- Later, we’ll use this to mark that we care about related objects as well
```go
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}
```

