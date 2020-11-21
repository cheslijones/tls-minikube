# Purpose

This repo just demonstrates how to implement a TLS certificate in a local development k8s cluster. 

There were a few issues I came across when I was integrating the current web app I'm working on with Azure AD and a few other web APIs:

1. Needing `https://` for redirect URLs. `http://` only works with `localhost`, which circumvents cluster routing.
2. Sure, you can just provide a redirect URL of `https://{minikube ip}`, but the browser "safety" warning messages due to an invalid certificate get pretty annoying.
3. While you can implement TLS on an IP address--[it isn't a common practice](https://stackoverflow.com/questions/2043617/is-it-possible-to-have-ssl-certificate-for-ip-address-not-domain-name)--it is prohibited on non-public IPs (e.g., `192.168.x.x`), which is likely what `minikube ip` will provide.
4. **Most importantly**: *the need to encrypt data both ways when communicating with an external web API*.

# Overview

What we'll do to get this working is the following:

1. Setting up our base project using (see [Install (or Cloning)](#install)):
    - `create-react-app`
    - `django`
    - `docker`
    - `kubectl`
    - `minikube`
    - `skaffold`

2. Once installed setting up the cluster:
    - Create `k8s` deployment manifests for `./client` and `./api`
    - Setting up `ingress-nginx`
    - Setting up `skaffold`

3. Spinning up the cluster with `skaffold` to make sure the cluster is working.

4. Modifiying the host file to assign a name to our `minikube up`.

5. Installing and adding a TLS certificate to the cluster.

# Setting up the project



## <a name="install"></a> Installation (or Cloning)

### Cloning

### Step-by-Step Installation

## Spinning up our 

# Questions, Issues and Feedback

Please create an issue if you have any questions, issues running the repo, or have feedback on how I can improve this repo or correct something that is wrong.