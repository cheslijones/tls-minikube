# Purpose

This repo just demonstrates how to implement a TLS certificate in a local development k8s cluster. 

There were a few issues I came across when I was integrating the current web app I'm working on with Azure AD and a few other web APIs:

1. Needing `https://` for redirect URLs. `http://` only works with `localhost`, which circumvents cluster routing.
2. Sure, you can just provide a redirect URL of `https://{minikube ip}`, but the browser "safety" warning messages due to an invalid certificate get pretty annoying.
3. While you can implement TLS on an IP address--[it isn't a common practice](https://stackoverflow.com/questions/2043617/is-it-possible-to-have-ssl-certificate-for-ip-address-not-domain-name)--it is prohibited on non-public IPs (e.g., `192.168.x.x`), which is likely what `minikube ip` will provide.
4. **Most importantly**: *the need to encrypt data both ways when communicating with an external web API*.

# Credit

I was able to sort this out in large part due to [this blog post](https://itnext.io/deploying-tls-certificates-for-local-development-and-production-using-kubernetes-cert-manager-9ab46abdd569) by Sébastien Dubois. 

# Overview

What we'll do to get this working is the following:

1. Setting up our base project using ([go to step](#install)):
    - `react`
    - `docker`
    - `kubectl`
    - `minikube`
    - `skaffold`

2. Once installed setting up the cluster ([go to step](#cluster)):
    - Create `k8s` deployment manifests for the `./client` service
    - Setting up `ingress-nginx`
    - Setting up `skaffold`

3. Spinning up the cluster with `skaffold` to make sure the cluster is working ([go to step](#start)).

4. Modifiying the host file to assign a domain name to our `minikube ip` ([go to step](#host)).

5. Installing and adding a TLS certificate to the cluster ([go to step](#certificate)):
    - Installing and setting up `cert-manager`
    - Installing and setting up `mkcert`

# Setting up the project

*Note*: I only have experience successfully doing this in Linux and macOS. I've spent countless hours trying to get the `docker`, `kubectl`, `minikube`, and `skaffold` stack working in WSL2 on Windows with no success. I've never tried on Windows natively or with `choco`. I'll provide links to installs for all three OSes, but I'm not able to provide much support when it comes to Windows.

## <a name="install"></a> Installation (or Cloning)

### Cloning

### Step-by-Step Installation

## <a name="cluster"></a> Setting up the Cluster

## <a name="start"></a> Spinning up our Cluster

## <a name="host"></a> Modifying the host file

## <a name="certificate"></a> Adding TLS certificate

# Questions, Issues and Feedback

Please create an issue if you have any questions, issues running the repo, or have feedback on how I can improve this repo or correct something that is wrong.