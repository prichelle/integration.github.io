---
layout: single
title: Threads limit for MQ on containers
subtitle: Evaluate and set the right max thread limits when running on a k8s environment
tags: [mq, k8s]
comments: true
category: MQ
author_profile: true
toc: true
toc_sticky: true
---

# Introduction

MQ is a multithreaded solution that might spawn a relative high number of threads that might be incompatible with the default value of the operating system when it will run.

OpenShift limit now by default the maximum threads number that can be spawn in a container to 1024 which might be not compatible for your MQ deployment.  

In this short post, I will provide some information and references on how to list the processes that are running on a linux system and how to configure OCP to allow more threads.

# Pids on your container

When MQ is running on a container, you can list the number of threads that have been created.
Log into the container and issue the following command:

```shell
sh-4.4$ ps -eo nlwp | tail -n +2 | awk '{ num_threads += $1 } END { print num_threads }'
240
```
or 
```
cat /sys/fs/cgroup/pids/pids.current
238
```


By using `ps` you will be able to list all the threads and process that are running:

```
sh-4.4$ ps -eLf       
UID         PID   PPID    LWP  C NLWP STIME TTY          TIME CMD
1000730+    108      1    108  0   14 Oct18 ?        00:00:25 /opt/mqm/bin/amqzxma0 -m mainmqm -x -u mqm
1000730+    108      1    132  0   14 Oct18 ?        00:00:05 /opt/mqm/bin/amqzxma0 -m mainmqm -x -u mqm
....
``` 

If you would like to know how many threads is running for a specific process, I would recommend to use top: 
- in a terminal issue the command "top"
- press "f" and go over **nTH** with the arrow 
- press "space"
- press "q"

![MQ Top](/assets/images/mq/mq-top.png)

