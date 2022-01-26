# OpenShift Routes verse Custom Resource Definitions

This document compares the differences between OpenShift Routes and Custom Resource Definitions. Custom Resource allows you to extend Kubernetes capabilities by adding any kind of API object useful for your application. Custom Resource Definition is what you use to define a Custom Resource. This is a powerful way to extend F5 CIS Kubernetes capabilities beyond the default installation and similar to OpenShift Routes.

In this example we are using a cafe application with three endpoints; **tea,coffee and mocha** as shown in the diagram below

![architecture](https://github.com/mdditt2000/openshift-4-9/blob/main/route-vs-crd/diagram/2022-01-26_13-58-40.png)

Demo on YouTube [video]()

## Using OpenShift Route

### Step 1: Deploy CIS

CIS 2.7 communicates directly with BIG-IP DNS via the Rest API and requires gtm-bigip-username and password. Since BIG-IP LTM and DNS are on the same device you can re-use the secret generic bigip-login when deploying CIS as shown below. In this example user-guide the Public IP address for BIGIP VirtualServer will be obtained from F5 IPAM. 

Add the following parameters to THE CIS deployment

* --custom-resource-mode=true - Configure CIS to only monitor CRDs. CIS will ignore all other resources
* --gtm-bigip-username - Provide username for CIS to access GTM
* --gtm-bigip-password - Provide password for CIS to access GTM
* --gtm-bigip-url - Provide url for CIS to access GTM. CIS uses the python SDK to configure GTM
* --ipam=true - CIS to pull VirtualServer IP address from IPAM range

```
args: [
  # See the k8s-bigip-ctlr documentation for information about
  # all config options
  # https://clouddocs.f5.com/containers/latest/
    "--bigip-username=$(BIGIP_USERNAME)",
    "--bigip-password=$(BIGIP_PASSWORD)",
    "--bigip-url=192.168.200.60",
    "--bigip-partition=k8s",
    "--gtm-bigip-username=$(BIGIP_USERNAME)",
    "--gtm-bigip-password=$(BIGIP_PASSWORD)",
    "--gtm-bigip-url=192.168.200.60",
    "--namespace=nginx-ingress",
    "--pool-member-type=cluster",
    "--flannel-name=fl-vxlan",
    "--insecure=true",
    "--custom-resource-mode=true",
    "--as3-validation=true",
    "--log-as3-response=true",
    "--ipam=true",
]
```

Deploy CIS in both locations

```
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=<secret>
kubectl create -f bigip-ctlr-clusterrole.yaml
kubectl create -f f5-bigip-ctlr-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```
- f5-bigip-node is required for Flannel
- bigip-ctlr-clusterrole is required for CIS permissions 

cis-deployment [repo](https://github.com/mdditt2000/k8s-bigip-ctlr/tree/main/user_guides/externaldns-nginx/cis/cis-deployment)

## Using OpenShift Route



