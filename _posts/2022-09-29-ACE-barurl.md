---
layout: single
title: Deploy ACE with remote BAR 
subtitle: How to deploy IntegrationServer with BAR file stored in a remote location
tags: [ace]
comments: true
category: ace
author_profile: true
toc: true
toc_sticky: true
---

# Introduction 
There are different ways to deploy an Integration Server on a kubernetes environment:
- build your own image with the AppConnect flow already deployed
- install the AppConnect dashboard and deploy an integration flow using BAR files that have been upload 
- use an external repository to store the bar file and configure the INtegrationServer to download it from there. 

This post focuses on the last option and provides information on how to configure an IntegrationServer to retrieve the BAR file (AppConnect deployment package) from a http remote location secured using TLS and Basic authentication.

# configuration

The Integration Server will be configured with the following parameters :
- barURL: this corresponds to the https link to download the bar file from the remote Repository url.
- configurations: the basic authentication is provided to the integration server using an AppConnect configuration.

## IntegrationServer
Here is an Integration Server example:

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: IntegrationServer
metadata:
  name: ace-restapi-test
  namespace: ace
spec:
  adminServerSecure: true
  barURL: https://eu-de.git.cloud.ibm.com/.../-/raw/main/ace/restapi-sample.bar
  configurations:
  - barauth-ace-config
  createDashboardUsers: true
  designerFlowsOperationMode: disabled
  enableMetrics: true
  license:
    accept: true
    license: L-APEH-CCHL5W
    use: CloudPakForIntegrationNonProduction
  pod:
    containers:
      runtime:
        resources:
          limits:
            cpu: 300m
            memory: 368Mi
          requests:
            cpu: 300m
            memory: 368Mi
  replicas: 1
  router:
    timeout: 120s
  service:
    endpointType: http
  version: "12.0"

```

## AppConnect configuration
The AppConnect configuration, used to reference the basic authentication to authenticate against the repository, can be created using the AppConnect dashboard or using the AppConnect operator user interface available in the openshift console.

The configuration definition reference a kubernetes secret that holds the credentials.
If you are using a script of a cli, you will need to first create the secret(s) before the configuration.

Information related to the barAuth AppConnect configuration can be found at the [knowledge center barurl](https://www.ibm.com/docs/en/app-connect/container?topic=types-barauth-type).

### Secret

The secret contains the credentials information and the certificate if required.

The secret is created using the following command:
```shell
oc create secret generic <barauth-ace-secret> --from-file=configuration=configurationFileName --namespace=namespaceName
```
The content of the secret would looks like:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitbar-ace
  namespace: ace
type: Opaque
data:
  configuration: eyJhdXRoVHlwZSI6IkJBU0lDX0FVVEgiLCJjcmVkZW50aWFscyI6eyJ1c2VybmFtZSI6IiIsInBhc3N3b3JkIjoiIiwiaW5zZWN1cmVTc2wiOiJ0cnVlIn19Cg==
```
The configurationFileName content is in a json format and contains the following information:

- If you are connecting to an endpoint that uses a certificate from a trusted CA, you can connect by using basic authentication without the need to specify any certificate details.
```json
{"authType":"BASIC_AUTH","credentials":{"username":"xxxx","password":"xxxx"}}
```
- If you don't need to have a mutual authentication, it is possible to avoid the certificate validation using the parameter "insecureSsl". 
```json
{"authType":"BASIC_AUTH","credentials":{"username":"xxxx","password":"xxxx","insecureSsl":"true"}}
```
- If you need to provide a certificate, it is possible to provide it in the secret as well.
```json
{"authType":"BASIC_AUTH","credentials":{"username":"xxxx","password":"xxxx","caCertSecret":"secretName"}}
```

If you need to create the secret for the ca certificate, you can use the following yaml:
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: mycaCertSecret
  namespace: namespaceName
data:
  ca.crt: >-
    CAinBase64
  tls.crt: ''
  tls.key: ''
type: kubernetes.io/tls
``` 

Where the CAinBase64 should be replaced by the CA certificate encoded in base64. The following command can be used for this purpose:

```shell
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' test.pem | base64
```

And the secret can then be created using teh following command
```shell
oc apply -f secretFileName.yaml
```

### AppConnect configuration

Finally the configuration will be as follow:

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: Configuration
metadata:
  name: barauth-ace-config
  namespace: ace
spec:
  description: Used to provide repo credentials to ACE
  secretName: gitbar-ace
  type: barauth
  version: 12.0.5.0-r2
```

