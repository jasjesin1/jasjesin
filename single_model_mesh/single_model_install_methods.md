**Standard Install**

- Single Model (KServe) is dependent on multiple components like KNative & Istio
	- Istio was setup n running before AI Operator was installed
- **Why not Standard Install?**
	- once we specify that we need KServe,
		- AI Operator will go ahead n will install KNative as well as Istio
			- where `managed=true` is set, informing that RH Operator will take care of 
				- installing & managing underlying components *(Serverless/KNative & SvcMesh/Istio)* for Single Model
	- So, if we go wid standard install, it defeats the purpose of existing installation of Istio in another ns
	- **Another challenge**: operator installs components in its own ns, named `istio-system`
		- We do not hv much control over changing this ns
- We r using Istio for all the routes tht r coming in, notebook URLs & everything is coming in via Istio Ingress Gateway
- Istio is alrdy handling the responsibility of
	- all the URLs
	- SSL cert termination
	- SSL certs
	- DNS
	- all the traffic from Istio Ingress GW till actual pod
- So, we dont want standard install to disturb that portion that is alrdy installed.
- So, standard install of Single Model is out of scope



**Manual Install**

- AI Operator understands install type, based on the annotation that we set as managed, unmanaged or removed @ [camp-infrastructure.yaml](https://github.com/cisco-it-cloud-infrastructure/camp-config-dev/blob/master/camp-rtp-dev-01/components/camp-infrastructure.yaml#L196-L214)
	- **Unmanaged**: informs that we install the components _(KNative & Istio)_ & manage their LC, cfg updates etc
	- **Removed**: operator makes sure that specific components, set to Removed, are uninstalled & not consuming any resources
- Extra layer of serverless is also created, that does that actual installation of KNative
	- So, if those serverless variables r not defined, knative is not installed
- Go thru manual steps specified @ [RH Single Model](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.8/html/serving_models/serving-large-models_serving-large-models#manually-installing-kserve_serving-large-models)
	- **Step 1** is to create SvcMesh/Istio, which we alrdy installed
	- **Step 2** is to create Serverless/KNative Serving instance
	- **Step 3** is to do extra cfg of KNative to talk to other svcs
- Since we alrdy had the installations done, we had to do the proper integration
	- installation documentation will alwys talk abt installing components in `istio-system` ns & do this in minimal way
	- we need to adjust accordingly as documentation asks to install KNative ingress Gateway n bunch of other things
	- In our SNCP _(Svc N/w Ctrl Plane)_, we hv installed things manually; not via SNCP but based on the concept of gateway injection
		- **SNCP** -- ctrl plane in svc n/w like Istio's ctrl plane
		- **Gateway injection** is automated insertion of GW proxies into svc mesh setup, like envoy sidecar proxy container with app container
	- followed RH documentation but adapted to our circumstances
- **KNative** install documentation is also written in such a way, assuming that istio system is automatically installed in default ns
	- So, thrs bunch of cfgs here, which they don't mention, that if it is written in a diff. ns, u should add that cfg
	- So, to understand all those missing pieces, open-src documentation had to be referred
		- Go thru KNative open-src documentation @ [KNative](https://knative.dev/docs/serving/)
			- and understand how they suggest to install istio svc mesh
			- their examples have been used in our code
	- open-src is much better documentation than wht is present in RH one
