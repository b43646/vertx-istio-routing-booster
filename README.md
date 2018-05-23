# Istio Routing Mission

## Purpose
Showcases Istio's dynamic routing capabilities with a minimal set of example applications.

## Prerequisites
* Openshift 3.9 cluster
* Istio 0.7.1 with authentication installed on the aforementioned cluster. To install Istio simply follow one of the 
following docs:
  * https://istio.io/docs/setup/kubernetes/quick-start.html
  * https://istio.io/docs/setup/kubernetes/ansible-install.html
* Enable automatic sidecar injection for Istio (See https://istio.io/docs/setup/kubernetes/sidecar-injection.html
   for details)


* In order for Istio automatic sidecar injection to work properly, the following Istio configuration needs to be in place:

  * The `policy` field is set to `disabled` in the `istio-inject` configmap  of the `istio-system` namespace.
  * The `istio-sidecar-injector` `MutatingWebhookConfiguration` should not limit the injection to properly labeled namespaces

* Expose services and Istio ingress:

  In order for Istio automatic sidecar injection to work properly the following Istio configuration needs to be in place:

  The `policy` field is set to `disabled` in the `istio-inject` configmap  of the `istio-system` namespace
  This can be checked by inspecting the output of

  `oc get configmap istio-inject -o jsonpath='{.data.config}' -n istio-system | grep policy`

  The `istio-sidecar-injector` `MutatingWebhookConfiguration` should not limit the injection to properly labeled namespaces.
  If Istio was installed using the default settings, then make sure the output of

  `oc get MutatingWebhookConfiguration istio-sidecar-injector -o jsonpath='{.webhooks[0].namespaceSelector}' -n istio-system`

  is empty. It is advised however that you inspect the output of

   `oc get MutatingWebhookConfiguration istio-sidecar-injector -o yaml`

  to make sure that no other "filters" have been applied.

* Create a new project/namespace on the cluster. This is where your application will be deployed.

```bash
oc new-project <whatever valid project name you want>
```

## Build and deploy the application

### With Fabric8 Maven Plugin (FMP)
Execute the following command to build the project and deploy it to OpenShift:

```bash
mvn clean fabric8:deploy -Popenshift
```

Configuration for FMP may be found both in pom.xml and `src/main/fabric8` files/folders.

This configuration is used to define service names and deployments that control how pods are labeled/versioned on the OpenShift cluster. Labels and versions are key concepts for creating load-balanced or multi-versioned pods in a service.

##  Use Cases

### Configure an ingress Route to access the application

* Create a RouteRule to forward traffic to the demo application. This is only necessary if your application accepts 
traffic at a different port/url than the default. In this case, our application accepts traffic at `/`, but we will access it with the path `/example`.

```bash
oc apply -f rules/client-route-rule.yaml
```

* Access the application
Run the following command to determine the appropriate URL to access our demo. Make sure you access the url with the HTTP scheme. HTTPS is NOT enabled by default:

```bash
echo http://$(oc get route istio-ingress -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/example/
```

The result of the above command is the istio-system istio-ingress URL, appended with the RouteRule path. Open this URL in your a web browser.

### Transfer load between two versions of an application/service

* Access the application as described in the previous use case
  * Click "Invoke Service" in the client UI (Do this several times.)
  * Notice that the services are load-balanced at exactly 50%, which is the default cluster behavior.

* Configure a load-balancing RouteRule. Sometimes it is important to slowly direct traffic to a new service over time, or 
use alternate weighting. In this case, we will supply another Istio RouteRule to control load balancing behavior. Run the 
following command:

```bash
oc apply -f rules/load-balancing-rule.yaml
```

  The RouteRule defined in the file above uses labels "a" and "b" to identify each unique version of the service. If multiple services match any of these labels, traffic will be divided between them accordingly. Additional routes/weights can be supplied using additional labels/service versions as desired.

* Click "Invoke Service" in the client UI. Do this several times. You will notice that traffic is no longer routed at 50/50%, and more traffic is directed to service version B than service version A. Adjust the weights in the rule-file and re-run the command above. You should see traffic adjust accordingly.


_NOTE:_ It could take several seconds for the RouteRule to be detected and applied by Istio.

Congratulations! You now know how to direct traffic between different versions of a service using Istio RouteRules.

## Undeploy the application

### With Fabric8 Maven Plugin (FMP)
```bash
mvn fabric8:undeploy
```

### Remove the namespace
This will delete the project from the OpenShift cluster
```bash
oc delete project <your project name>
```
