
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


