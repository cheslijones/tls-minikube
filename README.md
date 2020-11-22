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

4. Modifiying the host file to assign a domain name to our `minikube ip` ([go to step](#host)).

5. Installing and adding a TLS certificate to the cluster ([go to step](#certificate)):
    - Installing and setting up `cert-manager`
    - Installing and setting up `mkcert`

# Setting up the project

*Note*: I only have experience successfully doing this in Linux and macOS. I've spent dozens of hours trying to get the `docker`, `kubectl`, `minikube`, and `skaffold` stack working in WSL2 on Windows with no success. I've never tried on Windows natively or with `choco`. I'll provide links to installs for all three OSes, but I'm not able to provide much support when it comes to Windows. 

## <a name="install"></a> Setting up the Base Project and Environment

### Cloning the Repo

For brevity, you'll just want to clone what is basically just a fresh `create-react-app` application, `Dockerfile.dev` and `.yaml` config files:

```
# clone the repo
git clone https://github.com/cheslijones/tls-dev-k8s.git

# change into the directory
cd tls-dev-k8s
```

### Installing `docker`
Please follow the directions for your operating system [here](https://docs.docker.com/get-docker/). You'll like also want to 

### Installing `kubectl`
If you are using a Docker Desktop version, it just needs to be enabled in the Docker Desktop GUI. You can also refer to installation instructions for your OS [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/#before-you-begin).

### Installing `minikube`
Please follow the directions for your operating system [here](https://minikube.sigs.k8s.io/docs/start/). Step 1 is sufficient (up to `minikube start`).

### Installing `skaffold`
Please follow the directions for your operating system [here](https://skaffold.dev/docs/install/).

### Summary
I apologize for simply linking to the documentation. I would be basically retyping it out here and then have to maintain it when/if the directions change in the future. It is more efficient to refer to appropriate section of the current documentation.

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

If you stopped before running `minikube start`, good. If not, you want to run `minikube delete`.

`minikube start` creates a virtual machine (VM) and eventually we'll use `skaffold` to deploy our `k8s` cluster to it. If you do not have a hypervisor installed, or do not specify one when you run `minikube start`, it defaults to `docker`. I've never been able to figure out how to access a cluster from the browser that uses `docker` as the `--vm-driver`, so I'd advise against using it. 

For a list of acceptable drivers please check [here](https://minikube.sigs.k8s.io/docs/drivers/). In most cases, you'll need to install or enable the hypervisor. on macOS I use `hyperkit` and on Linux I use `kvm`. Virtual Box, VMware, etc. can be used instead.

Once you have a hypervisor installed, we are set to create the `minikube` VM with:
```
# replace <vm-driver> with your driver
minikube start --vm-driver=<vm-driver>

# or if you want to give the VM a specific name to keep clusters separate
minikube start -p <vm-name> --vm-driver=<vm-driver>
```
Within several seconds to a minute, your VM should be up and running.

### Enable `ingress-nginx` in `minikube`

Per [the documentation](https://kubernetes.github.io/ingress-nginx/deploy/#minikube), you only need to run this command:

```
minikube addons enable ingress

# or if you gave your vm a specific name
minikube -p <vm-name> addons enable ingress
```

Cool, you are now set with `minikube` running.

### Manifests

The repo comes with the `./ingress-ip.yaml`, `./client/deployment.yaml`, `./client/Dockerfile.dev`, and `./skaffold.yaml` already configured. They will do the following:

- [`./ingress.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/k8s/ingress.yaml): This your `ingress-nginx` ingress controller configuration. It accepts traffic for our web application and then routes it to the appropriate running service. For example, traffic going to our root route (`http://example.com/`) will be routed to the running `client` app.
- [`./client/deployment.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/client/deployment.yaml): This is the configuration for the `create-react-app` `client` service. This gives the `client` service an IP in the cluster so when `ingress-nginx` receives traffic heading to `/`, it knows what Deployment to send the traffic to.
- [`./skaffold.yaml`](https://github.com/cheslijones/tls-dev-k8s/blob/main/skaffold.yaml): This configuration will be how we spin the cluster up and the services. It does a number of cool things like checking the `docker` running in `minikube` to see if our services have been built into images, and if not, it will run our `Dockerfile.dev` to build the image. In addition, it will run and destroy the `deployment.yaml` when start and stop the cluster. Pretty cool and time saving stuff.

### Getting the IP of the Cluster

We should be almost set. Now we need to know where we can access the running cluster from the browser when we finally spin it up with `skaffold`. To get the IP run this command and keep note of it because we'll be using it in later steps:

```
minikube ip

# or if you gave your vm a specific name
minikube -p <vm-name> ip
```

Mine returns:

```
192.168.64.8
```

## <a name="start"></a> Spinning up our Cluster

Ok, the moment of truth.... we should have our cluster configs all set. To spin the cluster up, run the following command:

```
skaffold dev
```

It can take around a minute for the `client` image to be built and deployed in the cluster. When it is done, you should see something like the following:

```
Starting deploy...
 - Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
 - ingress.networking.k8s.io/ingress-service-dev created
 - deployment.apps/client-deployment-dev created
 - service/client-cluster-ip-service-dev created
Waiting for deployments to stabilize...
 - deployment/client-deployment-dev is ready.
Deployments stabilized in 4.476523883s
Press Ctrl+C to exit
Watching for changes...
[client] 
[client] > client@0.1.0 start /app
[client] > react-scripts start
[client] 
[client] ℹ ｢wds｣: Project is running at http://172.17.0.3/
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
[client]   On Your Network:  http://172.17.0.3:3000
[client] 
[client] Note that the development build is not optimized.
[client] To create a production build, use npm run build.
[client] 
```

Now, in your browser of choice, navigate to the IP address provided by `minikube ip`.

If everything is working, you should be greeted by the running `create-react-app` application.

[<img src="https://assets.digitalocean.com/articles/66983/React_template_project.png">](https://assets.digitalocean.com/articles/66983/React_template_project.png)

## <a name="host"></a> Modifying the host file



## <a name="certificate"></a> Adding TLS certificate

# Questions, Issues and Feedback

Please create an issue if you have any questions, issues running the repo, or have feedback on how I can improve this repo or correct something that is wrong.