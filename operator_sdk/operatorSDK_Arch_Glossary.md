- <span style="color: green;"><b>operator</b></span>: considered as client to API svr, to run ctrller to reconcile state of custom stateful apps, wid help of CR
- <span style="color: red;"><b>Process main.go</b></span>: main entrypoint, that is executed to instantiate mgr
- <font style="color:orange"><b>Manager</b></font>: orchestrates everything for us, by launching pods for ctrllers, clients, caches & webhooks; and monitoring these
- <font style="color:magenta"><b>client</b></font>: help access k8s API objects by taking care of authentication & protocols
- <font style="color:violet"><b>cache</b></font>: store recent requests for objects, by ctrllers n webhooks; used by client for faster txn
- <font style="color:purple"><b>controller</b></font>: contain actual business logic & use predicates to filter events (with event src & handler) for triggering reconcile requests
- <font style="color:gold"><b>event src & handler</b></font>: **event src** watches for k8s API object changes; 
	- **event handler** processes event like queuing up event request for object owner
- <font style="color:teal"><b>predicate</b></font>: filter out which events to consider for reconciliation
- <font style="color:pink"><b>reconciler</b></font>: implements a fn tht takes reconcile request, containing name & ns of object to reconcile; reconciles object, if needed & returns response


![kubeBuilder_Architecture](https://github.com/user-attachments/assets/e66fa8a9-43ee-4718-808e-58a1e26115c0)

**kubeBuilder Architecture**

---
- <font style="color:yellow"><b>scheme</b></font>: provides mappings b/w kinds & associated Go-types, to ctrller
- <font style="color:slateblue"><b>types</b></font>: contain schema of inputs to be provided via CR, for constructing a CRD
- <font style="color:brown"><b>event recorders</b></font>: emit events using mgr
- <font style="color:Tomato"><b>boilerplate</b></font>: foundational template that provides pre-defined set of files wid necessary structure, reusable code patters & configurations as a quick start base for building k8s operators, wid minimal changes; that follows best practices
- <font style="color:dodgerblue"><b>scaffold</b></font>: process & tools that generate initial project structure, including integration boilerplate code
- <font style="color:mediumseagreen"><b>Markers</b></font>: acts as extra metadata, telling [controller-tools](https://github.com/kubernetes-sigs/controller-tools) _(our code and YAML generator)_ extra information

- every functional object needs to contain a spec & a status; spec holds desired state, so any inputs to ctrller go here
- Any new fields you add must have json tags for the fields to be serialized.
- We can also use the `omitempty` struct tag to mark that a field should be omitted from serialization when empty
- design our statue to hold observed state; this contains info tht we want our users or other ctrllers to obtain
- use `metav1.Time` instead of `time.Time` to get the stable serialization

