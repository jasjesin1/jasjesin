**E2E Flow**
-  [Cronjob](https://github.com/ContainerPlatformHub/ccaas-camp-app/blob/develop/deployments/camp-rtp-dev-01/camp-consumption-metrics/bases/camp-consumption-metrics/namespace/camp-consumption-metrics/cronjob.batch/camp-consumption-metrics.yaml) executes hourly, to deploy a container with [image](https://containers.cisco.com/dexter/consumption-metrics) of consumption metrics code
	- This image is built using [Dockerfile](https://github.com/cisco-it-cloud-infrastructure/dexter-consumption-metrics/blob/main/Dockerfile) & uploaded to ECH using [Jenkinsfile](https://github.com/cisco-it-cloud-infrastructure/dexter-consumption-metrics/blob/main/Jenkinsfile) , both specified in [dexter-consumption-metrics](https://github.com/cisco-it-cloud-infrastructure/dexter-consumption-metrics/tree/main)

**Repo Structure**
- [ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app) CaaS Client App Repo contains final artifact of manifests for objects needed to be setup, for app to function in k8s cluster
	|__  [camp-config-$env$](https://github.com/cisco-it-cloud-infrastructure/camp-config-dev) config repo, i.e., [camp-consumption-metrics.yaml](https://github.com/cisco-it-cloud-infrastructure/camp-config-dev/blob/master/camp-rtp-dev-01/components/camp-consumption-metrics.yaml) _(like values.yaml file)_ contains key-value pairs for variables setup in following component repo, invoked here
		|__ [camp-consumption-metrics](https://github.com/cisco-it-cloud-infrastructure/camp-consumption-metrics.git) component repo contains code for app setup in [argoCD](https://camp-rtp-dev-01-k8s-gitops.cisco.com/applications/argocd/camp-consumption-metrics?view=tree&resource=) & jinja2 templating _(kustomizej2)_ overrides for k8s kind-specific keys, used in go-code's kustomize
			|__ [dexter-consumption-metrics](https://github.com/cisco-it-cloud-infrastructure/dexter-consumption-metrics.git) code repo wid go-code, wid base/defaults defined for k8s kind-specific keys in kustomize

**Process for changes**
- Once we fork a feature branch to make changes, raise PR & get it approved by 2 teamies,
- [CAMP Release pipeline](https://eps-idev-jenkins.cisco.com/view/CAMP/job/Camp_Release_Pipeline_camp-rtp-dev-01/), in Jenkins, needs to be executed to invoke [camp-consumption-metrics.yaml](https://github.com/cisco-it-cloud-infrastructure/camp-config-dev/blob/master/camp-rtp-dev-01/components/camp-consumption-metrics.yaml) to sync changes to [ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app) via DeDe Tool, that generates needed manifests
	- Console Output of this pipeline execution specifies PR in [ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app) repo, that needs to be self-reviewed/approved
		- [argoCD](https://camp-rtp-dev-01-k8s-gitops.cisco.com/applications/argocd/camp-consumption-metrics?view=tree&resource=) monitors changes in [ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app)
- Once PR in [ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app) is approved, get in [argoCD](https://camp-rtp-dev-01-k8s-gitops.cisco.com/applications/argocd/camp-consumption-metrics?view=tree&resource=) to monitor/validate reconciliation to live manifest.
	- If it doesn't trigger, try clicking Refresh/Sync manually.


camp-config-<dev|np|stage|prd> -- config repo -- created last, environment-specific key-value pairs; kind of values.yaml
								 -- invokes camp-consumption-metrics
								 
camp-consumption-metrics -- component repo -- created 2nd, has jinja-templating yamls; 
								 used for creating argoCD app n 
								 kustomize templating, copied from project-api repo
							-- invokes dexter-consumption-metrics
							
dexter-consumption-metrics -- code repo -- created 1st, has Go code; contains
- cmd/main.go -- all consumption logic is added here
	- using `consumption.NewManager()` n then Manager's process method
- consumption -- contains actual go code; all helper fns defined in separate go files
	- collect.go -- this is whr r query Prom & structure our msg based on MCMP payload
	- publish.go -- this is wht we hv fns to publish msg to a specific Kafka topic, by using cert files & broker list recvd from Kafka Team
	- manager.go -- this is like a connector/orchestrator to get lastPublished time n then invooke collect.go & publish.go
		- create new Mgr tht connects to Prom to fetch lastPublishedTime from cm

[ccaas-camp-app](https://github.com/ContainerPlatformHub/ccaas-camp-app) -- app repo -- changes to any of the above-specified repos are DeDe-tool'd and needed associated manifests r generated n placed in this final repo, provided by CaaS, for CAMP as client


So, each go file has its own individual fnality
Once evrything is published n successful, we publish a msg to Kafka topic & then job gets successfully executed

consumptionMetrics is a cronJob tht runs on hourly basis
- wen it runs, it populates cm values wid lastPublishedTime
- so, in case if CJ failed for last 2 hrs, then, it will pick last successful publishTime
- wen it is successful, only then lastPublishedTime is added to Kafka Topic
	- lastPublishedTime signifies time when message to Kafka Topic was last published
- once we hv lastPublishedTime, manager.go invokes collect.go, via m.collect method
	- This is whr we 
		- connect to Prom
		- execute queries
		- create a time range for the query, i.e., query range wid start n end time
		- store all 3 metrics for CPU, GPU & storage in namespaceMetrics
		- building kafka msg, by invoking m.namespaceMetricsToKafkaMessage fn n passing namespaceMetrics
			- This will generate 
				- ns Metrics
				- structure the response from Prom (as a JSON object) in a specific Kafka msg, that MCMP expects to recv in
					- basically construct the msg in such a way that MCMP can recognize it, like it needs to hv
						- header
							- consumption start time
							- msgID
							- whts the svc name
							- location
						- payload
							- external ID, i.e., clusterName_projectName
							- consumption will include details abt amount consumed
					- so, basically we construct Kafka msg, serialize it; adding the payload etc; by setting up structs for each block
- Wen the metrics is published, wid final msg published to Kafka; all this msg is structured in collect.go
- manager.go has all the fns which will help in connecting, like
	- getting lastPublishedTime
	- calling 
		- collect.go method
		- publish.go method

- Use jsonlint.com for validating format of json for final msg

**Struct setup:**
```go
KafkaMessage
|__ Header
	|__ ConsumptionPeriodType
	|__ ConsumptionStartTs
	|__ MessageID
	|__ ServiceLocation
	|__ ServiceName
|__ Payload
	|__ ExternalD
	|__ Consumption
		|__ CPU
		|__ GPU
		|__ StorageRWOStandard
		|__ StorageRWXStandard
		|__ StorageRWXBasic
		|__ StorageRWXPremium		
```

- Look for sample populated contract payload @ wiki.cisco.com/display/IPAS/Consumption+Metrics
```yaml
{
	"header": 
	{ 
		"consumption_period_type": "HOUR", 
		"consumption_start_ts": "2019-06-04T17:00:12.631118+00:00", 
		"message_id": "2019-06-04-17:00:1559667612", 
		"service_name": "OPENSHIFT", 
		"service_version": "GA", 
		"service_location": "ALLN" 
	}, 
	"payload": 
	[{ 
		"external_id": "cae-idev-np-alln-felix_dave1", 
		"consumption": 
		{ 
			"standard_block_storage": 0, 
			"ram": 0 
		} 
	},
	{ 
		"external_id": "cae-idev-np-alln-felix_containerss-list", 
		"consumption": 
		{ 
			"standard_block_storage": 0, 
			"ram": 0 
		} 
		}, 
		{ 
			"external_id": "cae-idev-np-alln-felix_containers-openshift", 
			"consumption": 
			{ 
				"standard_block_storage": 0, 
				"ram": 2 
			} 
			},
			{ 
				"external_id": "cae-idev-np-alln-felix_bens-openshift-project", 
				"consumption": 
				{ 
						"standard_block_storage": 0, 
						"ram": 4 
					} 
				}, 
				{ 
						"external_id": "cae-idev-np-alln-felix_beea", 
						"consumption": 
						{ 
							"standard_block_storage": 0, 
							"ram": 0 
						} 
					}] 
				}
```
- kustomize folders for base n overlays r added from dexter-project-api, wid 1 folder created for each kind, like
	- configmap
	- cronjob
	- namespace
	- rbac
		- monitoring-view
		- role
			- role_binding
			- sa_secret
			- service_account
	- So we define each kind/object tht is reqd for consumption metrics to operate
	- These folders r needed for specific job like 
		- cm needs to be created for 
			- job_config, whr all the following r listed  
				- cert file list
				- Kafka Topic
				- Broker list
	
**Dockerfile**
- build this consumption metrics & push this docker image to ECH @ [dexter/consumption-metrics](containers.cisco.com/repository/dexter/consumption-metrics)
	-  each time a PR is created for any change in this code


**Jenkinsfile**
- builds the image
- pushes image
- creates a tag for a specific version n pushes that tag in GIT repo


**camp-consumption-metrics**
- This is the repo we refer in camp-config-dev
-  This one contains
	- GITOps (argoCD) folder
		- we define application kind.
		- chk diff. in wht we specify in argoapplication.yaml.j2 & gitopsconfig.yaml.j2
	- kustomizej2 template folder
		- everything we had put in kustomize folder of dexter-consumption-metrics, we add same files n jinja2-templatize those, for which values come from camp-config-dev repo's camp-consumption-metrics.yaml file
		- then, we add overlays based on each environment
	- jenkinsfile
		- This jenkinsfile seems to just tag the GIT repo. check n validate it
	- kustomization.yaml
		- see reference of dexter-consumption-metrics repo here under resources



- For each change made in dexter-consumption-metrics repo, need to update imageTag in camp-config repo, wid latest generated image tag n then run Jenkins EPS pipeline

 - For consumption metrics, work done only on cmd & consumption folders
 - Once dockerfile was created n image was tested to be built
	 - then, copy kustomize folder from dexter-project-api to dexter-consumption-metrics
		 - then, deleted / updated yamls, basd on what is reqd 
	- same was done for camp-consumption-metrics repo as well, from camp-project-api repo n then updated according to specific reqmnts for consumption metrics



**ci8 pipeline**
- jenkinsfile for building of image for consumption-metrics is present here
	- 






- Arut had followed similar approach for project api to build
	- dexter-project-api
	- camp-project-api

















