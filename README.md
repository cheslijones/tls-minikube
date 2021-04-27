# Purpose

This repo demonstrates how to implement a TLS certificate in a local development `k8s` cluster. 

There were a few issues I came across when I was integrating the current web app I'm working on with Azure AD and a few other web APIs:

1. Needing `https://` for redirect URLs.
2. You can just provide a redirect URL of `https://<minikube-ip>`, but the browser "safety" warning messages due to an invalid certificate get pretty annoying.
3. While you can implement TLS on an IP address--[it isn't a common practice](https://stackoverflow.com/questions/2043617/is-it-possible-to-have-ssl-certificate-for-ip-address-not-domain-name)--it is prohibited on non-public IPs (e.g., `192.168.x.x`), which is likely what `minikube ip` will provide.
4. **Most importantly**: *the need to encrypt data in transit when communicating with an external web API*.

# Revisions

### 4/27/21

Rewriting the process as it has now been simplified:
- It used to involve using a `minikube --driver=` of the VM you prefer to use (`hyperkit`, `kvm`, `hyperv`, etc.). However, the prefered method has now become `minikube --drive=docker` and using `minikube tunnel`. It is more consistent across operating systems and involves fewer steps, but does require using an additional terminal for `minikube tunnel` to run.
- It used to involve modifying the operating system's `host` file to map `domain.local` to the `minikube ip`. This is no longer necessary and just uses `localhost`.
- These steps should work for macOS (Intel and M1), Windows and WSL2, and Linux.

# Credit

I was able to sort this out in large part due to [this blog post](https://itnext.io/deploying-tls-certificates-for-local-development-and-production-using-kubernetes-cert-manager-9ab46abdd569) by Sébastien Dubois. 

# Overview

What we'll do to get this working is the following:

1. Setting up our base project environment using ([go to step](#install)):
    - `react` (`create-react-app`)
    - `docker`
    - `kubectl`
    - `minikube`
    - `skaffold`

2. Once installed configuring the cluster ([go to step](#cluster)):
    - Setting up `minikube`
    - Setting up `ingress-nginx`

3. Spinning up the cluster with `skaffold` to make sure the cluster is working ([go to step](#start)).

4. Installing and adding a TLS certificate to the cluster ([go to step](#certificate)):
    - Installing and setting up `cert-manager`
    - Installing and setting up `mkcert`

# Setting up the project

## <a name="install"></a> Setting up the Base Project and Environment

### Cloning the Repo

For brevity, you'll just want to clone what is basically just a fresh `create-react-app` application, `Dockerfile.dev` and `.yaml` config files:

```
# clone the repo
git clone https://github.com/cheslijones/tls-minikube.git

# change into the directory
cd tls-minikube
```

### Installing `docker`
Please follow the directions for your operating system [here](https://docs.docker.com/get-docker/).

### Installing `kubectl`
If you are using a Docker Desktop version, it just needs to be enabled in the Docker Desktop GUI. You can also refer to installation instructions for your OS [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/#before-you-begin).

### Installing `minikube`
Please follow the directions for your operating system [here](https://minikube.sigs.k8s.io/docs/start/). 

### Installing `skaffold`
Please follow the directions for your operating system [here](https://skaffold.dev/docs/install/).

### Summary

At the end of this, you should be able to type run the following commands and get a list of the acceptable sub-commands for each command. 

```
# docker
docker run hello-world
# or
sudo docker run hello-world

# kubectl
kubectl version --client

# minikube
minikube

# skaffold
skaffold
```
If not, please refer to the documentation linked to above and follow the steps again.

## <a name="cluster"></a> Configuring the Cluster

### Creating a  `minikube` VM

It used to be preferred to use a VM driver like `kvm`, `hyperkit`, `hyperv`, etc., but using the `docker` driver tends to be more consistent across operating systems and doesn't require mapping the `minikube ip` to `domain.local` name in the `host` file.

Crete the `minikube` cluster with:
```
minikube start --vm-driver=docker

# or if you want to give the cluster a specific name to keep them separate
minikube start -p <cluster-name> --vm-driver=docker
```
Within several seconds to a minute, your `minikube` should be up and running.

### Enable `ingress-nginx` in `minikube`

In the  [`ingress-nginx` documentation](https://kubernetes.github.io/ingress-nginx/deploy/), you don't use the directions for `minikube`. Instead you use the [Docker for Mac](https://kubernetes.github.io/ingress-nginx/deploy/#docker-for-mac). It is counter-intuitive, but it does work whether you use Docker Desktop for Windows or macOS, or WSL2 and Linux:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/cloud/deploy.yaml
```

### Manifests

The repo comes with the `./ingress.yaml`, `./k8s/client.yaml`, `./client/Dockerfile.dev`, and `./skaffold.yaml` already configured. They will do the following:

- [`./ingress.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/k8s/ingress.yaml): This your `ingress-nginx` ingress controller configuration. It accepts traffic for our web application and then routes it to the appropriate running service. 
- [`./k8s/client.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/k8s/client.yaml): This is the configuration for the `create-react-app` `client` deployment.
- [`./client/Dockerfile.dev`](https://github.com/cheslijones/tls-dev-k8s/blob/main/client/Dockerfile.dev): This is the `docker` configuration for how to build the `client` image.
- [`./skaffold.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/skaffold.yaml): This configuration handles the building of the `docker` image and deploying to `minikube`.

## <a name="start"></a> Spinning up our Cluster

1. To spin the cluster up, run the following command:

    ```
    skaffold dev
    ```

    It can take around a minute for the `client` image to be built and deployed in the cluster. When it is done, you should see something like the following:

    ```
    Starting deploy...
    - ingress.networking.k8s.io/ingress-dev created
    - deployment.apps/client-deployment-dev configured
    - service/client-cluster-ip-service-dev configured
    Waiting for deployments to stabilize...
    - deployment/client-deployment-dev is ready.
    Deployments stabilized in 2.347 seconds
    Press Ctrl+C to exit
    Watching for changes...
    [client] 
    [client] > client@0.1.0 start /app
    [client] > react-scripts start
    [client] 
    [client] ℹ ｢wds｣: Project is running at http://172.17.0.4/
    [client] ℹ ｢wds｣: webpack output is served from 
    [client] ℹ ｢wds｣: Content not from webpack is served from /app/public
    [client] ℹ ｢wds｣: 404s will fallback to /
    [client] Starting the development server...
    [client] 
    [client] Compiled successfully!
    [client] 
    [client] You can now view client in the browser.
    [client] 
    [client]   Local:            http://localhost:3000
    [client]   On Your Network:  http://172.17.0.4:3000
    [client] 
    [client] Note that the development build is not optimized.
    [client] To create a production build, use npm run build.
    [client] 
    ```
2. In a separate terminal run the following command. You should be asked to provide your admin credentials:

    ```
    minikube tunnel

    $ minikube tunnel -p $(basename $PWD)
    ❗  The service ingress-nginx-controller requires privileged ports to be exposed: [80 443]
    🔑  sudo permission will be asked for it.
    🏃  Starting tunnel for service ingress-nginx-controller.
    [sudo] password for cheslijones: 
    ```

3. Now, in your browser of choice, navigate to `http://localhost`.

4. If everything is working, you should be greeted by the running `create-react-app` application.

    [<img src="https://assets.digitalocean.com/articles/66983/React_template_project.png">](https://assets.digitalocean.com/articles/66983/React_template_project.png)

Great! So we know our application is being deployed correctly into `minikube` by `skaffold`. You can press `CTRL + C` to stop the cluster from running and `skaffold` will automatically stop the cluster.

Of course, if you try to go to `https://localhost` you'll be greeted with this:

[<img src="https://helpdeskgeek.com/wp-content/pictures/2019/05/connection-is-not-private.png.webp">](https://helpdeskgeek.com/wp-content/pictures/2019/05/connection-is-not-private.png.webp)



## <a name="certificate"></a> Adding TLS certificate

Now we need to add the TLS certificate to address this error, and more importantly to encrypt our data in transit to external APIs.

### Installing `mkcert` and 

**NOTE: The instructions are slightly different for WSL2.** 

**For WSL2, you do need to use `choco` to install `mkcert` into Windows itself with `cmd` or `PowerShell` and *not* in WSL2.**

[`mkcert`](https://github.com/FiloSottile/mkcert), as the repo says, is a "... simple zero-config tool to make locally trusted development certificates with any names you'd like." It is, and does, just that.

Please refer to [the documentation](https://github.com/FiloSottile/mkcert#installation) for instructions for your OS.

### Generating Certificates

**NOTE: The instructions are slightly different for WSL2.**

**For WSL2, you need to run `mkcert -install` in `cmd` or `PowerShell` which will add the certificates to `C:\Users\<username>`. You then need to copy these into your root project in WSL2.**

Once installed, all you should need to do is the following:

```
$ mkcert -install
Created a new local CA 💥
The local CA is now installed in the system trust store! ⚡️
The local CA is now installed in the Firefox trust store (requires browser restart)! 🦊

$ mkcert localhost 127.0.0.1 ::1

Created a new certificate valid for the following names 📜
 - "localhost"
 - "127.0.0.1"
 - "::1"

The certificate is at "./localhost+2.pem" and the key at "./localhost+2-key.pem" ✅
```
You'll see that `localhost+2-key.pem` and `localhost+2.pem` have been created in your project's root directory.

### Installing `cert-manager`

Next, we need to [install `cert-manager`](https://cert-manager.io/docs/installation/kubernetes/#installing-with-regular-manifests) using one of the following commands:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

Run the following command to make sure the related Pods have been created and do not continue until they have.
```
$ kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-86548b886-wcl6r               1/1     Running   0          29s
cert-manager-cainjector-6d59c8d4f7-hws49   1/1     Running   0          29s
cert-manager-webhook-578954cdd-hn749       1/1     Running   0          29s
```
When `READY` is `1/1` for all three, you can continue.

### Adding the Certficates to `k8s` Secrets

The `.pem` files that were created need to be added to our `k8s` cluster secrets:

```
kubectl create secret tls tls-localhost-dev --key=localhost+2-key.pem --cert=localhost+2.pem
```
You should see the following if successful:
```
secret/tls-localhost-dev created
```
### `.pem` Warning

**DO NOT COMMIT THESE. Either add `*.pem` to `.gitignore` or just delete them as they are not needed after the previous step.**

### The `issuer.yaml` File

We need to apply the `issuer.yaml` manifest to our cluster now so that the certificate can be aquired and added to all traffic:
```
kubectl apply -f k8s/issuer.yaml
```
If successful, you should see:

```
issuer.cert-manager.io/letsencrypt-dev-issuer created
certificate.cert-manager.io/letsencrypt-dev-certificate created
```

### Finally....

Run `skaffold dev` again.

Navigate to `localhost` and if everything was done correctly, you should automatically be directed to `https://localhost` and have no warning messages from your browser.

# Questions, Issues and Feedback

Please create an issue if you have any questions, issues running the repo, or have feedback on how I can improve this repo or correct something that is wrong. I'm always looking for ways to improve.