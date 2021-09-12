# SSL Configuration for Rancher with Let's Encrypt

This document explains how to configure a SSL Certificate provided by [Let's Encrypt](https://letsencrypt.org) for a Kubernetes Namespace managed in [Rancher](https://rancher.com).

Note: You can configure a Certificate for your whole cluster but I prefer to use SSL certificates at Namespace's level so that I can have different certificates depending on the Namespace that I am using as my prod-Namespaces are linked to productive (meaning real) domain names for productive applicaitons.

The work is based on the article https://jmrobles.medium.com/free-ssl-certificate-for-your-kubernetes-cluster-with-rancher-2cf6559adeba

## Prerequisites
A running Kubernetes cluster. For the purpose of this work, we will use a cluster managed with Rancher.

## Let’s go, Let’s Encrypt
[Let's Encrypt](https://letsencrypt.org) is a certification authority (CA) created in 2016 by the Electronic Frontier Foundation and Mozilla Foundation. Its target is that every Internet communication be ciphered using TLS protocol.

The free Let’s Encrypt certificate has a validity of 3 months and has quite a tedious management.

Fortunately “[certbot](https://certbot.eff.org)”, the CLI Let’s Encrypt tool, allows to renew the certificates automatically. You can simply just put it in a CRON job.

## Introducing cert-manager
The Kubernetes way to get “Let’s Encrypt”-certicates is by using “[cert-manager](https://cert-manager.io/docs/)”.

![cert-manager overview](https://cert-manager.io/images/high-level-overview.svg)

The great feature for “cert-manager” is: _automate certificate management_, meaning "forget about certificates renewal".

In addition to Let’s Encrypt certificates (a.k.a ACME certs), cert-manager can emit self-signed certificates and manages others certificates issued by 3th parties CA.

## Installation
We will be using installation through manifests. 

In case of our Rancher’s cluster, just apply the following manifest:
> kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml

Note: the last stable version when I created this project is **_1.5.3_**, the latest version available should be used.

The installation process creates several new custom resource definitions (CRD) required.

The main CRDs created are:

1. *Issuer*: Let’s encrypt account setup to use for certificate request. The issuer is valid for a specific Namespace.
2. *ClusterIssuer*: It’s an Issuer valid for all the cluster, not just for a specific Namespace.
3. *Certificate*: The Let’s Encrypt certificate emitted after the issuer made the request, and Let’s Encrypt generates and gives the certificate.

### The Let’s Encrypt certificate issuer process
In the certificate issuer process, the CA needs to verify that you are the owner/manager of the domain/subdomain to use in the certificate.

There are two methods:

1. *TXT DNS Record challenge*, where we create a DNS entry for our domain with a specific token given by the CA.
2. *HTTP challenge*, where the CA tool generates a specific token file under our domain and the CA server tries to request it. If successful, the domain is verified and CA emits the new certificate. It’s the preferred method and the one that we’ll use too.

## Setup
We need to create a Issuer for our Namespace. In the example, we use the Namespace called "ns1".

Let’s encrypt has a sandbox environment. 
We first create a sandbox issuer due to the API usage limitation that the real environment has, and if the setup is not OK, Let’s Encrypt may ban us from using it for while.

We use the following YAML: [sandbox-le-issuer.yaml](https://raw.githubusercontent.com/dwamara/rancher-ssl-letsencrypt/main/sandbox-le-issuer.yaml)

As you can see in the YAML, the kind of object is *Issuer*, the namespace is “ns1” and the server is for the staging (sandbox) endpoint.

In the "solvers" section, we specify that we use the _http challenge_ and the ingress is of class _nginx_ class.

Note: The nginx ingress is installed by default with Rancher.

We will need to replace “_your_email_address_” with a real email as it is important to receive alerts about SSL certs close to expire.

Now we apply it
> kubectl apply -f sandbox-le-issuer.yaml

If everything went fine, we will obtain the following output
> issuer.cert-manager.io "letsencrypt-staging" created

We can then proceed on creating the production issuer by using following YAML manifest: [prod-le-issuer.yaml](https://raw.githubusercontent.com/dwamara/rancher-ssl-letsencrypt/main/prod-le-issuer.yaml)

## Test it!
We can’t test our issuer without an app/service.

We will use [kuard](https://github.com/kubernetes-up-and-running/kuard/blob/master/README.md) as our “hello-ssl-world” app. You need a domain that points to the IP of your load balancer.

We create a deployment and service to use it using following YAML manifest: [deployment-service-kuard.yaml](https://raw.githubusercontent.com/dwamara/rancher-ssl-letsencrypt/main/deployment-service-kuard.yaml)

We apply it
> kubectl apply -f deployment-service-kuard.yaml

All that remains is to create the corresponding ingress.

To use Let’s Encrypt certificates with our ingress, we just have to add the annotation: _cert-manager.io/issuer: "<issuer>"_.

In first place, we use the sandbox issuer, and if it goes well, we re-apply it with the production issuer.

Don’t forget to replace “__your_domain_” with your real domain!

We apply it
>kubectl apply -f ingress.yaml

After that, cert-manager creates a temporary pod to request the certificate. We can check the status with the following kubectl command

> kubectl get certificate -n ns1

Once the certificate is ready, we obtain an output like as the following:

> NAME             READY   SECRET           AGE
>
> quickstart-tls   True    quickstart-tls   2m

Everything seems fine, now we can emit the real certificate.

Just replace in the issuer.yaml file, _cert-manager.io/issuer: “letsencrypt-staging”_ by _cert-manager.io/issuer: “letsencrypt-prod”_, and re-apply it.
> kubectl apply -f ingress.yaml

This time the real certificate can take up to several minutes to be ready.

We can get more details using the next describe command
> kubectl describe certificate quickstart-tls -n ns1

In the last lines, you can see the status and all the events.
[...]
Reason:                CertIssued
    Status:                True
    Type:                  Ready
Events:
  Type     Reason          Age                From          Message
  ----     ------          ----               ----          -------
  Normal   CreateOrder     9m                 cert-manager  Created new ACME order, attempting validation...
  Normal   DomainVerified  8m                 cert-manager  Domain "yourdomain.com" verified with "http-01" validation
  Normal   IssueCert       8m                 cert-manager  Issuing certificate...
  Normal   CertObtained    7m                 cert-manager  Obtained certificate from ACME server
  Normal   CertIssued      7m                 cert-manager  Certificate issued Successfully

But the real test is to go to “http://yourdomain.com” and check if you are redirected to “https://yourdomain.com” with a valid free Let’s Encrypt certificate.

Kuard is up and secure with SSL and that's it. You can replace Kuard with your real service needing to use SSL.
