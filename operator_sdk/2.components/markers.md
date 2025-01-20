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

