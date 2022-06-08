# Next Generation OpenShift Routes 

This document demonstrates a new feature for OpenShift Routes using F5 Controller Ingress Services (CIS) called **Next Generation Routes Controller**. Next Generation Routes Controller extended F5 CIS to use multiple Virtual IP addresses. Before F5 CIS could only manage one Virtual IP address per CIS instance. 

In this example we are using a **cafe** and **cafenew** application with three endpoints; **tea,coffee and mocha** and **multiple namespace** as shown in the diagram below. 

![architecture](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-08_11-09-57.png)

Demo on YouTube [video]()

## Using OpenShift Route

### Step 1: Deploy CIS

Currently in CIS 2.8.1 only one **Public IP**, **Virtual Server** for BIG-IP can be configured for all Routes. Routes uses HOST Header Load balancing to determine the backend application. In this first example the backend is **/tea,/coffee and /mocha** using hostname **cafe.example.com** with **Virtual IP Address 10.192.125.65**

Next Generation Routes Controller uses extended ConfigMap, allowing the user to create multiple Virtual IP addresses for OpenShift Routes. Support for multi-partition is also available. Each namespace will be managed in a dedicated partition/tenant on BIG-IP

Add the following parameters to the CIS deployment

* Routegroup specific config for each namespace is provided as part of extendedSpec through ConfigMap.
* ConfigMap info is passed to CIS with argument --route-spec-configmap="namespace/configmap-name"
* Controller mode should be set to openshift to enable multiple VIP support(--controller-mode="openshift")

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
  "--route-spec-configmap="kube-system/global-cm"
  "--controller-mode="openshift"
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

CIS [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/multi-vip/cis)

**Note** Do not forget the CNI [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/multi-vip/cni)

### Step 2: Deploy Global ConfigMap

Using Global ConfigMap

* Global ConfigMap provides control to the admin to create and maintain the resource configuration centrally.
* RBAC can be used to restrict modification of global ConfigMap by users with tenant level access.
* If any specific tenant requires modify access for routeconfig of their namespace, admin can grant access by setting allowOverride to true in the extendedRouteSpec of the namespace.

* namespace: cafe, vserverAddr: 10.192.125.65

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-cm
  namespace: kube-system
  labels:
    f5nr: "true"
data:
  extendedSpec: |
    extendedRouteSpec:
    - namespace: cafe
      vserverAddr: 10.192.125.65
      vserverName: cafe
      allowOverride: true
```

Deploy global ConfigMap

```
oc create -f global-cm.yaml
```

### Step 3: Creating OpenShift Routes for cafe.example.com

User-case for the OpenShift Routes:

- Edge Termination
- Backend listening on PORT 8080

Create OpenShift Routes

```
oc create -f route-tea-edge.yaml
oc create -f route-coffee-edge.yaml
oc create -f route-mocha-edge.yaml
```

Routes [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/multi-vip/ocp-route/cafe/secure)

Validate OpenShift Routes using the BIG-IP

![big-ip route](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-07_15-35-21.png)

Validate OpenShift Virtual IP using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-07_15-37-33.png)

Validate OpenShift Routes policies on the BIG-IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-07_15-38-08.png)

Validate OpenShift Routes policies by connecting to the Public IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-07_15-38-33.png)

### Step 3: Creating OpenShift Routes for New Domain cafenew.example.com

Creating a second Public IP **Virtual Server** for BIG-IP to handle a difference group of applications, namespace or project. Routes uses HOST Header Load balancing to determine the backend application. In this example the backend is **/tea,/coffee and /mocha** using hostname **"cafenew.example.com"**

### Step 2: Apply Global ConfigMap Update

Update global ConfigMap with the second Virtual IP address for the new namespace **cafenew.example.com**

* namespace: cafenew, vserverAddr: 10.192.125.66

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-cm
  namespace: kube-system
  labels:
    f5nr: "true"
data:
  extendedSpec: |
    extendedRouteSpec:
    - namespace: cafe
      vserverAddr: 10.192.125.65
      vserverName: cafe
      allowOverride: true
    - namespace: cafenew
      vserverAddr: 10.192.125.66
      vserverName: cafenew
      allowOverride: true
```

Deploy the global ConfigMap update

```
oc apply -f global-cm.yaml
```

### Step 3: Creating OpenShift Routes

User-case for the OpenShift Routes:

- Edge Termination
- Backend listening on PORT 8080

Create OpenShift Routes

```
oc create -f route-tea-edge.yaml
oc create -f route-coffee-edge.yaml
oc create -f route-mocha-edge.yaml
```

Routes [repo](https://github.com/mdditt2000/openshift-4-9/tree/main/next-gen-routes/multi-vip/ocp-route/cafenew/secure)

Validate OpenShift Routes using the BIG-IP

![big-ip route](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-08_09-59-39.png)

Validate OpenShift Virtual IP using the BIG-IP

![big-ip pools](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-08_10-01-21.png)

Validate OpenShift Routes policies on the BIG-IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-08_10-02-44.png)

Validate OpenShift Routes policies by connecting to the Public IP

![traffic](https://github.com/mdditt2000/openshift-4-9/blob/main/next-gen-routes/diagram/2022-06-08_10-03-40.png)