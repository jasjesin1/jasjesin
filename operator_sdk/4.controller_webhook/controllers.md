
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

