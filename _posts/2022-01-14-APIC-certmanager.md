---
layout: single
title: API Connect V10 Cert Manager
subtitle: Check and Update self-signed certificate with new cert manager
tags: [apic, certificate, k8s]
comments: true
category: apic
author_profile: true
toc: true
toc_sticky: true
---

In the latest release the certificate manager has been changed and this might generate some issue when you upgrade your cluster from a previous version.
The cert manager used previously is "certmanager.k8s.io" and it has been upgraded to "cert-manager.io".

The procedure to upgrade the APIC Cluster is well documented:

[Upgrading on OpenShift - point 6](https://www.ibm.com/docs/en/api-connect/10.0.x?topic=uo-upgrading-openshift-cloud-pak-integration-in-online-environment)

The purpose of this topic is not to provide a new procedure but rather provide an introduction on how the tls certificate are used and generated with cert manager especially when using self-signed certificate.  
This helps to better understand what is happening behind the scene.  

It also provide some guidance and help if you have issue or error reported to certificate that have been expired or TLS connection error between APIC components.  

Indeed, if the CA certs have been refreshed and self signed certificate are used, it might be possible that some certificates are still signed with the old certificate. 

As API Connect is based on microservices and are communicating using TLS, it might be possible to have a component attempting to call another one with an old certificate which can not be verified using the new CA.  

I will provide first a small introduction on how certificate are issued into k8s, then I provide some commands that can be used to verify that the certificate are right and finally I provide some useful commands.



# Kubernetes Issuer & Certificates

Certificate object are used to create x509 certificate such as ca.crt, tls.crt and tls.key that are stored into Kubernetes Secret.  The object describe the content of the x509 certificate generated.   
 
The certificate object are used by the Issuer k8s object that generates from this configuration k8s Secrets containing the x509 certificates.  

There are different types of issuers and we focus here on SelfSigned and CA issuers.

SelfSigner Issuer generates a Secret with X509 certificate using a  Certificate k8s object. Each new secret created by this issuer will have distinct CA certificate within it.  

When using self signed certificate, it is common usage to use the self-signed issuer to issue self-signed certificate that will be used as CA root certificate. As we will see there is a special Issuer, called the CA Issuer, that can be used to generate secret having the same common CA certificate.

## SelfSigned Issuer
SelfSigned Issuer means that the Issuer will be able to signed certificate with a private key. This Issuer is defined using  

```yaml
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: sandbox
spec:
  selfSigned: {}
```  

When deployed it will be ready to sign certificate:
```shell
$ kubectl get issuers  -n sandbox -o wide selfsigned-issuer
NAME                READY   STATUS                AGE
selfsigned-issuer   True                          2m
```

Now if a certificate object is created using the self signed Issuer, the Issuer will create the ca.crt, tls.crt and tls.key.
The certificate object:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
spec:
  isCA: true
  duration: 87600h0m0s
  renewBefore: 720h0m0s
  commonName: selfsigned-ca
  secretName: selfsigned-ca-secret
  privateKey:
    algorithm: ECDSA
    rotationPolicy: Always
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
    group: cert-manager.io
```
The following TLS secret is created:

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    cert-manager.io/certificate-name: selfsigned-ca
    cert-manager.io/issuer-kind: Issuer
    cert-manager.io/issuer-name: selfsigned-issuer
  name: selfsigned-ca-secret
data:
  ca.crt: ###
  tls.crt: ###
  tls.key: ###
```

The content of the CA certificate generated (ca.crt in the secret) is:

``` yaml
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            56:2d:3c:c8:f1:ad:03:b3:b5:7a:aa:65:02:9e:5c:f3
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: CN=selfsigned-ca
        Validity
            Not Before: Jan  5 08:21:25 2022 GMT
            Not After : Jan  3 08:21:25 2032 GMT
        Subject: CN=selfsigned-ca
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE

```

CA Certificate object defines what issuer to use --> The Issuer (selfsigned) will issue a Secret --> TLS secret (ca.crt, tls.crt/key)

> Note about CA certificate: CA certificate is used only to sign other certificates and CRLs; validate the signature of issued certificates; [Optionally] set restrictions or constraints on the configuration of issued certificates.

To get additional information on the certificate, use the "wide" option:  

```shell
oc get cert -o wide                                                                                                                                                                       0.505s 10:54
NAME              READY   SECRET                   ISSUER              STATUS                                          AGE    EXPIRATION
selfsigned-ca     True    selfsigned-ca-secret     selfsigned-issuer   Certificate is up to date and has not expired   115m   2032-01-03T07:59:03Z
```

>The certificate "selfsigned-ca" has been used to create the secret "selfsigned-ca-cert" using the issuer "selfsigned-issuer".

## CA Issuer
The CA issuer represents a Certificate Authority whereby its certificate and private key are stored inside the cluster as a Kubernetes Secret.

In the previous section, a self signed issuer has been used to create a tls secret (ca, tls) based on a CA certificate description.

Now we are going to use this certificate as root CA certificate for the CA Issuer. The secrets that will be generated using this CA issuer will all contains the same ca certificate.  

The CA issuer is defined as follow where you have to specify the ca certificate that you would like to use:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: selfsigned-ca-secret
```

If you list now the issuer:
```shell
oc get issuer -o wide                                                                                                                                                                     5.855s 11:28
NAME                READY   STATUS                AGE
ca-issuer           True    Signing CA verified   35m
selfsigned-issuer   True                          168m
```

From here you can create multiple certificates that will have the same CA as signer.  
This is done by creating a certificate that reference this CA issuer:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert-ca-tls
spec:
  duration: 87600h0m0s
  renewBefore: 720h0m0s
  commonName: cert-ca-tls
  secretName: cert-ca
  privateKey:
    algorithm: ECDSA
    rotationPolicy: Always
    size: 256
  issuerRef:
    name: ca-issuer
    kind: Issuer
    group: cert-manager.io
```

## Additional information on Certificate 

Some interesting extension used in the certificate.

### Subject Alternative Name

The common name represents the host name that’s covered by the SSL certificate. Trying to use the certificate for a website that doesn’t match the common name will result in a security error, also known as host name mismatch error.
After the original specificaton, it became clear it would be helpful to have a single certificate to cover multiple host names.
The Subject Alternative Name (SAN) is an extension to the X.509 specification that allows users to specify additional host names for a single SSL certificate.

### Extended Key Usage
Extended key usage further refines key usage extensions. An extended key is either critical or non-critical. If the extension is critical, the certificate must be used only for the indicated purpose or purposes. If the certificate is used for another purpose, it is in violation of the CA's policy.


# Certificate Verification

## Check the certificates
Let's see how this can be leverage to check the certificate in API Connect.
Let's apply this to the management subsystem.


### 1 - List the secret with their issuer  

The following command provides the list of certificate object for the management:  

```shell  

oc get cert -o wide | grep mgmt

apic-mgmt-aa286878-postgres             True    apic-mgmt-aa286878-postgres             apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:04:29Z
apic-mgmt-aa286878-postgres-pgbouncer   True    apic-mgmt-aa286878-postgres-pgbouncer   apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:04:34Z
apic-mgmt-admin                         True    apic-mgmt-admin                         apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:05:08Z
apic-mgmt-api-manager                   True    apic-mgmt-api-manager                   apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:05:05Z
apic-mgmt-ca                            True    apic-mgmt-ca                            apic-self-signed      Certificate is up to date and has not expired   9d    2032-01-02T17:00:49Z
apic-mgmt-client                        True    apic-mgmt-client                        apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:15Z
apic-mgmt-consumer-api                  True    apic-mgmt-consumer-api                  apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:05:07Z
apic-mgmt-hub                           True    apic-mgmt-hub                           apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:07:12Z
apic-mgmt-natscluster-mgmt              True    apic-mgmt-natscluster-mgmt              apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:53Z
apic-mgmt-platform-api                  True    apic-mgmt-platform-api                  apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:07:22Z
apic-mgmt-server                        True    apic-mgmt-server                        apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:30Z
apic-mgmt-turnstile                     True    apic-mgmt-turnstile                     apic-ingress-issuer   Certificate is up to date and has not expired   9d    2023-06-18T17:07:23Z
apicuser                                True    apic-mgmt-db-client-apicuser            apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:07Z
pgbouncer                               True    apic-mgmt-db-client-pgbouncer           apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:48Z
postgres                                True    apic-mgmt-db-client-postgres            apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:13Z
postgres-operator                       True    pgo.tls                                 apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T22:03:18Z
replicator                              True    apic-mgmt-db-client-replicator          apic-mgmt-ca          Certificate is up to date and has not expired   9d    2024-01-04T17:01:58Z

```

- The management certificate are generated by a CA Issuer **apic-mgmt-ca**
- The root ca certificate used to create the **apic-mgmt-ca** CA issuer is most probably stored in the secret **apic-mgmt-ca** 
- The secret **apic-mgmt-ca** is created using the certificate object 

A secret containing the x509 certificate has annotation that identifies the certificate an issuer that has been used to create it. I provide at the end a command to display this.

Now let's see how we can verify that the secret generated contains the right x509 ca to verify incoming TLS connections.

Certificate can be validated using openssl:   
```shell
openssl verify -CAfile ca.crt tls.crt
```


When certificate are issued by the same CA issuer, it is possible to validate the tls certificate stored on the secret against it. 
This might be useful if the ca cert has been refreshed while the secret wasn't.


### 2 - Get the secret from the **ca issuer**

This secret will contains the ca that is used for all the generated certificate.  
We store this ca locally so we can use it afterwards to verify the certificates.  
In our case the CA Issuer is **apic-mgmt-ca**

```shell
oc get issuer apic-mgmt-ca -o json | jq ' .metadata.name + " > sec: " +  .spec.ca.secretName'
```

Generate the output:
"ca-issuer > sec: ca-issuer-secret"

Or if you want to list all the ca issuer with their associated secret:

```shell
oc get issuer apic-mgmt-ca -o json | jq ' .metadata.name + " > sec: " +  .spec.ca.secretName'                                                                                                 0.534s 09:02
"apic-mgmt-ca > sec: apic-mgmt-ca"
```

As expected the secret containing the root X509 ca used by the CA Issuer is **apic-mgmt-ca**.   

### 3 - extract the ca certificate from the ca issuer secret  

```shell
oc get secret apic-mgmt-ca -o jsonpath="{.data.ca\.crt}" | base64 -D > apic-mgmt-ca.crt
```

### 4 - Validate all the related certificate with the ca

The following command, list the certificate with the ca issuer "apic-mgmt-ca" and gets the generated secret. From the secret extract the tls crt and verify it against the previously saved ca.

```shell
for i in $(oc get certs -o wide | grep apic-mgmt-ca | awk '{ print $3 }'); do echo "###$i"; oc get secrets $i -o jsonpath="{.data.tls\.crt}" | base64 -D | openssl verify -CAfile apic-mgmt-ca.crt; done
```
> print$3 gets the secret name generated by the certificate.  
> echo ###\$i is the secrets   

Output is:
```
###apic-mgmt-aa286878-postgres
stdin: OK
###apic-mgmt-aa286878-postgres-pgbouncer
stdin: OK
###apic-mgmt-ca
stdin: OK
###apic-mgmt-client
stdin: OK
###apic-mgmt-natscluster-mgmt
stdin: OK
...
```

## Solve issue

If a certificate can't be verified this means that the secret is outdated.

It happens that the root ca has been refreshed. This happens when the secret used by the CA Issuer has been deleted. This is because the ca secret is generated by a SelfSigned Issuer. If the secret is deleted, the SelfSigned Issuer will regenerate a new self signed ca that will be stored in the secret.

If you need to refresh the certificates stored in the secret and used by the microservices (f.e. management subsystem), you will then need to delete the secrets (not the ca secret !). 
Those will be regenerated by the CA issuer with the new root CA.

> In API Connect, the certificate objects and Issuers are managed by the API Connect Operator. If you delete those, they will be recreated automatically.

The following command list the secret having the CA issuer **apic-mgmt-ca"**:  

```shell
oc get secret -o json | jq '.items[] | select(.metadata.annotations."cert-manager.io/issuer-name" == "apic-mgmt-ca") | .metadata.name'
```

To delete the associated secrets just add ```| xargs oc delete secrets```.


# Useful commands

**display content of ca.crt in secret**

The ca certificate can be displayed using
```shell
oc get secret dns-secret -o jsonpath="{.data.ca\.crt}" | base64 -D | openssl x509 -text
```

**Display cert used for secret**
To display the tls secret with their associated certificate:

```shell
oc get secret -o json | jq '.items[] | select(.type == "kubernetes.io/tls" ) | .metadata.name + " > cert: " +  .metadata.annotations."cert-manager.io/certificate-name" + " > issuer: " + .metadata.annotations."cert-manager.io/issuer-name"'
```

**Display cert with secret and issuer**

```shell
oc get certs -o wide                                                                                                                                                                      0.392s 12:20
NAME              READY   SECRET                   ISSUER              STATUS                                          AGE     EXPIRATION
cert-ca-tls       True    cert-ca                  ca-issuer           Certificate is up to date and has not expired   47m     2032-01-03T10:33:22Z

```
**Validate the certificate**


```shell
for i in $(oc get certs -o wide | grep apic-mgmt-ca | awk '{ print $3 '}); do echo "###$i"; oc get secrets $i -o jsonpath="{.data.tls\.crt}" | base64 -D > $i.tls.crt; openssl verify -CAfile mgmt-ca-ca.crt $i.tls.crt; done
```
**Display secret used by CA issuer**

```shell
oc get issuer ca-issuer -o json | jq ' .metadata.name + " > sec: " +  .spec.ca.secretName'
```

Or if you want to list all the ca issuer with their associated secret:

```shell
oc get issuer -o json | jq ' .items[] | select(.spec.ca) | .metadata.name + " > sec: " +  .spec.ca.secretName'
```

