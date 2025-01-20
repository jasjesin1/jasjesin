logr offers an(other) opinion on how Go programs and libraries can do logging without becoming coupled to a particular logging implementation
- This is not an implementation of logging - it is an API 
- In fact it is two APIs with two different sets of users
	- The `Logger` type is intended for application and library authors
		- It provides a relatively small API which can be used everywhere you want to emit logs
		- It defers the actual act of writing logs _(to files, to stdout, or whatever)_ to the `LogSink` interface
	- The `LogSink` interface is intended for logging library implementers
		- It is a pure interface which can be implemented by logging frameworks to provide the actual logging functionality
- This decoupling allows application and library developers to write code in terms of 
	- `logr.Logger` _(which has very low dependency fan-out)_ while the implementation of logging is managed "up stack" _(e.g. in or near `main()`)_ 
	- Application developers can then switch out implementations as necessary
- Package logr defines a general-purpose logging API and abstract interfaces to back that API
- Packages in the Go ecosystem can depend on this package
- **Usage:**
	- Logging is done using a Logger instance
	- Logger is a concrete type with methods, which defers the actual logging to a LogSink interface
	- The main methods of Logger are Info() and Error()
	- Arguments to Info() and Error() are key/value pairs rather than printf-style formatted strings, emphasizing "structured logging"
- With Go's standard log package, we might write:
```go
log.Printf("setting target value %s", targetValue)
log.Printf("failed to open the pod bay door for user %s: %v", user, err)
```
- With logr's structured logging, we'd write:
```go
logger.Info("setting target", "value", targetValue)
logger.Error(err, "failed to open the pod bay door", "user", user)
```
- Error() messages are always logged, regardless of the current verbosity
	- If there is no error instance available, passing nil is valid

- **Typical usage:**
	- Somewhere, early in an application's life, 
		- it will make a decision about which logging library (implementation) it actually wants to use
		- Something like:
```go
    func main() {
        // ... other setup code ...

        // Create the "root" logger.  We have chosen the "logimpl" implementation,
        // which takes some initial parameters and returns a logr.Logger.
        logger := logimpl.New(param1, param2)

        // ... other setup code ...
```
- Most apps will call into other libraries, create structures to govern the flow, etc
- The `logr.Logger` object can be 
	- passed to these other libraries, 
	- stored in structs, or even 
	- used as a package-global variable, if needed 
- For example:
```go
    app := createTheAppObject(logger)
    app.Run()
```
- Outside of this early setup, no other packages need to know about the choice of implementation. 
	- They write logs in terms of the `logr.Logger` that they received:
```go
    type appObject struct {
        // ... other fields ...
        logger logr.Logger
        // ... other fields ...
    }

    func (app *appObject) Run() {
        app.logger.Info("starting up", "timestamp", time.Now())

        // ... app code ...
```
- **Background:**
	- If the Go standard library had defined an interface for logging, 
		- this project probably would not be needed
	- When the Go developers started developing such an interface with [slog](https://github.com/golang/go/issues/56345), 
		- they adopted some of the `logr` design but also left out some parts and changed others:
	- The high-level slog API is explicitly meant to be one of many different APIs that can be layered on top of a shared `slog.Handler`
	- logr is one such alternative API, with [interoperability](https://pkg.go.dev/github.com/go-logr/logr#readme-slog-interoperability) provided by some conversion functions



----------
**Why structured logging?**
- Structured logs are more easily queryable
	- Since you've got key-value pairs, it's much easier to query your structured logs for particular values by filtering on the contents of a particular key
		- think searching request logs for error codes, 
			- Kubernetes reconcilers for the name and namespace of the reconciled object, etc
- Structured logging makes it easier to have cross-referenceable logs
	- if you maintain conventions around your keys, 
		- it becomes easy to gather all log lines related to a particular concept
- Structured logs allow better dimensions of filtering
	- if you have structure to your logs, 
		- you've got more precise control over how much information is logged
		- you might choose in a particular configuration to log certain keys but not others, 
			- only log lines where a certain key matches a certain value, etc., 
			- instead of just having v-levels and names to key off of
- Structured logs better represent structured data
	- data that you want to log is inherently structured _(think tuple-link objects.)_ 
	- Structured logs allow you to preserve that structure when outputting


**Why not allow format strings, too?**
   Format strings negate many of the benefits of structured logs
- They're not easily searchable 
	- without resorting to fuzzy searching, regular expressions, etc
- They don't store structured data well, 
	- since contents are flattened into a string
- They're not cross-referenceable    
- They don't compress easily, since the message is not constant


