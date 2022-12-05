---
layout: single
title: Monitoring Integration with Instana
subtitle: Setup integration running on OCP to be monitored by Instana
tags: [ace, monitoring, cp4i]
comments: true
category: cp4i
author_profile: true
toc: true
toc_sticky: true
---


Statistics information are available for MQ, DP and ACE.
Tracing is supported for MQ and ACE. 
DataPower tracing is available for containers as tech preview, more information can be found at the [support page](https://www.ibm.com/support/pages/node/6825263) .

This post explains how to configure AppConnect and Instana when running on OCP.

# Configuration 

To activate the tracing, App Connect needs to have the tracing libraries as well as a configuration to activate it. 

The tracing libraries is provided by mounting a volume to the container.
The configuration is activating though the server.conf.yaml file.
 
If using an IntegrationServer custom resource, the following configuration can be added in the custom resource:

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: IntegrationServer
metadata:
  name: is-acestandalone
spec:
  configurations:
    - instana-server-cfg
  env:
  - name: INSTANA_AGENT_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  pod:
    containers:
      runtime:
        resources:
        volumeMounts:
          - mountPath: /home/aceuser/shared-classes
            name: instana-otracing
    volumes:
      - name: instana-otracing
        persistentVolumeClaim:
          claimName: instana-opentracing
version: "12.0"
```

<mark> Configuration: Instana-server-cfg </mark>

This configuration is used to define the server.conf.yaml file that is used to activate the exit.
It is provided here through the ACE custom resource configuration “instana-server-cfg”:
```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: Configuration
metadata:
  name: instana-server-cfg
  namespace: ace
spec:
  type: serverconf
  contents: VXNlckV4aXRzOgogIGFjdGl2ZVVzZXJFeGl0TGlzdDogJ0FDRU9wZW5UcmFjaW5nVXNlckV4aXQnCiAgdXNlckV4aXRQYXRoOiAnL2hvbWUvYWNldXNlci9zaGFyZWQtY2xhc3Nlcyc=
  version: 12.0.3.0-r1
```
 
The “contents” is base64 encoded and the value is provided here after
```yaml
UserExits:
  activeUserExitList: 'ACEOpenTracingUserExit'
  userExitPath: '/home/aceuser/shared-classes'
```

The operator will add this content (base64 decoded) in the server.conf.yaml file located under the “overrides” directory of the /home/aceuser/ace-server.
 
<mark>volumes instana-otracing</mark>

The tracing libraries required for AppConnect is provided by mounting a volume that contains the openTracing exit (libraries). 
In this example, the volume is mounted on the default “shared-classes”.  

A configuration file needs to be provided to the openTracing configuration having the name  “acetracingexit.conf” (don’t change the file name). 
Content of the file:
```
# configuration for mq tracing exits 2021.4.0
LOG_LEVEL="info" #Log level: info, warn, error, debug
SPAN_FORMAT="instana" #specify opentracing span format as instana or jaeger
MONITOR_LEVEL="verbose" #was set to normal
```
<mark>INSTANA_AGENT_HOST</mark>

The INSTANA_AGENT_HOST needs to have the host where the Instana agent is installed.
As instana agent is running on all worker node, the host is set to the worker node using `status.hostIP`.

# Instana Agent configuration

## App Connect
Nothing needs to be configured. Instana auto discover App Connect instances.

However, if the AppConnect admin rest api is secured with LDAP, the configuration should be added in the instana-agent config map.

# DataPower configuration
 
DataPower tracing is available for containers as tech preview, more information can be found at the [support page](https://www.ibm.com/support/pages/node/6825263) .

The setup is simply made by configuring the instana agent using a configMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: instana-agent
data:
  cluster_name: CP4I
  configuration.yaml: |
    com.instana.plugin.ibmdatapower:
      enabled: true
      poll_rate: 60
      instances:                   # Multiple DataPower instances can be specified
        dp-remote-apicgw-mgmt.apic.svc.cluster.local:                  # Remote DataPower Host or IP address
          port: '5554'             # DataPower instance REST management interface port (required)
         username: ‘xxx’       # User account to connect to DataPower (required)
          password: ‘xxx’   # User password to connect to DataPower (required)
```

The instances correspond to the DataPower hosts. In this case the host is **dp-remote-apicgw-mgmt.apic.svc.cluster.local** which corresponds to the kubernetes service object that matches the DataPower pod with the management port **5554**. 

The username and password to access the REST admin interface should be provided here.
 
# side notes

To support transaction correlation, openTracing needs to store an id within the message that goes through the different integration solutions.  
Usually those information are stored on additional headers and dome attention needs to be taken when using MQ and OpenTracing.
In MQ, there is no place to add information on existing standard MQ message without adding a special MQ Header called **MQRFH2**. Standard MQ message consists of an MQMD header followed by the payload. 
When using MQRFH2 header, the message consists of an MQMD-MQRFH2-Payload. 
> when using JMS, MQRFH2 header is added automatically to include JMS header. 

In most cases, the MQ application consuming this message will not be impacted by this message, but in some case, such as when the application wasn't expected to have another header than the MQMD, application would need to be modified to properly handle the MQ Message having this MQRFH2 header added.  

When an enabled openTracing application puts an MQ message, the application will add this RFH2 header on the background. For example if tracing is activated on App Connect, a tracing code - a user exit program -, this MQRFH2 header will be automatically added to the MQ message. The application that will consume this message would need to be able to support the added MQRFH2 header that wasn’t added before the activation of the tracing.
