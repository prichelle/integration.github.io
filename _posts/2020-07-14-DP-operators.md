---
layout: post
title: DataPower Operators  
subtitle: k8s DataPower on steroids
tags: [dp, admin, k8s]
comments: true
category: dp
---

In this post I would go through an incredible new powerful feature that has been introduce to deploy DataPower on a Kubernetes cluster: **[DataPower operator](https://ibm.github.io/datapower-operator-doc/)**

## Introduction
For those that are not familiar with operators, here is a quick introduction.

From [kubernetes definition](https://kubernetes.io/), we can read:
> Operators are software extensions to Kubernetes that make use of custom resources to manage applications and their components. Operators follow Kubernetes principles, notably the control loop.

It means that **operators** are running as pod on your cluster and are interacting with the Kubernetes Control plane. 
They are watching the cluster through the control plane to get information about specific resource that has been created.   
They are responsible to execute the necessary actions o, the cluster such that the final state corresponds to what has been defined by those specific resource.  
Type of actions that they might do:
- resource creation: create replica sets, execute prescript jobs, instantiate pods, ...
- upgrade: performs the necessary tasks to upgrade the deployed software ... 

The specific resource that the DataPower operator is in control is a kubernetes **[custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)** called **DataPowerService**.  
This custom resource is defined on the kubernetes cluster by applying a **custom resource definition**. 

Once the custom resource definition has been created on the cluster it is possible to create custom ressource by applying a yaml file that defines the fields that corresponds to this specific resource.
This yaml file will contains configuration for your specific DataPower like ports, volumes, environement variables, ....

In summary:
- The custom resource definition DataPowerService is used to tells kubernetes what is a DataPowerService, what fields it contains, what operations are possible, ...
- A custom resource DataPowerService is created with your specific configuration
- The DataPower operator detects that a DataPowerService resource has been created, retrieves the information from it and based on this information creates the datapower pods (pulled from [ibmcom/datapower](https://hub.docker.com/r/ibmcom/datapower/)) and statefulsets.
- If something is changed on the DataPowerService resource, the operator will act accordingly.

The magic comes once the operator is deployed, you just create your CR DataPowerService and the datapower is deployed.

## DataPower installation using operator

### prereqs

A Kubernetes cluster.

To illustrate the steps, I am using in this post a k8s cluster on [IBM Cloud Kubernetes Service](https://www.ibm.com/cloud/container-service/).
The [cluster setup](#IKS-cluster-setup) can be found at the end of the post.

The following cli needs to be installed:
- helm v3
- kubectl

> a wrong version of kubectl can lead to this error : "error: No Auth Provider found for name "oidc"


The DataPower version that will be deployed is a developer edition.

### Principle
The installation and use of the new feature follow these steps:
1. Install the CRD
2. Deploy a DataPower operator
3. Create a CR

The steps 1 and 2 can be performed using one of the other following approaches as documented at [IBM DataPower Operator](https://ibm.github.io/datapower-operator-doc/):
- using the OLM [Operator Lifecyle Manager](https://docs.openshift.com/container-platform/4.2/operators/understanding_olm/olm-understanding-olm.html) available with OpenShift
- using [Helm charts](https://github.com/IBM/datapower-operator-chart)

As explained previously the operator is a controller that runs as a pod and watching for DataPowerService CRDs.
You may choose to deploy one operator for the whole cluster that will watch all namespaces or to deploy one operator in each namespace that will be responsible for this specific namespace only. 

The operator can be deployed using helm or using simple yaml file that will create the following resources:
- The cluster role: (his might be created once and allows the operator to interact with your cluster
- A service account: this service will be used by the operator and is linked to the cluster role with a cluster role binding. Each namespace will have it's own service account
- A cluster role binding: it is used to link the sa user used by the operator to the cluster role
- Deployment: defines the 
    - replica set for HA
    - the secret used to access the registry to pull the datapower image and operator
    - the container spec that uses the image *ibmcom/datapower-operator:1.0.0* 

We will see how in the next section how this is done in real.


### Install the DataPower operator

- Create a namespace: ``` kubectl create ns <myNs> ```
- Install the operator. This is done using the provided [helm chart](https://gi thub.com/IBM/datapower-operator-chart/tree/master/charts/stable/datapower-operator)  
```
git clone git@github.com:IBM/datapower-operator-chart.git
cd datapower-operator-chart/charts/stable/datapower-operator
helm install dp-operator . --namespace <yourNs>
```
- The helm install
  - the DataPowerService CRD. It can be checked with ``` kubectl get crd | grep -i datapower ```
  - a service account **dp-operator-default-datapower-operator** to run the operator
  - a clusterrole for the operator: "dp-operator-default-datapower-operator" ``` kubectl get clusterrole | grep -i datapower ```
  - a clusterrolebinding "dp-operator-default-datapower-operator" to link the serviceaccount to the clusterrole. ``` kubectl get clusterrolebinding | grep -i datapower ```
  - an operator into the namespace. 
  
To check the operator, the following command can be used:  
```
kubectl get po | grep datapower

datapower-operator-5577975b6-79cxj   1/1     Running   0          28s
```


### Deploying DataPower

You need to have an operator running.
```shell
kubectl get po -n <yourNs> -l app.kubernetes.io/instance=datapower-operator

datapower-operator-5d88c99bfc-2qs48   1/1     Running   0          18h
```

The datapower service resource requires to have a password for the DataPower admin user. The password is provided using a secret.
The name of the secret is defined in the DPS yaml under spec->users->passwordSecret.
Here is an example on how to create a secret holding a password:
```
kubectl create secret generic datapower-admin-credentials --from-literal=password=helloworld --from-literal=salt=12345678 --from-literal=method=md5 -n <myNs>
```

To deploy your datapower, simply create a DataPowerService object using a yaml definition. Create the file (for minimum configuration) dps-dp-demo.yaml:
```yaml
apiVersion: datapower.ibm.com/v1beta1
kind: DataPowerService
metadata:
  name: dp-demo
spec:
  license:
    accept: true
    use: developers-limited
  replicas: 1
  users:
  - accessLevel: privileged
    name: admin
    passwordSecret: datapower-admin-credentials
  version: 10.0.0
```
>Note that image is not mandatory, it depends if you are using a custom registry or not.

And create your object:
```shell
kubectl apply -f dps-dp-demo.yaml -n <myNs>
```

You can check the status of you DataPower deployment by checking the status of your DPS:
```shell
kubectl get dp -n <yourNs>

NAME      READY   SUMMARY                           VERSION    ERROR   AGE
dp-demo   True    StatefulSet replicas ready: 1/1   10.0.0.0   n/a     18h
```
The operator will detect the DPS customer resource and will create the 
- DataPower monitor pod
- DataPower pod

> Note that the operator will apply default values for all the fields that have not been configured explicitly. 
> You can find the list of fields in the [DataPower operator documentation](https://ibm.github.io/datapower-operator-doc/apis/datapowerservice/spec/)

## Troubleshooting
Check the status of your DPS. For example here the secret were missing:
```shell
kubectl get dp -n <yourNs>

NAME      READY   SUMMARY                           VERSION    ERROR                                  AGE
dp-demo   False   StatefulSet replicas ready: 0/1   10.0.0.0   Secret "admin-credentials" not found   5m30s
```

Check the logs of the operator:
```shell
kubectl logs datapower-operator-... -n <yourNs>
```  


## IKS cluster setup 

### Prereq

- An IBM account
Install the following cli (be sure to have the proper version - had an oidc issue due to a wrong version)
- [ibmcloud cli](https://cloud.ibm.com/docs/cli?topic=cli-getting-started)

### Setup
This part will explain how to setup a k8s cluster on IKS.
Most of the steps here were found in the great blog from Chris on [APIC on IKS blog](https://chrisphillips-cminion.github.io/apiconnect/2020/06/24/APIConnect-v10-install-on-IKS.html).

- Provision a cluster with one node with at least 5 cores
- Log into your cluster  
```shell
ibmcloud login -sso
```
  - Get your OTP code and login
  - Choose your account
  - check that you are in the right region. Region can be changed with "-r". More information on the [IBM cli doc](https://cloud.ibm.com/docs/cli?topic=cli-ibmcloud_cli)

- check the cluster that you have 
```
ibmcloud cs clusters
```
- set the cluster configuration for your helm and kubectl
````
ibmcloud cs cluster config --cluster <clustername>
````
- init helm. Helm will be used to setup the block storage and for installing the DataPower operator.  
Note that if you don't want to persist any DataPower configuration, block storage is not required.
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm init
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
- Install IBM Block Storage
````
helm repo add iks-charts https://icr.io/helm/iks-charts
helm repo update
helm install --name ibm-block-storage iks-charts/ibmcloud-block-storage-plugin -n kube-system
````
> class storage can be double check with ```  kubectl get storageclass  ```

If you would like to access your dp from outside, you can install ingress as described in the [APIC on IKS blog](https://chrisphillips-cminion.github.io/apiconnect/2020/06/24/APIConnect-v10-install-on-IKS.html) at Part 2.6 and 7

## Resources
- [DataPower operator documentation](https://ibm.github.io/datapower-operator-doc/apis/datapowerservice/spec/)
- [DataPower on Docker](https://github.com/ibm-datapower/datapower-labs/tree/master/docker)
- [DataPower on docker hub](https://hub.docker.com/r/ibmcom/datapower/)
- [IBM charts](https://github.com/IBM/charts/tree/master/stable)

- [API Connect on IKS](https://chrisphillips-cminion.github.io/apiconnect/2020/06/24/APIConnect-v10-install-on-IKS.html)
