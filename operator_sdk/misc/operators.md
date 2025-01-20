**Operators** 
- These r go-lang programs, written for k8s, that r considered as client to API svr tht basically runs ctrller to reconcile state of custom stateful apps, with the help of CRs (Custom Resources)
- Operator is built on top of kubeBuilder, so we can leverage functions/features of kubeBuilder in operator
	- Like for Start n End, we can specify range of values and its validation as well; n then definition will b generated based on those ranges n validations
	- We can do this in the form of kubeBuilder annotations like
```Go
// +kubebuilder:validation:Maximum=23
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Required
```
- Refer to KubeBuilder official doc to learn more about such validations n other features tht can be leveraged in operators
- 
- We create CRs
	- Based on definition provided in CRs; operator takes decision of maintaining state of k8s app
- in-built ctrllers
	- deployment / rs ctrller tht reconcile state
- custom operator to scale up/down resources, based on time specified

- initialize project
	- This creates a project for us, wid some scaffolding
	- From this, we create the types & ctrller loop
	- After project is initialized, we see folder structure wid files created
```
operator-sdk init --plugins go/v3 --domain cisco.jasjesin --owner "Jas Singh" --repo github.com/jasjesin/scaler-operator

go get sigs.k8s.io/controller-runtime@v0.12.2
go mod tidy
operator-sdk create api
```

- Create API resource
	- API is the go-type or the object-type that we need to create
```
operator-sdk create api --kind Scaler --group api --version v1alpha1

go mod tidy
make generate
make manifests
```
- This asks to create resource _(CustomResource, i.e., CR)_ and controller _(basically the API)_
	- This creates kustomize manifests and scaffolds in the following folder structure
		- `api/v1alpha1/scaler_types.go`
			- API is the place whr we define structure of our Custom Resource n how it should look like
		- `controllers/scaler_controller.go`
			- Controller is the place whr we define business logic like how we want to scale up/down, via reconcile logic

- Examples of few manifests tht r generated:
	- Location: `config/crd/patches`
- 1st build manifests by executing `make manifests`
	- This will create manifest file for CRD as well as for some sample custom resources _(CRs)_
- Now, check, above cmd creates `bases`folder and manifest for CRD in it
	- This is the definition for how our CR should b structured
- Under config/samples, we can find a sample CR that would have been generated, based on above CRD
	- Specify start, end n count of replicas
	- also specify which deployment needs to be scaled up/down
		- form a structure for this such that this is an array of deployments n its associated ns, that may need to be controlled
```yaml
#api_v1alpha1_scaler.yaml
apiVersion: api.cisco.jasjesin/v1alpha1
kind: Scaler
metadata:
	name: scaler-sample
spec:
	start: 5           # Times are in UTC TZ
	end: 10            # Times are in UTC TZ
	replicas: 5
	deployments:
	- name: nginx
	  ns: default
	- name: xyz
	  ns: jas-poc
```

- Now, to have a structure like above, we need to define it in CRD
	- For this, go to `scaler_types.go`
	- Look for `type Scaler struct`
		- inside this, look for `Spec ScalerSpec` 
			- Now get into `ScalerSpec` 
				- This is whr we need to define the structure to start accepting requests, specified in the spec of CR
```Go 
//scaler_types.go
---
const (
	// The following two constant values can be used in controller code
	SUCCESS = "Success"
	FAILED = "Failed"
)

type Scaler struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec ScalerSpec     `json:"spec,omitempty"`
	Status ScalerStatus `json:"spec,omitempty"`
}

type ScalerStatus {    // use this to display status of deployment replica update as success or failed
	Status string `json:"status,omitempty"` // create constants for value of this as success or failed
}

type ScalerSpec struct {
	Foo string `json:"foo,omitempty"` // this is existing, as created wid make manifest. Remove this and add the following ones
	
// +kubebuilder:validation:Maximum=23
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Required
	Start int `json:"start"`

// +kubebuilder:validation:Maximum=23
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Required
	End int `json:"end"`

// +kubebuilder:validation:Required
	Replicas int32 `json:"replicas"`
	Deployments []NamespacedName `json:"deployments"`	
}

typed NamespacedName struct {
	Name string `json:"name"`
	Ns string `json:"ns"`
}
```
- with this, we hv defined the schema of how a CR should look like.
- Now validate, if CR gets generated correctly, after defining these new variables, by executing `make manifests` again
	- This will generate CR by referencing updated content of CRD
	- Now we will hv something like properties under metadata like >>
```yaml 
#api.cisco.jasjesin.yaml
---
metadata:
	type: string
spec:
	description:
	properties:
		deployments:
			items:
				properties:
					name:
						type: string
					ns:
						type: string
				required:
				- name
				- ns
				type: object
			type: array
		end:
			maximum: 23
			minimum: 0
			type: integer
		replicas: 
			type: integer
		start:
			maximum: 23
			minimum: 0
			type: integer
	required:
	- deployments
	- end
	- replicas
	- start
	type: object
```