To know what are the MQ tasks please refer to the [MQ documentation](https://www.ibm.com/docs/en/ibm-mq/9.3?topic=i-mq-tasks).

# Max threads

To list the max threads configured on the system:

```
sh-4.4$ ulimit -a | grep -i processes
max user processes              (-u) 255833
```

# MQ threads
One object that can consume quite a lot of threads is the MQ Channel.

When an application is connecting through a client connection, on MQ side a server connection channel instance is started. This type of channels are called MQI channels.   
In a multithreaded application, it is possible to create a connection to the queue manager within each thread as described for example fo [a multithreaded java app](https://www.ibm.com/docs/en/ibm-mq/9.3?topic=applications-multithreaded-programs-in-java).  
Each client connection results on a new MQI channel instance on the server side which corresponds to a new thread.

Therefore it is a good practices to limit the maximum number of channel instances that can be active at the same time for a specific server connection channel to avoid a badly written application to exhaust the maximum limit allowed on the server. This can be done using the **MAXINST** parameter described in the [sever-connection channel limits](https://www.ibm.com/docs/en/ibm-mq/9.3?topic=function-server-connection-channel-limits).  

It is also a good practice to create a server connection channel per application such that you have more control on the connection.

You can display the number of instances per connection on a queue manager using **CHSTATUS**.

```
echo "DIS CHSTATUS(*) WHERE(CHLTYPE eq SVRCONN)" | runmqsc <yourQMgr>
```
An example of output can be:
```
     1 : DIS CHSTATUS(*) WHERE(CHLTYPE eq SVRCONN)
AMQ8417I: Display Channel Status details.
   CHANNEL(CL.PAYMENTS)                    CHLTYPE(SVRCONN)
   CONNAME(172.30.54.27)                   CURRENT
   STATUS(RUNNING)                         SUBSTATE(RECEIVE)
AMQ8417I: Display Channel Status details.
   CHANNEL(CL.PAYMENTS)                    CHLTYPE(SVRCONN)
   CONNAME(172.30.54.27)                   CURRENT
   STATUS(RUNNING)                         SUBSTATE(RECEIVE)
AMQ8417I: Display Channel Status details.
   CHANNEL(CL.PAYMENTS)                    CHLTYPE(SVRCONN)
   CONNAME(172.30.54.13)                   CURRENT
   STATUS(RUNNING)                         SUBSTATE(RECEIVE)
```
We can see that the server connection channel "CL.PAYMENTS" has three active connections from two different applications. One application with the connection name 172.30.54.27 has two active connection.

You can count the number of connection for a specific CONNAME: 
```
echo "DIS CHSTATUS(*) " | runmqsc <yourqmgr> | grep <conname> -A1 | grep CONNAME | sort | uniq -c
```
Example:
```
sh-4.4$ echo "DIS CHSTATUS(*) " | runmqsc mainmqm | grep 172.30.54.27 -A1 | grep CONNAME | sort | uniq -c
      2    CONNAME(172.30.54.27)                   CURRENT
```

If you want to further analyze what this application is doing, you can list the connection for this specific application (172.30.54.27) or channel (CL.PAYMENTS) using the following code matching our example:
```
echo "DIS CONN(*) WHERE (CHANNEL EQ CL.PAYMENTS) ALL" | runmqsc mainmqm
AMQ8276I: Display Connection details.
   CONN(12346263002D0040)                
   EXTCONN(414D51436D61696E6D716D2020202020)
   TYPE(CONN)                            
   PID(148477)                             TID(6) 
   APPLDESC(IBM MQ Channel)                APPLTAG(cli.ConnectDistributed)
   APPLTYPE(USER)                          ASTATE(NONE)
   CHANNEL(CL.PAYMENTS)                    CLIENTID( )
   CONNAME(172.30.54.13)                
   CONNOPTS(MQCNO_HANDLE_SHARE_BLOCK,MQCNO_SHARED_BINDING,MQCNO_GENERATE_CONN_TAG)
   USERID(root)                            UOWLOG( )
   UOWSTDA( )                              UOWSTTI( )
   UOWLOGDA( )                             UOWLOGTI( )
   URTYPE(QMGR)                         
   EXTURID(XA_FORMATID[] XA_GTRID[] XA_BQUAL[])
   QMURID(0.0)                             UOWSTATE(NONE)
   CONNTAG(MQCT12346263002D0040mainmqm_2021-11-04_10.01.18cli.ConnectDistributed)
```
or for the conname:
```
echo "DIS CONN(*) WHERE (CONNAME EQ 172.30.54.13) ALL" | runmqsc mainmqm
```

And finally determine what queue is been used by this application using QSTATUS and search for the PID or CHANNEL. 
Example using the PID:

```
sh-4.4$ echo "DIS QS(*) WHERE(PID EQ 148477) type(handle) all" | runmqsc mainmqm
5724-H72 (C) Copyright IBM Corp. 1994, 2022.
Starting MQSC for queue manager mainmqm.


     1 : DIS QS(*) WHERE(PID EQ 148477) type(handle) all
AMQ8450I: Display queue status details.
   QUEUE(LQ.PAYMENTS.EVENT.SOURCE)         TYPE(HANDLE)
   APPLDESC(IBM MQ Channel)                APPLTAG(cli.ConnectDistributed)
   APPLTYPE(USER)                          BROWSE(YES)
   CHANNEL(CL.PAYMENTS)                    CONNAME(172.30.54.13)
   ASTATE(ACTIVE)                          HSTATE(ACTIVE)
   INPUT(SHARED)                           INQUIRE(YES)
   OUTPUT(NO)                              PID(148477) 
   QMURID(0.28672)                         SET(NO)
   TID(*)                               
   URID(XA_FORMATID[] XA_GTRID[] XA_BQUAL[])
   URTYPE(QMGR)                            USERID(root)
AMQ8450I: Display queue status details.
   QUEUE(LQ.PAYM.PROC.REQ)                 TYPE(HANDLE)
   APPLDESC(IBM MQ Channel)                APPLTAG(IntegrationServ)
   APPLTYPE(USER)                          BROWSE(NO)
   CHANNEL(CL.PAYMENTS)                    CONNAME(172.30.54.27)
   ASTATE(ACTIVE)                          HSTATE(ACTIVE)
   INPUT(SHARED)                           INQUIRE(YES)
   OUTPUT(NO)                              PID(148477) 
   QMURID(0.12290)                         SET(NO)
   TID(*)                               
   URID(XA_FORMATID[] XA_GTRID[] XA_BQUAL[])
   URTYPE(QMGR)                            USERID(root)
One MQSC command read.
No commands have a syntax error.
All valid MQSC commands were processed.
```

# Setting the pid max

Information can be found on the knowledge center under [linux tuning](https://www.ibm.com/docs/en/ibm-mq/9.3?topic=linux-configuring-tuning-operating-system).

For Kubernetes and more specifically on OCP it is described here on [openshift configuration](https://ibm-messaging.github.io/mqperf/openshift/configuration.html).

More detailed information can be found on this page [comuting for geek - change pids limit in OCP](https://computingforgeeks.com/change-pids-limit-value-in-openshift/).

In a nutshell this is done by creating a ContainerRuntimeConfig CR on OCP that defines the pidsLimit and then assign this configuration on the required node by using a specific label.
Example of ContainerRuntimeConfig:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: ContainerRuntimeConfig
metadata:
  name: set-pid-4k
spec:
  machineConfigPoolSelector:
    matchLabels:
      config-crio: set-pid
  containerRuntimeConfig:
    pidsLimit: 4096
``` 

You can find what value is currently set on your worked node by using the following command where the node name in this example is "10.75.10.195":
```
oc debug node/10.75.10.195
Starting pod/107510195-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.75.10.195
sh-4.4# chroot /host
sh-4.2# grep pids_limit /etc/crio/crio.conf
pids_limit = 102333
```

# References
Very good article on how to get information about linux processes:
[Max thread processes](https://www.golinuxcloud.com/check-threads-per-process-count-processes/)


