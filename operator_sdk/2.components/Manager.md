
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
