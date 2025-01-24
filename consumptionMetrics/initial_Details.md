camp-config-<dev|np|stage|prd> -- created last, environment-specific key-value pairs; kind of values.yaml
								 -- invokes camp-consumption-metrics
								 
camp-consumption-metrics -- created later, has jinja-templating yamls; 
								 used for creating argoCD app n 
								 kustomize templating, copied from project-api repo
							-- invokes dexter-consumption-metrics
							
dexter-consumption-metrics -- created 1st, has Go code; contains
- cmd/main.go -- all consumption logic is added here
	- using consumption.NewManager n then Manager's process method
- consumption -- contains actual go code; all helper fns defined in separate go files
	- collect.go
	- publish.go
	- manager.go
		- create new Mgr tht connects to Prom to fetch lastPublishedTime from cm


consumptionMetrics is a cronJob tht runs on hourly basis
- wen it runs, it populates cm values wid lastPublishedTime
- so, in case if CJ failed for last 2 hrs, then, it will pick last successful publishTime
- wen it is successful, only then lastPublishedTime is added to Kafka Topic
- once we hv lastPublishedTime, manager.go invokes collect.go, via m.collect method
	- This is whr we 
		- connect to Prom
		- execute queries
		- create a time range for the query
		- building kafka msg, by invoking m.namespaceMetricsToKafkaMessage fn
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
							- external ID, i.e., cluster name & project name
							- consumption will include details abt amount consumed
					- so, basically we construct Kafka msg, serialize it; adding the payload etc; by setting up structs for each block
- Wen the metrics is published, wid final msg published to Kafka; all this msg is structured in collect.go
- manager.go has all the fns which will help in connecting

Look for sample populated contract payload @ wiki.cisco.com/display/IPAS/Consumption+Metrics

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
	























