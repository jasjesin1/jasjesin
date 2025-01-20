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



