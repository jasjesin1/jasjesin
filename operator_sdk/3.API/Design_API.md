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