- Now, need to write the reconcile logic in controller, to scale up/down replicas; under Reconcile function
```Go
//scaler_controller.go
func (r *ScalerReconciler) Reconcile (ctx context.Context) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here

	log := logger.WithValues("Request.Namespace", req.Ns, "Request.Name", req.Name)
	log.info("Reconcile invoked")

	scaler := &apiv1alpha1.Scaler{}

	err := r.Get(ctx, req.NamespacedName, Scaler)             // to get instance of scaler
	if err != Nil {
		return ctrl.Result{}, nil
	}

	// if no error, then fetch start, end time n count of replicas
	// validate if start n end time r within defined range to scale replicas up/down
	startTime := Scaler.Spec.Start
	endTime := Scaler.Spec.End
	replicaCount := Scaler.Spec.Replicas

	// compute current time n the compare it wid desired time range
	currentHour := time.Now().UTC.Hour()

	log.Info(fmt.Sprintf("Current Time: %d", currentHour))

	if currentHour >= startTime && currentHour <= endTime {
		// Since NamespacedName is an array of deployments, we can loop through it, to fetch name n ns to update replica count
		log.Info(fmt.Sprintf("Condition met")) // debugger sttmts
// The following code is moved into separate function to make code look more clean
/*
		for _, deploy := range Scaler.spec.deployments {
			deployment := &v1.Deployment{}          // helps provide all deployment details
			err := r.Get(ctx, types.NamespacedName{
				Ns: deploy.Ns,
				Name: deploy.Name,
			},
				deployment,
			)

			log.Info(fmt.Sprintf("Deployment: %v", deployment)) // debugger sttmts

			if err != nil {
				return ctrl.Result{}, nil
			}
			
			if deployment.Spec.Replicas != &replicas { // gives mismatch error, set replicas type to int32 and run `make manifests` as this is a change in schema for type
				log.Info(fmt.Sprintf("Replicas to be updated")) // debugger sttmts

				deployment.Spec.Replicas = &replicas

				err := r.update(ctx, deployment)
				if err != nil {
					return ctrl.Result{}, nil
				}
			}

			}
		}
*/
		if err = scaleDeployment(scaler, r, ctx, int32(scaler.Spec.Replicas)); err != Nil {
			return ctrl.Result{}, err
		}
	}
	
	return ctrl.Result{ReQueueAfter: time.Duration(30 * time.Second)}, nil  // Pass time for how frequently this controller loop to reconcile code should be invoked. 
																			// If we dont pass any value here, this reconciler loop wont b called again
}
```

- Ideal way is to write dedicated function for actual work, i.e., for scale up/down
```Go
func scaleDeployment(scaler *apiv1alpha1.Scaler, r *ScalerReconciler, ctx context.Context, replicas int32) error {
	for _, deploy := range scaler.Spec.Deployments {
		dep := &v1.Deployment{}
		err := r.Get(ctx, types.NamespacedName{
			Namespace: deploy.Namespace,
			Name:      deploy.Name,
		}, dep)
		if err != nil {
			return err
		}

		if dep.Spec.Replicas != &replicas {
			dep.Spec.Replicas = &replicas
			err := r.Update(ctx, dep)
			if err != nil {
				scaler.Status.Status = apiv1alpha1.FAILED
				return err
			}
			scaler.Status.Status = apiv1alpha1.SUCCESS
			err = r.Status().Update(ctx, Scaler)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

- Next create cluster using kind
	- `kind create cluster`
- create deployment to be used for testing
	- `k create deploy nginx --image=nginx`
- create CRD
	- `k apply config/crd/bases/api_v1alpha1_scaler.yaml`
	- `k get crd` to validate successful creation

- start operator locally
	- `make run`
- deploy CR
	- `k apply config/samples/api_v1alpha1_samples.yaml`
- Validate changes
	- `k get pod -w # in separate terminal window`


- For more changes in CR, for testing purpose, execute *(maybe needed to adjust start & end time based on current UTC time)*
```bash
k delete config/samples/api_v1alpha1_samples.yaml
# Restart operator process
# - Ctrl+C process running wid make run, to kill existing process
make run # execute again to rekick new operator process
k apply config/samples/api_v1alpha1_samples.yaml
k get pod -w # in separate terminal window
```


--------------

