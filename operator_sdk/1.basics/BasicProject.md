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


