# SSL Configuration for Rancher with Let's Encrypt

This document explains how to configure a SSL Certificate with Letsencrypt for a Kubernetes namespace managed in Rancher.
I prefer to use SSL certificates at namespace's level so that I can have different certificates depending on the namespace that I am using as my namespaces are link to production domain names.

The work is based on the article https://jmrobles.medium.com/free-ssl-certificate-for-your-kubernetes-cluster-with-rancher-2cf6559adeba
