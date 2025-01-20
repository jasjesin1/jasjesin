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


