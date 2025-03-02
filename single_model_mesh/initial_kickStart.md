**Basic Starting Point**: 

- **Documentation**
	- **KServe**
		- [RH-specific Single Model](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.8/html/serving_models/serving-large-models_serving-large-models)
		- [open-src specific KServe](https://github.com/kserve/kserve)
		- [KServe main site](https://kserve.github.io/website/latest/)
	- **KNative**
		- [RH-specific Serverless](https://docs.openshift.com/serverless/1.33/about/about-serverless.html)
		- [open-src specific KNative](https://knative.dev/docs/serving/)

- RH is proficient @ re-packaging open-src things by adding few LOCs _(lines of code)_ and building a proprietary product out of it
- RH's customized version of **KServe** is **Single Model Mesh**
	- KServe is the underlying component that does most of the work under the hood, to serve Large Models
- KServe depends on KNative
	- RH's customized version of **KNative** is **openShift Serverless**
