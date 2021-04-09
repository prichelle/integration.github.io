---
layout: post
title: APIC CLI Getting started
subtitle: How to get the cli and login
tags: [apic, devops]
comments: true
category: apic
---

In this small post I will provide information on how to get started with the APIC cli.

## Introduction
The command line interface, also known as [developer toolkit command line interface](https://www.ibm.com/support/knowledgecenter/SSMNED_v10/com.ibm.apic.toolkit.doc/capim-toolkit-cli-overiew.html), is a powerful tool that can be used 
- to administer the API Connect cloud
- to integrate with your DevOps pipeline. It allows to manage the publication of your APIs as well as their lifecycle. 

You can download the toolkit by navigating to the manager UI and then click on the botton left the help link.

## Getting started

The CLI is used to execute commands within an organisation.
There are two main type of organisation
- the administration organisation. This organisation is used to administer the cloud environment. There is only one administration organisation
- the provider organisation. There may be multiple provider organisations. A provider organisation is used to author, publish and manage the lifecyle of the APIs.

To start with the cli you need
- A **user** and its credentials that is granted to login the organisation
- The realm where this user is known
- the **host name** of the manager endpoint. This is the host name of the [platform API](https://www.ibm.com/support/knowledgecenter/SSMNED_v10/com.ibm.apic.install.doc/capic_deploy_overview.html). API Connect allows to either have the same hostname for the four endpoints or different endpoints.
- the name of the provider organisation is you need to manage your APIs

The steps are  

  a. Login  
  b. execute your commands  
  c. logout  


__1. Get your realm__  
The following command will provide you the id provider that can be used on the login command.  

```
apic identity-providers:list --scope provider -s <platform-api-hostname>

default-idp-2
ibm-common-services
cpldap

```

The id provider that you could use is "default-idp-2" (which is the local APIC UR) or "cpldap" (this is a created UR using an LDAP on this stack).  
The scope can be either **admin** or **provider**.  

__2. Login__  

```
apic login -u <yourUser> --scope <scope> -s <platform-api-hostname> -r <scope>/<id-provider>
```

Example  

```
apic login -u prichelle --scope provider -s <platform-api-hostname> -r provider/cpldap

Enter your API Connect credentials
Password?  
Logged into myapicmgr.com successfully
```

__3. Get your organisation__  

When accessing a provider organisation, the user might be part of multiple organisation. You need to choose the one that you want to work with.

```
apic orgs:list -s <platform-api-hostname> --my
```

Example 

```
apic orgs:list -s myapicmgr.com --my
innovative-org
```

__4. Execute your commands__

For example to list the drafts available in your provider organisation:
```
apic drafts:list -s <platform-api-hostname> --org <providerOrg>
```
> providerOrg is the provider organisation that you got previously (example innovative-org)

List of products deployed in a catalog  
```
apic products --org <providerOrg> --scope catalog --catalog <catalogName> -s <platform-api-hostname>
```

You can find all the possible commands at the [command line tool reference](https://www.ibm.com/support/knowledgecenter/SSMNED_v10/com.ibm.apic.cliref.doc/apic.html)