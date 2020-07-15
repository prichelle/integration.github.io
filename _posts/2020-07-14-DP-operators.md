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

## Installation

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

## Deploying your first DP
Once the CRD is installed, you can check the existence with:

```shell
kubectl get crd -n <yourNs>| grep -i datapower

datapowerservices.datapower.ibm.com                2020-06-26T12:01:07Z
```

You need to have an operator running.
```shell
kubectl get po -n <yourNs> -l app.kubernetes.io/instance=datapower-operator

datapower-operator-5d88c99bfc-2qs48   1/1     Running   0          18h
```

The datapower service resource requires to have a password secret for the datapower that will be created.
The name of the secret is defined in the DPS yaml under spec->users->passwordSecret
You need to create one before the creation of your CR. This can be achieved using:
```shell
kubectl create secret generic datapower-admin-credentials --from-literal=password=<myPwd> -n dp <myNs>
```

To deploy your datapower, simply create a DataPowerService object using a yaml definition. A minimul vital version:
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
  imagePullSecrets:
  - ibm-entitlement-key
  image: cp.icr.io/cp/apic/ibm-apiconnect-gateway-datapower-prod@sha256:d1ec732177048ce5b8c99d3f5dce43e36f74f4320d222dd45e478c147df1c5e1
```
>Note that image is not mandatory, it depends if you are using a custom registry or not.

And create your object:
```shell
kubectl apply -f <myDPS> -n <myNs>
```

The operator will detects the object and will create the 
- DataPower monitor pod
- DataPower pod

> Note that the operator will apply default values for all the fields that have not be configured explicitly. 
> You can find the list of fields in the [DataPower operator documentation](https://ibm.github.io/datapower-operator-doc/apis/datapowerservice/spec/)

You can check the status of you DataPower deployment by checking the status of your DPS:
```shell
kubectl get dp -n <yourNs>

NAME      READY   SUMMARY                           VERSION    ERROR   AGE
dp-demo   True    StatefulSet replicas ready: 1/1   10.0.0.0   n/a     18h
```

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

## Resources
- [DataPower operator documentation](https://ibm.github.io/datapower-operator-doc/apis/datapowerservice/spec/)
- [DataPower on Docker](https://github.com/ibm-datapower/datapower-labs/tree/master/docker)
- [DataPower on docker hub](https://hub.docker.com/r/ibmcom/datapower/)
- [IBM charts](https://github.com/IBM/charts/tree/master/stable)
### Assets
<a href="../assets/files/ibm-datapower.yaml" download="download">DataPower operator deployment</a>