Every set of controllers needs a [_Scheme_](https://book.kubebuilder.io/cronjob-tutorial/gvks#err-but-whats-that-scheme-thing), which provides mappings between Kinds and their corresponding Go types
instantiate a [_manager_](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager?tab=doc#Manager), which keeps track of running all of our controllers, as well as setting up shared caches and clients to the API server (notice we tell the manager about our Scheme).
We run our manager, which in turn runs all of our controllers and webhooks. The manager is set up to run until it receives a graceful shutdown signal. This way, when we’re running on Kubernetes, we behave nicely with graceful pod termination
`Manager` can restrict the namespace that all controllers will watch for resources by
```Go
mgr, err := ctrl.NewManager(cfg, manager.Options{Namespace: namespace})
```
By default this will be empty string which means watch all namespaces

- ns-scoped vs cluster-scoped operator
	- By default, operator-sdk scaffolds a cluster-scoped operator

When a [Manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) instance is created in the `main.go` file, the Namespaces are set via [Cache Config](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache#Config) as described below
## Manager watching options [](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/#manager-watching-options)

### Watching resources in all Namespaces (default)[](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/#watching-resources-in-all-namespaces-default)

A [Manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) is initialized with no Cache option specified, or with a Cache.DefaultNamespaces of `Namespace: ""` will watch all Namespaces:

```go
...
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:             scheme,
    MetricsBindAddress: metricsAddr,
    Port:               9443,
    LeaderElection:     enableLeaderElection,
    LeaderElectionID:   "f1c5ece8.example.com",
})
...
```

### Watching resources in specific Namespaces[](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/#watching-resources-in-specific-namespaces)

To restrict the scope of the [Manager’s](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) cache to a specific Namespace, set `Cache.DefaultNamespaces` field in [Options](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Options):

```go
...
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:             scheme,
    MetricsBindAddress: metricsAddr,
    Port:               9443,
    LeaderElection:     enableLeaderElection,
    LeaderElectionID:   "f1c5ece8.example.com",
    Cache: cache.Options{
      DefaultNamespaces: map[string]cache.Config{"operator-namespace": cache.Config{}},
    },
})
...
```

### Watching resources in a set of Namespaces[](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/#watching-resources-in-a-set-of-namespaces)

It is also possible to use `DefaultNamespaces` to watch and manage resources in a set of Namespaces:

```go
...
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:             scheme,
    MetricsBindAddress: metricsAddr,
    Port:               9443,
    LeaderElection:     enableLeaderElection,
    LeaderElectionID:   "f1c5ece8.example.com",
    Cache: cache.Options{
      DefaultNamespaces: map[string]cache.Config{
        "operator-namespace1": cache.Config{},
        "operator-namespace2": cache.Config{},
      },
    },
})
...
```

In the above example, a CR created in a Namespace not in the set passed to `Cache.DefaultNamespaces` will not be reconciled by its controller because the [Manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) does not manage that Namespace. Further restrictions and qualifications can created on a per-namespace basis by setting fields in the cache.Config object

An operator’s scope defines its [Manager’s](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager) cache’s scope

RBAC permissions are found in the directory `config/rbac/`. The `ClusterRole` in `role.yaml` and `ClusterRoleBinding` in `role_binding.yaml` are used to grant the operator permissions to access and manage its resources.

For changing the operator’s scope only the `role.yaml` and `role_binding.yaml` manifests need to be updated

[`RBAC markers`](https://book.kubebuilder.io/reference/markers/rbac.html) defined in the controller (e.g `controllers/memcached_controller.go`) are used to generate the operator’s [RBAC ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) (e.g `config/rbac/role.yaml`). The default markers don’t specify a `namespace` property and will result in a `ClusterRole`.

Update the [`RBAC markers`](https://book.kubebuilder.io/reference/markers/rbac.html) in `<kind>_controller.go` with `namespace=<namespace>` where the `Role` is to be applied, such as:

```go
//+kubebuilder:rbac:groups=cache.example.com,namespace=memcached-operator-system,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,namespace=memcached-operator-system,resources=memcacheds/status,verbs=get;update;patch
```

Then run `make manifests` to update `config/rbac/role.yaml`. In our example it would look like:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: manager-role
  namespace: memcached-operator-system
```

## Configuring watch namespaces dynamically[](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/#configuring-watch-namespaces-dynamically)

Instead of having any Namespaces hard-coded in the `main.go` file a good practice is to use an environment variable to allow the restrictive configurations. The one suggested here is `WATCH_NAMESPACE`, a comma-separated list of namespaces passed to the manager at deploy time.

After modifying the `*_types.go` file always run the following command to update the generated code for that resource type:

```sh
make generate
```

The above makefile target will invoke the [controller-gen](https://sigs.k8s.io/controller-tools) utility to update the `api/v1alpha1/zz_generated.deepcopy.go` file to ensure our API’s Go type definitions implement the `runtime.Object` interface that all Kind types must implement.

OpenAPI validation defined in a CRD ensures CRs are validated based on a set of declarative rules. All CRDs should have validation.





---------


