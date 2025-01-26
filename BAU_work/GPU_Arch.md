**GPU Arch**
- nvidia supports use of GPU on OCP
- OCP is security-focused & hardened k8s platform, developed & supported by RH, for deploying & managing k8s clusters @ scale
- nvidia GPU Operator uses Operator f/w inside OCP to manage full LC of nvidia s/w components, tht r reqd to run GPU-accelerated workloads
	- These s/w components include
		- nvidia drivers to enable CUDA
		- k8s device plugin for GPUs
		- nvidia Container Toolkit
		- automatic node tagging using GPU Feature Discovery (GFD)
			- NodeFeatureDiscovery (NFD) Operator should be installed &
			- `nodefeaturediscovery` instance should be created
		- DCGM-based monitoring

**How GPU Arch can be enabled for openShift?**
![[GPU_Arch_in_openShift.png]]

- **GPUs & BareMetal**
	- Deploy OCP on nvidia-certified BareMetal svc but wid some limitations:
		- ctrl plane nodes can be CPU nodes
		- worker nodes must be GPU nodes, hosting >=1 GPUs of same type/flavor
			- nvidia device plugin for k8s doesn't support mix of diff. GPU models/type/flavor on same node
		- 1 or 3 svrs r reqd
			- Clusters wid 2 svrs r not supported
	- To access containerized GPUs, use 1 of the 2 methods
		- GPU passthrough
		- MIG

- **GPUs & Virtualization**
	- RHOS Virtualization incorporates VMs into containerized workflows within clusters
		- This helps develop & maintain apps tht run on VMs
	- To connect worker nodes to GPUs, use 1 of the 2 methods
		- GPU passthrough to access & use GPU h/w within a VM
		- GPU (vGPU) time-slicing

- **GPU Sharing Methods**
	- RH & nvidia hv developed GPU concurrency & sharing mechanisms to simplify GPU-accelerated computing on OCP cluster, to overcome GPU under-utilization; thereby, reducing deployment cost & maximize GPU utilization
	- GPU Concurrency Mechanisms:
		- ComputeUnifiedDeviceArchitecture (**CUDA**) streams -- within single app
			- CUDA is a platform for parallel computing & programming model, developed by nvidia, for general computing on GPUs
			- Stream is a sequence of operations that execute issue-order on GPU
				- New task doesn't start until preceding task is completed
			- Async processing of operations across diff. streams helps achieve parallel execution of multiple tasks simultaneously, in no prescribed order, leading to improved performance
		- CUDA Multi-Process Svc (**MPS**) -- for multiple apps in multi-user environment
			- allows single GPU  to use multiple CUDA processes to run in parallel on GPU, to eliminate saturation of GPU compute resources
			- MPS also enables concurrent execution or overlapping, of kernel operations & memory copying from diff. processes to enhance utilization
			- MPS is a server-client model where the MPS control daemon runs on the server, and multiple client applications connect to it
			- MPS server manages GPU resources and schedules tasks from different processes, allowing them to share the GPU efficiently
			- MPS can reduce the overhead associated with context switching between different CUDA processes
		- **Time-slicing**
			- Good for older nvidia cards
			- interleaves workloads, tht r scheduled on overloaded GPU, when multiple CUDA apps r running
			- time-slicing can b enabled by defining set of replicas for a GPU, each of which can be independently distributed to a pod, to run workloads on.
				- Example, if a GPU is split into 7 slices, each slice is allocated to an individual pod, making it 7 pods for a single GPU
			- Thr is no memory or fault isolation b/w replicas, i.e., memory is not sliced/shared, stays same full amount of memory, for each slice
			- This is used to multiplex workloads from replicas of same underlying GPU
			- time-slicing can b applied cluster-wide as well as node-specific
				- We do node-specific slicing, by labeling nodes wid node-specific cfg
				- This helps provide diff. GPU flavors in the cluster
		- Multi-instance GPU (**MIG**)
			- Good for MIG-enabled cards on BareMetal
			- MIG is only supported with A30, A100, A100X, A800, AX800, H100, and H800
			- MIG can b used to split GPU compute units & memory, into multiple MIG instances.
				- Each instance represents a standalone GPU device from a system perspective & can b connected to any app, container or a VM on that node
				- s/w tht uses GPU, treats each MIG instance as individual GPU
			- useful wen app doesnt require full power of GPU
			- nvidia Ampere enables to split h/w resources into multiple GPU instances, wid each instance avl to OS, as an independent CUDA-enabled GPU
			- GPU instances r designed to support up to 7 independent CUDA apps, tht can operate completely isolated wid dedicated h/w resources
		- Virtualization wid **vGPU**
			- Best choice for VMs
			- VMs can directly access a single GPU by using nvidia vGPU
			- vGPUs can b created n shared by multiple VMs across the enterprise & can b accessed by other devices
				- This capability helps combine power of GPU performance wid mgmt & security benefits, provided by vGPU, along with 
					- proactive mgmt & monitoring of VM environment, 
					- workload balancing for mixed VDI & compute workloads, 
					- resource sharing across multiple VMs
	- Scenarios & recommendations:
		- VMs wid multiple GPUs, using passthrough & vGPU
			- Use separate VMs
		- BareMetal wid openShift Virtualization & multiple GPUs
			- Use pass-through for hosted VMs &
			- time-slicing for containers



**Key Differences**
**• Scope**:
	**• CUDA Streams**: Operate within a single process, allowing that process to manage multiple streams of execution on the GPU
	**• CUDA MPS**: Operates across multiple processes, enabling them to share the GPU resources concurrently
**• Level of Concurrency**:
	**• CUDA Streams**: Manage concurrency at the task level within a single application
	**• CUDA MPS**: Manages concurrency at the application level, allowing multiple processes to run simultaneously
**• Use Cases**:
	**• CUDA Streams**: Useful for optimizing the performance of a single application by managing internal concurrency
	**• CUDA MPS**: Useful for multi-user or multi-application environments where GPU resources need to be shared


**nvidia GPU features for OCP**
- **Container Toolkit**
	- 
- **AI Enterprise**
	- 
- **GPU Feature DIscovery (GFD)**
	- 
- **GPU Operator wid openShift Virtualization**
	- 
- **GPU Monitoring Dashboard**
	- 