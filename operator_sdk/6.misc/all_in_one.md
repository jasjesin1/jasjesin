
- <font style="color:green"><b>operator</b></font> -- considered as client to API svr, to run ctrller to reconcile state of custom stateful apps, wid help of CR
- <font style="color:red"><b>Process main.go</b></font> -- main entrypoint, that is executed to instantiate mgr
- <font style="color:orange"><b>Manager</b></font> -- orchestrates everything for us, by launching pods for ctrllers, clients, caches & webhooks; and monitoring these
- <font style="color:magenta"><b>client</b></font> -- help access k8s API objects by taking care of authentication & protocols
- <font style="color:violet"><b>cache</b></font> -- store recent requests for objects, by ctrllers n webhooks; used by client for faster txn
- <font style="color:purple"><b>controller</b></font> -- contain actual business logic & use predicates to filter events (with event src & handler) for triggering reconcile requests
- <font style="color:gold"><b>event src & handler</b></font> -- event src watches for k8s API object changes; event handler processes event like queuing up event request for object owner
- <font style="color:teal"><b>predicate</b></font> -- filter out which events to consider for reconciliation
- <font style="color:yellow"><b>scheme</b></font> -- provides mappings b/w kinds & associated Go-types, to ctrller
- <font style="color:slateblue"><b>types</b></font> -- contain schema of inputs to be provided via CR, for constructing a CRD
- <font style="color:pink"><b>reconciler</b></font> -- implements a fn tht takes reconcile request, containing name & ns of object to reconcile; reconciles object, if needed & returns response
- <font style="color:brown"><b>event recorders</b></font> -- emit events using mgr
- <font style="color:Tomato"><b>boilerplate</b></font> -- foundational template that provides pre-defined set of files wid necessary structure, reusable code patters & configurations as a quick start base for building k8s operators, wid minimal changes; that follows best practices
- <font style="color:dodgerblue"><b>scaffold</b></font> -- process & tools that generate initial project structure, including integration boilerplate code
- <font style="color:mediumseagreen"><b>Markers</b></font> -- acts as extra metadata, telling [controller-tools](https://github.com/kubernetes-sigs/controller-tools) _(our code and YAML generator)_ extra information


![[kubeBuilder_Architecture 1.png]]
**kubeBuilder Architecture:**

- every functional object needs to contain a spec & a status; spec holds desired state, so any inputs to ctrller go here
- Any new fields you add must have json tags for the fields to be serialized.
- We can also use the `omitempty` struct tag to mark that a field should be omitted from serialization when empty
- design our statue to hold observed state; this contains info tht we want our users or other ctrllers to obtain
- use `metav1.Time` instead of `time.Time` to get the stable serialization



operator Architecture
scaffolding
APIs
Glossary
Controller, Reconciler &Context


**Operator**
- go-progam, considered as client to API svr
- runs ctrller to reconcile state of custom stateful apps, with the help of CRs
- built on top of kubeBuilder
	- can leverage functions/features of kubeBuilder in operator, as markers
- Each set of controllers needs a [_Scheme_](https://book.kubebuilder.io/cronjob-tutorial/gvks#err-but-whats-that-scheme-thing), that provides mappings between Kinds and their corresponding Go types
- Based on the above design diagram, 

**Manager** 
- instantiate a [_manager_](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager?tab=doc#Manager), which keeps track of 
	- running all of our controllers, as well as 
	- setting up shared caches and clients to the API server _(notice we tell the manager about our Scheme)_
- We run manager, which, in turn, runs everything in operator for us, like all our controllers & webhooks
- When a [Manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) instance is created in the `main.go` file, [Manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) is initialized with no Cache option specified, 
	- or with a Cache.DefaultNamespaces of `Namespace: ""` will watch all Namespaces
	- `Manager` can restrict the namespace that all controllers will watch for resources by
		```Go
		mgr, err := ctrl.NewManager(cfg, manager.Options{Namespace: namespace})
		```
- Mgr is orchestrator responsible to 
	- start/stop operator, 
	- register scheme _(framework/schema that maps k8s API kinds wid associated Go types)_ for CR
		- Each set of controllers needs a [_Scheme_](https://book.kubebuilder.io/cronjob-tutorial/gvks#err-but-whats-that-scheme-thing), 
			- Scheme provides mappings between Kinds and their corresponding Go types
		- Based on provided Scheme, Manager keeps track of 
			- running all of our controllers, as well as 
			- setting up shared caches and clients to the API server
		- This helps operator to understand & work wid CRs
	- setup & manage
		- ctrller(s)
			- These watch for changes to specific resource types (CRs) & take action towards reconciliation
		- event-handling loop
			- This listens to changes in resources & send it to ctrllers
		- webhook(s)
			- These allow for custom logic to be executed wen resources r CRUD
	- Monitoring Resources changes & Reconciliation
	- Cfg & dependency injection

- initialize project
	- creates a project for us, wid some scaffolding
		- From this, we create the types & ctrller loop
	- After project is initialized, we see folder structure wid files created
```sh
export GO111MODULE=on      # helps activate module support
operator-sdk init --domain example.com --repo github.com/example/memcached-operator --plugins=go/v4
```
- domain is used as prefix of API grp that CRs will be created in.
	- These API grps r used internally to version k8s resources
	- Domain is helpful to group our resource types together
		- These domains/API Groups help determine how access can be controlled for our resource type, via RBAC
- This creates `kustomize` manifests and scaffolds in the following folder structure
	- `api/v1alpha1/scaler_types.go`
		- API is the place whr we define structure of our Custom Resource n how it should look like
	- `controllers/scaler_controller.go`
		- Controller is the place whr we define business logic like how we want to scale up/down, via reconcile logic


- [`RBAC markers`](https://book.kubebuilder.io/reference/markers/rbac.html) defined in the controller (e.g `controllers/memcached_controller.go`) are used to generate the operator’s [RBAC ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) (e.g `config/rbac/role.yaml`). 
	- The default markers don’t specify a `namespace` property and will result in a `ClusterRole`.


```sh
make generate
```
- The above makefile target will invoke the [controller-gen](https://sigs.k8s.io/controller-tools) utility to update the `api/v1alpha1/zz_generated.deepcopy.go` file to ensure our API’s Go type definitions implement the `runtime.Object` interface that all Kind types must implement.
- OpenAPI validation defined in a CRD ensures CRs are validated based on a set of declarative rules. All CRDs should have validation.

-------------

**Basic Project**
   When scaffolding out a new project, Kubebuilder provides us with a few basic pieces of boilerplate.
- ## [Build Infrastructure](https://book.kubebuilder.io/cronjob-tutorial/basic-project.html#build-infrastructure) 
  basic infrastructure for building your project:
	- `go.mod`: A new Go module matching our project, with basic dependencies
	- `Makefile`: Make targets for building and deploying your controller
	- `PROJECT`: Kubebuilder metadata for scaffolding new components
- ## [Launch Configuration](https://book.kubebuilder.io/cronjob-tutorial/basic-project.html#launch-configuration)
	- under the [`config/`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config) contains
		- [Kustomize](https://sigs.k8s.io/kustomize) YAML definitions required to launch our controller on a cluster, 
			- once we get started writing our controller, it’ll also hold our 
				- CustomResourceDefinitions, 
				- RBAC configuration, and 
				- WebhookConfigurations
	- [`config/default`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/default) contains a [Kustomize base](https://github.com/kubernetes-sigs/kubebuilder/blob/master/docs/book/src/cronjob-tutorial/testdata/project/config/default/kustomization.yaml) for launching the controller in a standard configuration
	- Each other directory contains a different piece of configuration, refactored out into its own base:
		- [`config/manager`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/manager): launch your controllers as pods in the cluster
		- [`config/rbac`](https://github.com/kubernetes-sigs/kubebuilder/tree/master/docs/book/src/cronjob-tutorial/testdata/project/config/rbac): permissions required to run your controllers under their own service account
- ## [The Entrypoint](https://book.kubebuilder.io/cronjob-tutorial/basic-project.html#the-entrypoint)
	- Kubebuilder scaffolds out the basic entrypoint of our project: `main.go`.



---
**controller-runtime**
- set of go libraries for building Controllers, leveraged by [Kubebuilder](https://book.kubebuilder.io/) and [Operator SDK](https://github.com/operator-framework/operator-sdk)
- The main entry-point for controller-runtime is this root package
	- it contains all of the common types needed to get started building controllers:
- A brief-ish walkthrough of the layout of this library
	- **Controllers:**
		- Controllers _(pkg/controller)_ use events _(pkg/event)_ to eventually trigger reconcile requests. 
		- Controllers are often constructed with a Builder _(pkg/builder)_, 
			- This eases the wiring of 
				- event sources _(pkg/source)_, like `Kubernetes API object changes`, to 
				- event handlers _(pkg/handler)_, like `enqueue a reconcile request for the object owner`
		- Predicates _(pkg/predicate)_ can be used to filter which events actually trigger reconciles.
	- **Reconcilers:**
		- Controller logic is implemented in terms of Reconcilers _(pkg/reconcile)_ 
		- A Reconciler implements a function which 
			- takes a reconcile Request containing the name and namespace of the object to reconcile, 
			- reconciles the object, and 
			- returns a Response or an error indicating whether to requeue for a second round of processing
	- **Clients and Caches:**
		- Reconcilers use Clients _(pkg/client)_ to access API objects. 
		- The default client provided by the manager 
			- reads from a local shared cache _(pkg/cache)_ and 
			- writes directly to the API server
		- clients can be constructed that only talk to the API server, without a cache
		- The Cache will auto-populate with watched objects, as well as when other structured objects are requested
		- The default split client 
			- does not promise to invalidate the cache during writes _(nor does it promise sequential create/get coherence)_, and 
			- code should not assume a get immediately following a create/update will return the updated resource. 
		- Caches may also have indexes, which can be created via a FieldIndexer _(pkg/client)_ obtained from the manager. 
			- Indexes can used to quickly and easily look up all objects with certain fields set. 
		- Reconcilers may retrieve event recorders _(pkg/recorder)_ to emit events using the manager.
	- **Schemes:**
		- Clients, Caches, and many other things in Kubernetes use Schemes _(pkg/scheme)_ to associate 
			- Go types to 
			- Kubernetes API Kinds _(Group-Version-Kinds, to be specific)_
	- **Webhooks:**
		- webhooks _(pkg/webhook/admission)_ are often constructed using a builder _(pkg/webhook/admission/builder)_ 
		- They are run via a server _(pkg/webhook)_ which is managed by a Manager
	- **Logging and Metrics:**
		- Logging _(pkg/log)_ is done via structured logs, using a log set of interfaces called logr ([https://pkg.go.dev/github.com/go-logr/logr](https://pkg.go.dev/github.com/go-logr/logr))
			- controller-runtime provides easy setup for using Zap ([https://go.uber.org/zap](https://go.uber.org/zap), _pkg/log/zap_)
		- Metrics _(pkg/metrics)_ are registered into a controller-runtime-specific Prometheus metrics registry
			- The manager can serve these by an HTTP endpoint, and 
			- additional metrics may be registered to this Registry as normal
	- **Testing:**
		- Build integration and unit tests for your controllers and webhooks using the test Environment _(pkg/envtest)_
		- This will automatically 
			- stand up a copy of etcd and kube-apiserver, and 
			- provide the correct options to connect to the API server
		- It's designed to work well with the Ginkgo testing framework
```go
import (
	ctrl "sigs.k8s.io/controller-runtime"
)
```




---
**logr**
- logr offers an(other) opinion on how Go programs and libraries can do logging without becoming coupled to a particular logging implementation
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



--------
**zap**
- Package zap provides fast, structured, leveled logging
- For applications that log in the hot path, 
	- reflection-based serialization and string formatting are prohibitively expensive 
		- they're CPU-intensive and make many small allocations
	- Put differently, using `json.Marshal` and `fmt.Fprintf` to log tons of interface{} makes your application slow
- Zap takes a different approach
	- It includes a 
		- reflection-free, 
		- zero-allocation JSON encoder, and the 
		- base Logger strives to avoid serialization overhead and 
			- allocations wherever possible
	- By building the high-level SugaredLogger on that foundation, 
		- zap lets users choose 
			- when they need to count every allocation and 
			- when they'd prefer a more familiar, loosely typed API

- **Choosing a Logger:**
	- In contexts where performance is nice, but not critical, use the SugaredLogger 
		- It's 4-10x faster than other structured logging packages and 
			- supports both structured and printf-style logging
	- Like log15 and go-kit, 
		- SugaredLogger's structured logging APIs are loosely typed and accept a variadic number of key-value pairs
```go
sugar := zap.NewExample().Sugar()
defer sugar.Sync()
sugar.Infow("failed to fetch URL",
  "url", "http://example.com",
  "attempt", 3,
  "backoff", time.Second,
)
sugar.Infof("failed to fetch URL: %s", "http://example.com")
```
- By default, loggers are unbuffered
	- However, since zap's low-level APIs allow buffering, 
		- calling Sync before letting your process exit is a good habit
- In the rare contexts where every microsecond and every allocation matter, 
	- use the Logger
	- It's even faster than the SugaredLogger and allocates far less, 
		- but it only supports strongly-typed, structured logging
```go
logger := zap.NewExample()
defer logger.Sync()
logger.Info("failed to fetch URL",
  zap.String("url", "http://example.com"),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Second),
)
```
- Choosing between the Logger and SugaredLogger doesn't need to be an application-wide decision: 
	- converting between the two is simple and inexpensive
```go
logger := zap.NewExample()
defer logger.Sync()
sugar := logger.Sugar()
plain := sugar.Desugar()
```
- **Configuring Zap:**
	- The simplest way to build a Logger is to use zap's opinionated presets: 
		- NewExample, 
		- NewProduction, and 
		- NewDevelopment 
	- These presets build a logger with a single function call:
```go
logger, err := zap.NewProduction()
if err != nil {
  log.Fatalf("can't initialize zap logger: %v", err)
}
defer logger.Sync()
```
- Presets are fine for small projects, 
	- but larger projects and organizations naturally require a bit more customization
- More unusual configurations _(splitting output between files, sending logs to a message queue, etc.)_ are possible, 
	- but require direct use of go.uber.org/zap/zapcore





----------------
**main.go**
   Our package starts out with some basic imports. Particularly:
- The default controller-runtime logging, [Zap](https://pkg.go.dev/go.uber.org/zap) 
- The core [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime?tab=doc) library
```go
package main

import (
    "flag"
    "os"

    // Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
    // to ensure that exec-entrypoint and run can make use of them.
    _ "k8s.io/client-go/plugin/pkg/client/auth"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    "sigs.k8s.io/controller-runtime/pkg/metrics/server"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    // +kubebuilder:scaffold:imports
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    // +kubebuilder:scaffold:scheme
}
```
- At this point, our main function is fairly simple:
	- We set up some basic flags for metrics.
	- We instantiate a [_manager_](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager?tab=doc#Manager)
- While we don’t have anything to run just yet, remember where that `+kubebuilder:scaffold:builder` comment is – things’ll get interesting there soon.
```go
func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false,
        "Enable leader election for controller manager. "+
        "Enabling this will ensure there is only one active controller manager.")
    opts := zap.Options{
        Development: true,
    }
    opts.BindFlags(flag.CommandLine)
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
        Metrics: server.Options{
            BindAddress: metricsAddr,
        },
        WebhookServer:          webhook.NewServer(webhook.Options{Port: 9443}),
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    // +kubebuilder:scaffold:builder

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

With that out of the way, we can get on to scaffolding our API!

---

**Why APIs?**
- APIs help teach Kubernetes about our custom objects. 
- The Go structs are used to generate a CRD which includes 
	- schema for our data as well as 
	- tracking data like what our new type is called. 
- We can then create instances of our custom objects which will be managed by our [controllers](https://book.kubebuilder.io/cronjob-tutorial/controller-overview).
- Our APIs and resources represent our customized solutions on the clusters, that are not avl natively. 
- Basically, 
	- CRDs are a definition of our customized Objects, and 
	- CRs are an instance of it.

**API**s contain 
- groups
	- collection of related functionality
	- Each group has one or more _versions_
- versions
	- allow us to change how an API works over time
- kinds
	- Each API group-version contains one or more API types, that is called _kinds_
	- With CRDs, however, each Kind will correspond to a single resource
- resources
	- use of a Kind in the API
	- resources are always lowercase form of the Kind

- Often, there’s a one-to-one mapping between Kinds and resources. 
	- For instance, the `pods` resource corresponds to the `Pod` Kind. 
- However, sometimes, the same Kind may be returned by multiple resources. 
	- For instance, the `Scale` Kind is returned by all scale subresources, like `deployments/scale` or `replicasets/scale`. 
- This is what allows the Kubernetes HorizontalPodAutoscaler to interact with different resources. 







----
**GO**
- When we refer to a kind in a particular group-version, 
	- we’ll call it a _GroupVersionKind_, or GVK for short. 
- Same with resources and GVR. 
- each GVK corresponds to a given root Go type in a package.
- `Scheme` is a way to keep track of what Go type corresponds to a given GVK
	- **Example:** suppose we mark the 
		- `"tutorial.kubebuilder.io/api/v1".CronJob{}` type as being in the 
		- `batch.tutorial.kubebuilder.io/v1` API group _(implicitly saying it has the Kind `CronJob`)_.
	- Then, we can later construct a new `&CronJob{}` given some JSON from the API server that says
```yaml
{     
	"kind": "CronJob",     
	"apiVersion": "batch.tutorial.kubebuilder.io/v1",     
	... 
}
```








---
**Scaffold out a new project**
```bash
# create a project directory, and then run the init command. 
mkdir project 
cd project 
# we'll use a domain of tutorial.kubebuilder.io, 
# so all API groups will be <group>.tutorial.kubebuilder.io. 
kubebuilder init --domain tutorial.kubebuilder.io --repo tutorial.kubebuilder.io/project
```
- Project’s name defaults to current working directory. 
	- You can pass `--project-name=<dns1123-label-string>` to set a different project name.

```go
kubebuilder create api --group batch --version v1 --kind CronJob
```
- The goal of this command is to create 
	- Custom Resource (CR) and 
	- Custom Resource Definition (CRD) for our Kind(s)
- This command for each group-version, will create a directory for the new group-version.
	- In this case, the [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) directory is created, 
		- corresponding to the `batch.tutorial.kubebuilder.io/v1` (remember our [`--domain` setting](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial#scaffolding-out-our-project) from the beginning?).
- It has also added a file for our `CronJob` Kind, `api/v1/cronjob_types.go`. 
- Each time we call the command with a different kind, it’ll add a corresponding new file.
- We start out by importing `meta/v1` API group
	- This is not usually exposed by itself, but instead contains metadata common to all Kubernetes Kinds.
```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```
- Kubernetes functions by reconciling 
	- desired state (`Spec`) with actual cluster state (other objects’ `Status`) and external state, and then 
	- recording what it observed (`Status`). 
- Thus, every _functional_ object includes spec and status
```go
// CronJobSpec defines the desired state of CronJob 
type CronJobSpec struct { 
} 

// CronJobStatus defines the observed state of CronJob 
type CronJobStatus struct { 
}
```

- all Kubernetes objects contain 
	- `TypeMeta` 
		- describes API version and Kind
	- `ObjectMeta`
		- holds things like name, namespace, and labels






---
**Marker**
- acts as extra metadata, telling [controller-tools](https://github.com/kubernetes-sigs/controller-tools) _(our code and YAML generator)_ extra information
- This particular one tells the `object` generator that this type represents a Kind. 
- Then, the `object` generator generates an implementation of the [runtime.Object](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Object) interface for us, 
	- This is the standard interface that all types representing Kinds must implement.
```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CronJobSpec   `json:"spec,omitempty"`
	Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CronJob `json:"items"`
}
```
- Any new fields you add must have json tags for the fields to be serialized.
- Finally, we add the Go types to the API group. 
- This allows us to add the types in this API group to any [Scheme](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Scheme)
```go
func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```




----
**Designing an API**
- In Kubernetes, we have a few rules for how we design APIs 
	- Namely, all serialized fields _must_ be 
		- `camelCase`, so we use JSON struct tags to specify this
		- We can also use the `omitempty` struct tag to mark that a field should be omitted from serialization when empty
- Fields may use most of the primitive types 
- Numbers are the exception: 
	- for API compatibility purposes, we accept three forms of numbers: 
		- `int32` and `int64` for integers, and 
		- `resource.Quantity` for decimals
			- Quantities are a special notation for decimal numbers 
				- These have an explicitly fixed representation that makes them more portable across machines
			- These r used for specifying resources requests and limits on pods in Kubernetes
			- They conceptually work similar to floating point numbers: 
				- they have a significant, 
					- base, and 
					- exponent
			- Their serializable and human readable format uses whole numbers and suffixes 
				- to specify values much the way we describe computer storage
			- For instance, the 
				- value `2m` means `0.002` in decimal notation 
				- `2Ki` means `2048` in decimal, while 
				- `2K` means `2000` in decimal
			- If we want to specify fractions, we switch to a suffix that lets us use a whole number: 
				- `2.5` is `2500m`
			- There are two supported bases: 
				- 10 and 2 _(called decimal and binary, respectively)_
					- Decimal base is indicated with “normal” SI suffixes (e.g. `M` and `K`), while 
					- Binary base is specified in “mebi” notation (e.g. `Mi` and `Ki`)
				- Think [megabytes vs mebibytes](https://en.wikipedia.org/wiki/Binary_prefix)
	- There’s one other special type that we use: `metav1.Time`
		- This functions identically to `time.Time`, 
			- except that it has a fixed, portable serialization format.
```go
package v1  

//Imports

import (
    batchv1 "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.
```
- spec holds _desired state_, so any “inputs” to our controller go here.
- Fundamentally a CronJob needs the following pieces:
	- A schedule _(the **cron** in CronJob)_
	- A template for the Job to run _(the **job** in CronJob)_
- We’ll also want a few extras, which will make our users’ lives easier:
	- A deadline for starting jobs _(if we miss this deadline, we’ll just wait till the next scheduled time)_
	- What to do if multiple jobs would run at once _(do we wait? stop the old one? run both?)_
	- A way to pause the running of a CronJob, in case something’s wrong with it
	- Limits on old job history
- need to have some other way to keep track of whether a job has run
	- We can use at least one old job to do this
- use several markers (`// +comment`) to specify additional metadata. 
	- These will be used by [controller-tools](https://github.com/kubernetes-sigs/controller-tools) when generating our CRD manifest
	- controller-tools also use GoDoc to form descriptions for the fields
```go
// CronJobSpec defines the desired state of CronJob.
type CronJobSpec struct {
    // +kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    Schedule string `json:"schedule"`

    // +kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1.JobTemplateSpec `json:"jobTemplate"`

    // +kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    // +kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```
- define a custom type to hold our concurrency policy
	- It’s actually just a string under the hood, 
		- but the type gives extra documentation, and 
		- allows us to attach validation on the type instead of the field, 
			- making the validation more easily reusable.
```go
// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    // AllowConcurrent allows CronJobs to run concurrently.
    AllowConcurrent ConcurrencyPolicy = "Allow"

    // ForbidConcurrent forbids concurrent runs, skipping next run if previous
    // hasn't finished yet.
    ForbidConcurrent ConcurrencyPolicy = "Forbid"

    // ReplaceConcurrent cancels currently running job and replaces it with a new one.
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```
- design our status, which holds observed state 
	- It contains any information we want users or other controllers to be able to easily obtain
- keep a list of 
	- actively running jobs, as well as the 
	- last time that we successfully ran our job
- use `metav1.Time` instead of `time.Time` to get the stable serialization
```go
// CronJobStatus defines the observed state of CronJob.
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // A list of pointers to currently running jobs.
    // +optional
    Active []corev1.ObjectReference `json:"active,omitempty"`

    // Information when was the last time the job was successfully scheduled.
    // +optional
    LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```
- rest of the boilerplate 
	- we don’t need to change this, except to mark that 
	- we want a status subresource, 
		- so that we behave like built-in kubernetes types.
```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API.
type CronJob struct {
// Root Object Definitions
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronJobList contains a list of CronJob.
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}

func init() {
    SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```
- rest of the files in the [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) directory, has two additional files beyond `cronjob_types.go`: 
	- `groupversion_info.go` and 
	- `zz_generated.deepcopy.go`
- Neither of these files ever needs to be edited _(the former stays the same and the latter is autogenerated)_
## [`groupversion_info.go`](https://book.kubebuilder.io/cronjob-tutorial/other-api-files#groupversion_infogo)
- `groupversion_info.go` contains common metadata about the group-version: [project/api/v1/groupversion_info.go](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1/groupversion_info.go)
- First, we have some _package-level_ markers that denote that 
	- there are Kubernetes objects in this package, and that 
	- this package represents the group `batch.tutorial.kubebuilder.io`
- The `object` generator makes use of the former, while 
	- the latter is used by the CRD generator to generate the right metadata for the CRDs it creates from this package
```go
// Package v1 contains API Schema definitions for the batch v1 API group.
// +kubebuilder:object:generate=true
// +groupName=batch.tutorial.kubebuilder.io
package v1

import (
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/scheme"
)
```
- Then, we have the commonly useful variables that help us set up our Scheme
- Since we need to use all the types in this package in our controller, 
	- it’s helpful _(and the convention)_ to have a convenient method to add all the types to some other `Scheme`
		- SchemeBuilder makes this easy for us.
```go
var (
    // GroupVersion is group version used to register these objects.
    GroupVersion = schema.GroupVersion{Group: "batch.tutorial.kubebuilder.io", Version: "v1"}

    // SchemeBuilder is used to add go types to the GroupVersionKind scheme.
    SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

    // AddToScheme adds the types in this group-version to the given scheme.
    AddToScheme = SchemeBuilder.AddToScheme
)
```
## [`zz_generated.deepcopy.go`](https://book.kubebuilder.io/cronjob-tutorial/other-api-files#zz_generateddeepcopygo)
- `zz_generated.deepcopy.go` contains the autogenerated implementation of the aforementioned `runtime.Object` interface, 
	- This marks all of our root types as representing Kinds
- The core of the `runtime.Object` interface is a deep-copy method, `DeepCopyObject`
- The `object` generator in controller-tools also generates two other handy methods for each root type and all its sub-types: 
	- `DeepCopy` and 
	- `DeepCopyInto`

- **Root Object Definitions**
	- Now that we have an API, we’ll need to write a controller to actually implement the functionality.




---
**Controllers**
- core of k8s & any operator
	- Each controller focuses on one _root_ Kind, but may interact with other Kinds
- its job is to ensure that, for any given object, 
	- actual state of the world 
		- _(both the cluster state, and potentially external state like running containers for Kubelet or loadbalancers for a cloud provider)_ matches 
	- desired state in the object
	- this process is called _reconciling_
		- logic that implements reconciling for a specific kind is called a [_Reconciler_](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile?tab=doc)
		- Reconciler takes the name of an object, and returns whether or not we need to try again 
			- _(e.g. in case of errors or periodic controllers, like the HorizontalPodAutoscaler)_

**Basic structure of a Reconciler** 
- Start with some standard imports. 
	- we need
		- core controller-runtime library, as well as
		- client package, and 
		- package for our API types.
```go
package controllers

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```
- Next, `kubebuilder` has scaffolded a basic reconciler struct for us. 
- Pretty much every reconciler needs to 
	- log, and  
	- fetch objects, _so these are added out of the box_
```go
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```
- Most controllers eventually end up running on the cluster, 
	- so they need RBAC permissions, which we specify using controller-tools [RBAC markers](https://book.kubebuilder.io/reference/markers/rbac). 
	- These are the bare minimum permissions needed to run
```go
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
```
- The `ClusterRole` manifest at `config/rbac/role.yaml` is generated from the above markers via controller-gen with the following command:
```bash
make manifests
```
- `Reconcile` actually performs the reconciling for a single named object. 
```go
type Request struct {
	// NamespacedName is the name and namespace of the object to reconcile.
	[types][NamespacedName]
}
```
- Our [Request](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile?tab=doc#Request) just has a name, but we can use the client to fetch that object from the cache.
	- Request contains the information necessary to reconcile a Kubernetes object. 
		- This includes the information to uniquely identify the object - its Name and Namespace. 
		- It does NOT contain information about any specific Event or the object contents itself.
- We return an empty result and no error
	- This indicates to controller-runtime that 
		- we’ve successfully reconciled this object and 
		- don’t need to try again until there’s some changes
- Most controllers need a 
	- logging handle and a 
	- context, _so we set them up here_





----
**Context**
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





---
**Implementing a Controller**
- https://book.kubebuilder.io/cronjob-tutorial/controller-implementation








---
**Deploying/Running a Controller**
- If any changes are made to the API definitions, then before proceeding, 
	- generate the manifests like CRs or CRDs with
```bash
make manifests
```
- To test out the controller, 
	- we can run it locally against the cluster
- Before we do so, though, we’ll need to install our CRDs, as per the [quick start](https://book.kubebuilder.io/quick-start)
	- This will automatically update the YAML manifests using controller-tools, if needed:
```bash
make install
```
- running the controller against our cluster will use whatever credentials that we connect to the cluster with, so we don’t need to worry about RBAC just yet

# [Running webhooks locally](https://book.kubebuilder.io/cronjob-tutorial/running#running-webhooks-locally)
- to run the webhooks locally, 
	- generate certificates for serving the webhooks, and 
	- place them in the right directory (`/tmp/k8s-webhook-server/serving-certs/tls.{crt,key}`, by default)
- If you’re not running a local API server, 
	- you’ll also need to figure out how to proxy traffic from the remote cluster to your local webhook server. 
	- For this reason, we generally recommend disabling webhooks when doing your local code-run-test cycle
- In a separate terminal, run
```bash
export ENABLE_WEBHOOKS=false make run
```
- see logs from the controller about starting up, but it won’t do anything just yet
- At this point, we need a CronJob to test with
	- Let’s write a sample to `config/samples/batch_v1_cronjob.yaml`, and use that:
```yaml
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: CronJob
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: cronjob-sample
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Allow # explicitly specify, but Allow is also default.
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
  
```bash
kubectl create -f config/samples/batch_v1_cronjob.yaml
```
- Look for your cronjob running, and updating status:
```bash
kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml kubectl get job
```
- Now that we know it’s working, we can run it in the cluster
	- Stop the `make run` invocation, and run
```bash
make docker-build docker-push IMG=<some-registry>/<project-name>:tag make deploy IMG=<some-registry>/<project-name>:tag
```
# [Registry Permission](https://book.kubebuilder.io/cronjob-tutorial/running#registry-permission)
- Consider incorporating Kind into your workflow for a faster, more efficient local development and CI experience
	- if you’re using a Kind cluster, there’s no need to push your image to a remote container registry
	- You can directly load your local image into your specified Kind cluster:
```bash
kind load docker-image <your-image-name>:tag --name <your-kind-cluster-name>
```
- To know more, see: [Using Kind For Development Purposes and CI](https://book.kubebuilder.io/reference/kind)

# [RBAC errors](https://book.kubebuilder.io/cronjob-tutorial/running#rbac-errors)
- If you encounter RBAC errors, 
	- you may need to grant yourself cluster-admin privileges or be logged in as admin.
- See [Prerequisites for using Kubernetes RBAC on GKE cluster v1.11.x and older](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#iam-rolebinding-bootstrap) which may be your case




----
**Writing Controller Tests**
- Testing Kubernetes controllers is a big subject, and the 
	- boilerplate testing files generated by kubebuilder are fairly minimal
- The basic approach is that, in generated `suite_test.go` file, you will 
	- use envtest to create a local Kubernetes API server, 
	- instantiate and run your controllers, and then 
	- write additional `*_test.go` files to test it using [Ginkgo](http://onsi.github.io/ginkgo).
- If you want to tinker with how your envtest cluster is configured, 
	- see section [Configuring envtest for integration tests](https://book.kubebuilder.io/reference/envtest) as well as the [`envtest docs`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest?tab=doc).

## [Test Environment Setup](https://book.kubebuilder.io/cronjob-tutorial/writing-tests#test-environment-setup)
- [../../cronjob-tutorial/testdata/project/internal/controller/suite_test.go](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/internal/controller/suite_test.go)
- When we created the CronJob API with `kubebuilder create api` in a [previous chapter](https://book.kubebuilder.io/cronjob-tutorial/new-api), 
	- Kubebuilder already did some test work for you
	- Kubebuilder scaffolded a `internal/controller/suite_test.go` file 
		- that does the bare bones of setting up a test environment.
- First, it will contain the necessary imports
```go
package controller

import (
    "context"
    "os"
    "path/filepath"
    "testing"

    ctrl "sigs.k8s.io/controller-runtime"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"

    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
    // +kubebuilder:scaffold:imports
)

// These tests use Ginkgo (BDD-style Go testing framework). Refer to
// http://onsi.github.io/ginkgo/ to learn more about Ginkgo.
```
- Now, let’s go through the code generated.
```go
var (
    ctx       context.Context
    cancel    context.CancelFunc
    testEnv   *envtest.Environment
    cfg       *rest.Config
    k8sClient client.Client // You'll be using this client in your tests.
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)

    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

    ctx, cancel = context.WithCancel(context.TODO())

    var err error
```
- The CronJob Kind is added to the runtime scheme used by the test environment
	- This ensures that the CronJob API is registered with the scheme, 
		- allowing the test controller to recognize and interact with CronJob resources
```go
    err = batchv1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())
```
- After the schemas, you will see the following marker
	- This marker is what allows new schemas to be added here automatically 
		- when a new API is added to the project
```go
    // +kubebuilder:scaffold:scheme
```
- The envtest environment is configured to load Custom Resource Definitions (CRDs) from the specified directory
	- This setup enables the test environment to recognize and interact with the custom resources defined by these CRDs
```go
    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    // Retrieve the first found binary directory to allow running tests from IDEs
    if getFirstFoundEnvTestBinaryDir() != "" {
        testEnv.BinaryAssetsDirectory = getFirstFoundEnvTestBinaryDir()
    }
```
- Then, we start the envtest cluster
```go
    // cfg is defined in this file globally.
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())
```
- A client is created for our test CRUD operations
```go
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())
```
- One thing that this autogenerated file is missing, however, is 
	- a way to actually start your controller 
- The code above will set up a client for interacting with your custom Kind, 
	- but will not be able to test your controller behavior
- to test your custom controller logic, 
	- you’ll need to add some familiar-looking manager logic to your BeforeSuite() function, 
	- so you can register your custom controller to run on this test cluster

- the code below runs your controller with nearly identical logic to your CronJob project’s main.go! 
- The only difference is that the manager is started in a separate goroutine 
	- so it does not block the cleanup of envtest when you’re done running your tests

- Note that we set up both a “live” k8s client and a separate client from the manager
	- This is because when making assertions in tests, 
		- you generally want to assert against the live state of the API server
- If you use the client from the manager (`k8sManager.GetClient`), 
	- you’d end up asserting against the contents of the cache instead, 
		- which is slower and can introduce flakiness into your tests
	- We could use the manager’s `APIReader` to accomplish the same thing, 
		- but that would leave us with two clients in our test assertions and 
		- setup _(one for reading, one for writing)_, and 
		- it’d be easy to make mistakes

- Note that we keep the reconciler running against the manager’s cache client, though
	- we want our controller to behave as it would in production, and 
	- we use features of the cache (like indices) in our controller 
		- which aren’t available when talking directly to the API server
```go
    k8sManager, err := ctrl.NewManager(cfg, ctrl.Options{
        Scheme: scheme.Scheme,
    })
    Expect(err).ToNot(HaveOccurred())

    err = (&CronJobReconciler{
        Client: k8sManager.GetClient(),
        Scheme: k8sManager.GetScheme(),
    }).SetupWithManager(k8sManager)
    Expect(err).ToNot(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        err = k8sManager.Start(ctx)
        Expect(err).ToNot(HaveOccurred(), "failed to run manager")
    }()
})
```
- Kubebuilder also generates boilerplate functions for cleaning up envtest and 
	- actually running your test files in your controllers/ directory
- You won’t need to touch these.
```go
var _ = AfterSuite(func() {
    By("tearing down the test environment")
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```
- Now that you have your controller running on a test cluster and 
	- a client ready to perform operations on your CronJob, 
		- we can start writing integration tests!
```go
// getFirstFoundEnvTestBinaryDir locates the first binary in the specified path.
// ENVTEST-based tests depend on specific binaries, usually located in paths set by
// controller-runtime. When running tests directly (e.g., via an IDE) without using
// Makefile targets, the 'BinaryAssetsDirectory' must be explicitly configured.
//
// This function streamlines the process by finding the required binaries, similar to
// setting the 'KUBEBUILDER_ASSETS' environment variable. To ensure the binaries are
// properly set up, run 'make setup-envtest' beforehand.
func getFirstFoundEnvTestBinaryDir() string {
    basePath := filepath.Join("..", "..", "bin", "k8s")
    entries, err := os.ReadDir(basePath)
    if err != nil {
        logf.Log.Error(err, "Failed to read directory", "path", basePath)
        return ""
    }
    for _, entry := range entries {
        if entry.IsDir() {
            return filepath.Join(basePath, entry.Name())
        }
    }
    return ""
}
```

## [Testing your Controller’s Behavior](https://book.kubebuilder.io/cronjob-tutorial/writing-tests#testing-your-controllers-behavior)
[../../cronjob-tutorial/testdata/project/internal/controller/cronjob_controller_test.go](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/internal/controller/cronjob_controller_test.go)

- Ideally, we should have one `<kind>_controller_test.go` for each controller scaffolded and
	- called in the suite_test.go
- So, let’s write our example test for the CronJob controller _(cronjob_controller_test.go)_

- start with the necessary imports
	- also define some utility variables
```go
package controller

import (
    "context"
    "reflect"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    batchv1 "k8s.io/api/batch/v1"
    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"

    cronjobv1 "tutorial.kubebuilder.io/project/api/v1"
)
```
- The first step to writing a simple integration test is to 
	- actually create an instance of CronJob you can run tests against
- Note that to create a CronJob, 
	- you’ll need to create a stub CronJob struct 
		- that contains your CronJob’s specifications

- Note that when we create a stub CronJob, 
	- the CronJob also needs stubs of its required downstream objects
- Without the stubbed Job template spec and the Pod template spec below, 
	- the Kubernetes API will not be able to create the CronJob
```go
var _ = Describe("CronJob controller", func() {

    // Define utility constants for object names and testing timeouts/durations and intervals.
    const (
        CronjobName      = "test-cronjob"
        CronjobNamespace = "default"
        JobName          = "test-job"

        timeout  = time.Second * 10
        duration = time.Second * 10
        interval = time.Millisecond * 250
    )

    Context("When updating CronJob Status", func() {
        It("Should increase CronJob Status.Active count when new Jobs are created", func() {
            By("By creating a new CronJob")
            ctx := context.Background()
            cronJob := &cronjobv1.CronJob{
                TypeMeta: metav1.TypeMeta{
                    APIVersion: "batch.tutorial.kubebuilder.io/v1",
                    Kind:       "CronJob",
                },
                ObjectMeta: metav1.ObjectMeta{
                    Name:      CronjobName,
                    Namespace: CronjobNamespace,
                },
                Spec: cronjobv1.CronJobSpec{
                    Schedule: "1 * * * *",
                    JobTemplate: batchv1.JobTemplateSpec{
                        Spec: batchv1.JobSpec{
                            // For simplicity, we only fill out the required fields.
                            Template: v1.PodTemplateSpec{
                                Spec: v1.PodSpec{
                                    // For simplicity, we only fill out the required fields.
                                    Containers: []v1.Container{
                                        {
                                            Name:  "test-container",
                                            Image: "test-image",
                                        },
                                    },
                                    RestartPolicy: v1.RestartPolicyOnFailure,
                                },
                            },
                        },
                    },
                },
            }
            Expect(k8sClient.Create(ctx, cronJob)).To(Succeed())

        
```
- let’s check that the CronJob’s Spec fields match what we passed in
- because the k8s apiserver may not have finished creating a CronJob after our `Create()` call from earlier, 
	- we will use Gomega’s Eventually() testing function 
		- instead of Expect() 
	- to give the apiserver an opportunity to finish creating our CronJob

- `Eventually()` will repeatedly run the function provided as an argument every interval seconds until 
	- the assertions done by the passed-in `Gomega` succeed, or 
	- the number of attempts * interval period exceed the provided timeout value

- In the examples below, timeout and interval are Go Duration values of our choosing.
```go
            cronjobLookupKey := types.NamespacedName{Name: CronjobName, Namespace: CronjobNamespace}
            createdCronjob := &cronjobv1.CronJob{}

            // We'll need to retry getting this newly created CronJob, given that creation may not immediately happen.
            Eventually(func(g Gomega) {
                g.Expect(k8sClient.Get(ctx, cronjobLookupKey, createdCronjob)).To(Succeed())
            }, timeout, interval).Should(Succeed())
            // Let's make sure our Schedule string value was properly converted/handled.
            Expect(createdCronjob.Spec.Schedule).To(Equal("1 * * * *"))
        
```
- Now that we’ve created a CronJob in our test cluster, 
	- the next step is to write a test that actually tests our CronJob controller’s behavior
- Let’s test the CronJob controller’s logic 
	- responsible for updating CronJob.Status.Active with actively running jobs
- We’ll verify that when a CronJob has a single active downstream Job, 
	- its CronJob.Status.Active field contains a reference to this Job

- First, we should get the test CronJob we created earlier, and 
	- verify that it currently does not have any active jobs
- We use Gomega’s `Consistently()` check here 
	- to ensure that the active job count remains 0 over a duration of time
```go
            By("By checking the CronJob has zero active Jobs")
            Consistently(func(g Gomega) {
                g.Expect(k8sClient.Get(ctx, cronjobLookupKey, createdCronjob)).To(Succeed())
                g.Expect(createdCronjob.Status.Active).To(HaveLen(0))
            }, duration, interval).Should(Succeed())
        
```
- Next, we actually create a stubbed Job that will belong to our CronJob, as well as 
	- its downstream template specs
- We set the Job’s status’s “Active” count to 2 to simulate the Job running two pods, 
	- which means the Job is actively running

- We then take the stubbed Job and set its owner reference to point to our test CronJob
	- This ensures that the test Job belongs to, and 
	- is tracked by, our test CronJob
- Once that’s done, we create our new Job instance
```go
            By("By creating a new Job")
            testJob := &batchv1.Job{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      JobName,
                    Namespace: CronjobNamespace,
                },
                Spec: batchv1.JobSpec{
                    Template: v1.PodTemplateSpec{
                        Spec: v1.PodSpec{
                            // For simplicity, we only fill out the required fields.
                            Containers: []v1.Container{
                                {
                                    Name:  "test-container",
                                    Image: "test-image",
                                },
                            },
                            RestartPolicy: v1.RestartPolicyOnFailure,
                        },
                    },
                },
            }

            // Note that your CronJob’s GroupVersionKind is required to set up this owner reference.
            kind := reflect.TypeOf(cronjobv1.CronJob{}).Name()
            gvk := cronjobv1.GroupVersion.WithKind(kind)

            controllerRef := metav1.NewControllerRef(createdCronjob, gvk)
            testJob.SetOwnerReferences([]metav1.OwnerReference{*controllerRef})
            Expect(k8sClient.Create(ctx, testJob)).To(Succeed())
            // Note that you can not manage the status values while creating the resource.
            // The status field is managed separately to reflect the current state of the resource.
            // Therefore, it should be updated using a PATCH or PUT operation after the resource has been created.
            // Additionally, it is recommended to use StatusConditions to manage the status. For further information see:
            // https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
            testJob.Status.Active = 2
            Expect(k8sClient.Status().Update(ctx, testJob)).To(Succeed())
        
```
- Adding this Job to our test CronJob should trigger our controller’s reconciler logic
- After that, we can write a test that evaluates 
	- whether our controller eventually updates our CronJob’s Status field as expected!
```go
            By("By checking that the CronJob has one active Job")
            Eventually(func(g Gomega) {
                g.Expect(k8sClient.Get(ctx, cronjobLookupKey, createdCronjob)).To(Succeed(), "should GET the CronJob")
                g.Expect(createdCronjob.Status.Active).To(HaveLen(1), "should have exactly one active job")
                g.Expect(createdCronjob.Status.Active[0].Name).To(Equal(JobName), "the wrong job is active")
            }, timeout, interval).Should(Succeed(), "should list our active job %s in the active jobs list in status", JobName)
        })
    })

})
```
- After writing all this code, you can run `go test ./...` in your `controllers/` directory again to run your new test!

- This Status update example above demonstrates a general testing strategy for a custom Kind with downstream objects
- By this point, you hopefully have learned the following methods for testing your controller behavior:
	- Setting up your controller to run on an envtest cluster
	- Writing stubs for creating test objects
	- Isolating changes to an object to test specific controller behavior












---
**Implementing Webhooks**
- If you want to implement [admission webhooks](https://book.kubebuilder.io/reference/admission-webhook) for your CRD, 
	- the only thing you need to do is to implement the 
		- `CustomDefaulter` and (or) the 
		- `CustomValidator` interface

- Kubebuilder takes care of the rest for you, such as
	1. Creating the webhook server
	2. Ensuring the server has been added in the manager
	3. Creating handlers for your webhooks
	4. Registering each handler with a path in your server

- First, let’s scaffold the webhooks for our CRD _(CronJob)_
- run the following command with the
	- `--defaulting` and 
	- `--programmatic-validation` flags _(since our test project will use defaulting and validating webhooks)_
``` bash
kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation
```
- This will scaffold the webhook functions and 
	- register your webhook with the manager in your `main.go` for you
```go
package v1

// Go Imports
import (
    "context"
    "fmt"
    "github.com/robfig/cron"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime/schema"
    validationutils "k8s.io/apimachinery/pkg/util/validation"
    "k8s.io/apimachinery/pkg/util/validation/field"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```
- Next, setup a logger for the webhooks
```go
var cronjoblog = logf.Log.WithName("cronjob-resource")
```
- Then, we set up the webhook with the manager
```go
// SetupCronJobWebhookWithManager registers the webhook for CronJob in the manager.
func SetupCronJobWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).For(&batchv1.CronJob{}).
        WithValidator(&CronJobCustomValidator{}).
        WithDefaulter(&CronJobCustomDefaulter{
            DefaultConcurrencyPolicy:          batchv1.AllowConcurrent,
            DefaultSuspend:                    false,
            DefaultSuccessfulJobsHistoryLimit: 3,
            DefaultFailedJobsHistoryLimit:     1,
        }).
        Complete()
}
```
- use kubebuilder markers to generate webhook manifests
	- This marker is responsible for generating a mutating webhook manifest
- The meaning of each marker can be found [here](https://book.kubebuilder.io/reference/markers/webhook)
```go
// +kubebuilder:webhook:path=/mutate-batch-tutorial-kubebuilder-io-v1-cronjob,mutating=true,failurePolicy=fail,sideEffects=None,groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=create;update,versions=v1,name=mcronjob-v1.kb.io,admissionReviewVersions=v1

// CronJobCustomDefaulter struct is responsible for setting default values on the custom resource of the
// Kind CronJob when those are created or updated.
//
// NOTE: The +kubebuilder:object:generate=false marker prevents controller-gen from generating DeepCopy methods,
// as it is used only for temporary operations and does not need to be deeply copied.
type CronJobCustomDefaulter struct {

    // Default values for various CronJob fields
    DefaultConcurrencyPolicy          batchv1.ConcurrencyPolicy
    DefaultSuspend                    bool
    DefaultSuccessfulJobsHistoryLimit int32
    DefaultFailedJobsHistoryLimit     int32
}

var _ webhook.CustomDefaulter = &CronJobCustomDefaulter{}
```
- use the `webhook.CustomDefaulter`interface to set defaults to our CRD
	- A webhook will automatically be served that calls this defaulting
- The `Default`method is expected to mutate the receiver, setting the defaults
```go
// Default implements webhook.CustomDefaulter so a webhook will be registered for the Kind CronJob.
func (d *CronJobCustomDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    cronjob, ok := obj.(*batchv1.CronJob)

    if !ok {
        return fmt.Errorf("expected an CronJob object but got %T", obj)
    }
    cronjoblog.Info("Defaulting for CronJob", "name", cronjob.GetName())

    // Set default values
    d.applyDefaults(cronjob)
    return nil
}

// applyDefaults applies default values to CronJob fields.
func (d *CronJobCustomDefaulter) applyDefaults(cronJob *batchv1.CronJob) {
    if cronJob.Spec.ConcurrencyPolicy == "" {
        cronJob.Spec.ConcurrencyPolicy = d.DefaultConcurrencyPolicy
    }
    if cronJob.Spec.Suspend == nil {
        cronJob.Spec.Suspend = new(bool)
        *cronJob.Spec.Suspend = d.DefaultSuspend
    }
    if cronJob.Spec.SuccessfulJobsHistoryLimit == nil {
        cronJob.Spec.SuccessfulJobsHistoryLimit = new(int32)
        *cronJob.Spec.SuccessfulJobsHistoryLimit = d.DefaultSuccessfulJobsHistoryLimit
    }
    if cronJob.Spec.FailedJobsHistoryLimit == nil {
        cronJob.Spec.FailedJobsHistoryLimit = new(int32)
        *cronJob.Spec.FailedJobsHistoryLimit = d.DefaultFailedJobsHistoryLimit
    }
}
```
- We can validate our CRD beyond what’s possible with declarative validation
	- Generally, declarative validation should be sufficient, 
		- but sometimes more advanced use cases call for complex validation
		- For instance, use this to validate a well-formed cron schedule without making up a long regular expression
- If `webhook.CustomValidator` interface is implemented, 
	- a webhook will automatically be served that calls the validation
- The `ValidateCreate`, `ValidateUpdate` and `ValidateDelete` methods are expected to 
	- validate its receiver upon creation, update and deletion respectively
- separate out ValidateCreate from ValidateUpdate to allow behavior like making certain fields immutable, 
	- so that they can only be set on creation
- ValidateDelete is also separated from ValidateUpdate to allow different validation behavior on deletion 
	- Here, however, we just use the same shared validation for `ValidateCreate` and `ValidateUpdate`
	- And we do nothing in `ValidateDelete`, since we don’t need to validate anything on deletion

- This marker is responsible for generating a validation webhook manifest.
```go
// +kubebuilder:webhook:path=/validate-batch-tutorial-kubebuilder-io-v1-cronjob,mutating=false,failurePolicy=fail,sideEffects=None,groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=create;update,versions=v1,name=vcronjob-v1.kb.io,admissionReviewVersions=v1

// CronJobCustomValidator struct is responsible for validating the CronJob resource
// when it is created, updated, or deleted.
//
// NOTE: The +kubebuilder:object:generate=false marker prevents controller-gen from generating DeepCopy methods,
// as this struct is used only for temporary operations and does not need to be deeply copied.
type CronJobCustomValidator struct {
    // TODO(user): Add more fields as needed for validation
}

var _ webhook.CustomValidator = &CronJobCustomValidator{}

// ValidateCreate implements webhook.CustomValidator so a webhook will be registered for the type CronJob.
func (v *CronJobCustomValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    cronjob, ok := obj.(*batchv1.CronJob)
    if !ok {
        return nil, fmt.Errorf("expected a CronJob object but got %T", obj)
    }
    cronjoblog.Info("Validation for CronJob upon creation", "name", cronjob.GetName())

    return nil, validateCronJob(cronjob)
}

// ValidateUpdate implements webhook.CustomValidator so a webhook will be registered for the type CronJob.
func (v *CronJobCustomValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
    cronjob, ok := newObj.(*batchv1.CronJob)
    if !ok {
        return nil, fmt.Errorf("expected a CronJob object for the newObj but got %T", newObj)
    }
    cronjoblog.Info("Validation for CronJob upon update", "name", cronjob.GetName())

    return nil, validateCronJob(cronjob)
}

// ValidateDelete implements webhook.CustomValidator so a webhook will be registered for the type CronJob.
func (v *CronJobCustomValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    cronjob, ok := obj.(*batchv1.CronJob)
    if !ok {
        return nil, fmt.Errorf("expected a CronJob object but got %T", obj)
    }
    cronjoblog.Info("Validation for CronJob upon deletion", "name", cronjob.GetName())

    // TODO(user): fill in your validation logic upon object deletion.

    return nil, nil
}
```
- validate the name and the spec of the CronJob
```go
// validateCronJob validates the fields of a CronJob object.
func validateCronJob(cronjob *batchv1.CronJob) error {
    var allErrs field.ErrorList
    if err := validateCronJobName(cronjob); err != nil {
        allErrs = append(allErrs, err)
    }
    if err := validateCronJobSpec(cronjob); err != nil {
        allErrs = append(allErrs, err)
    }
    if len(allErrs) == 0 {
        return nil
    }

    return apierrors.NewInvalid(
        schema.GroupKind{Group: "batch.tutorial.kubebuilder.io", Kind: "CronJob"},
        cronjob.Name, allErrs)
}
```
- Some fields are declaratively validated by OpenAPI schema. 
	- kubebuilder validation markers (prefixed with `// +kubebuilder:validation`) in the [Designing an API](https://book.kubebuilder.io/cronjob-tutorial/api-design) section
	- all of the kubebuilder supported markers for declaring validation r displayed by running `controller-gen crd -w`, or [here](https://book.kubebuilder.io/reference/markers/crd-validation).
```go
func validateCronJobSpec(cronjob *batchv1.CronJob) *field.Error {
    // The field helpers from the kubernetes API machinery help us return nicely
    // structured validation errors.
    return validateScheduleFormat(
        cronjob.Spec.Schedule,
        field.NewPath("spec").Child("schedule"))
}
```
- validate if the [cron](https://en.wikipedia.org/wiki/Cron) schedule is well-formatted
```go
func validateScheduleFormat(schedule string, fldPath *field.Path) *field.Error {
    if _, err := cron.ParseStandard(schedule); err != nil {
        return field.Invalid(fldPath, schedule, err.Error())
    }
    return nil
}
```
- Validate object name
- Validating the length of a string field can be done declaratively by the validation schema
	- But the `ObjectMeta.Name` field is defined in a shared package under the apimachinery repo, 
		- so we can’t declaratively validate it using the validation schema
```go
func validateCronJobName(cronjob *batchv1.CronJob) *field.Error {
    if len(cronjob.ObjectMeta.Name) > validationutils.DNS1035LabelMaxLength-11 {
        // The job name length is 63 characters like all Kubernetes objects
        // (which must fit in a DNS subdomain). The cronjob controller appends
        // a 11-character suffix to the cronjob (`-$TIMESTAMP`) when creating
        // a job. The job name length limit is 63 characters. Therefore cronjob
        // names must have length <= 63-11=52. If we don't validate this here,
        // then job creation will fail later.
        return field.Invalid(field.NewPath("metadata").Child("name"), cronjob.ObjectMeta.Name, "must be no more than 52 characters")
    }
    return nil
}
```



---
