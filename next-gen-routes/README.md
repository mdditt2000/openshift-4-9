# Next Generation OpenShift Routes 

This document compares the differences between **OpenShift Routes** and **OpenShift Routes using Next Generation Routes Controller**. Next Generation Routes Controller **extended F5 Controller Ingress Services to use multiple Virtual IP addresses**. Before F5 CIS could only manage one Virtual IP address per CIS instance. 

In this example we are using a cafe application with three endpoints; **tea,coffee and mocha** as shown in the diagram below

![architecture](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-06_15-57-24.png)

Demo on YouTube [video]()

## Using OpenShift Route

### Step 1: Deploy CIS

Currently in CIS 2.8.1 only one Public IP **Virtual Server** for BIG-IP can be configured for all Routes. Routes uses HOST Header Load balancing to determine the backend application. In this example the backend is **/tea,/coffee and /mocha** using hostname **cafe.example.com**

Add the following parameters to the CIS deployment

* --route-vserver-addr=10.192.125.65 - Public IP for BIG-IP for all Routes
* --manage-routes=true - Configure CIS to watch for Routes
* --bigip-partition=OpenShift - CIS uses BIG-IP tenant OpenShift to manage Routes
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.60",
  "--bigip-partition=OpenShift",
  "--namespace=default",
  "--pool-member-type=cluster",
  "--openshift-sdn-name=/Common/openshift_vxlan",
  "--insecure=true",
  "--manage-routes=true",
  "--route-vserver-addr=10.192.125.65",
  "--as3-validation=true",
  "--log-as3-response=true",
]
```

Deploy CIS in OpenShift

```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
oc create -f bigip-ctlr-clusterrole.yaml
oc create -f f5-bigip-ctlr-deployment.yaml
```

CIS [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/single-vip/cis)

**Note** Do not forget the CNI [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/single-vip/cni)

### Step 2: Creating OpenShift Routes

User-case for the OpenShift Routes:

- Edge Termination
- Redirect HTTP to HTTPS
- Health monitor of the backend NGINX application using HOST **cafe.example.com** and **PATH /coffee, /tea and /mocha**
- Backend listening on PORT 8080

Create OpenShift Routes

```
oc create -f cafe-override.yaml
oc create -f route-tea.yaml
oc create -f route-coffee.yaml
oc create -f route-mocha.yaml
```
Routes [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/single-vip/ocp-route)

Validate OpenShift Routes using the OpenShift Dashboard

![route](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-01-27_16-53-47.png)

Validate OpenShift Routes using the BIG-IP

![big-ip route](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-06_12-39-40.png)

Validate OpenShift Routes pool-members using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-01-27_11-38-40.png)

Validate OpenShift Routes by connecting to the Public IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-01-27_11-44-57.png)

## Next Generation OpenShift Routes

### Step 1: Deploy CIS

Next Generation Routes Controller uses extended ConfigMap, allowing the user to create multiple Virtual IP addresses for OpenShift Routes. Support for multi-partition is also available.

Creating a second Public IP **Virtual Server** for BIG-IP to handle a difference group of applications, namespace or project. Routes uses HOST Header Load balancing to determine the backend application. In this example the backend is **/tea,/coffee and /mocha** using hostname **"cafeTwo.example.com"**

Add the following parameters to the CIS deployment

* Routegroup specific config for each namespace is provided as part of extendedSpec through Configmap.
* ConfigMap info is passed to CIS with argument --route-spec-configmap="namespace/configmap-name"
* Controller mode should be set to openshift to enable multiple VIP support(--controller-mode="openshift")

Using Local ConfigMap

 * Local configmap is used to specify route config for namespace and allows tenant users access to fine tune the route config. It is processed by CIS only when allowOverride is set to true in global confimap for this namespace.
* Only one local configmap is allowed per namespace. Local configmap must have only one entry in extendedRouteSpec list and that should be the current namespace only

* --route-vserver-addr=10.192.125.65 - Public IP for BIG-IP for all Routes
* --manage-routes=true - Configure CIS to watch for Routes
* --bigip-partition=OpenShift - CIS uses BIG-IP tenant OpenShift to manage Routes
* --openshift-sdn-name=/Common/openshift_vxlan - CNI policy on BIG-IP to connect to the PODs in OpenShift

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
  "--bigip-username=$(BIGIP_USERNAME)",
  "--bigip-password=$(BIGIP_PASSWORD)",
  "--bigip-url=10.192.125.60",
  "--bigip-partition=OpenShift",
  "--namespace=default",
  "--pool-member-type=cluster",
  "--openshift-sdn-name=/Common/openshift_vxlan",
  "--insecure=true",
  "--manage-routes=true",
  "--route-spec-configmap="default/global-cm"
  "--controller-mode="openshift"
  "--as3-validation=true",
  "--log-as3-response=true",
]
```