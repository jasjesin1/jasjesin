**Multi-model**

- uses RHODS AI Operator, that uses open-src **opendatahub-operator** _(this is actually the AI Operator)_ under the hood
	- [opendatahub-operator][https://github.com/opendatahub-io/opendatahub-operator]
	- [opendatahub][https://github.com/opendatahub-io]
	- [modelmesh-runtime-adapter][https://github.com/opendatahub-io/modelmesh-runtime-adapter]
		- [modelmesh-serving][https://github.com/kserve/modelmesh-serving]
- AI Operator deploys models by setting up a Model Svr 1st
	- In that model svr, we deploy a model, by giving S3 path whr model is stored
- Its called **multi-model** cuz tht model svr can run multiple models @ same time
	- **Analogy**: this is similar to JBOSS App Svr, whr we can run multiple web-containers  _(.ear/.war files)_
- Based on path-based routing of model URL / endpoints, Model Svr can redirect internally to specific model
- In this case, we r scaling big application svr / model svr; which is an extra overhead
- Multi-models r ideally suited for smaller ML models
- So, multi-model svrs run constantly/continuously


**Single-model**

- If we have a vry large model, we prefer to run it in multiple pods
- Wenevr it comes to large models like LLMs, 
	- these need to run on big pods that may have GBs of memory; and
	- need to run on multiple pods
- wen we r actually not using LLMs, it should automatically scale it down to zero, i.e., 
	- Model svr shouldn't run that model wen not in use; for cost savings
- Single models do not need to run continuously, can be scaled up n down; 
	- by leveraging KServe, that internally uses KNative _(serverless platform)_ under the hood, for autoscaling

- **Single Model** consists of
	- **KServe**
	- RHOS **Serverless _(KNative)**_
	- RHOS **Svc Mesh _(Istio)**_
