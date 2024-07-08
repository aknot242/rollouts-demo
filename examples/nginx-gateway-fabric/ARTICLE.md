# Powering Progressive Deployment in Kubernetes

Recently, the Kubernetes community celebrated the platform's 10th year in existence. Inarguably, it has come a long way since its early days at Google and has established itself as the de-facto platform for delivering cloud-native modern applications.

In your application modernization journey, you have no doubt made use of containerization technology, automated your build and deployment pipelines with CI/CD tooling of GitOps solutions to handle your infrastructure in code as you already do your applications. With all the capabilities in Kubernetes, augmented by the rich ecosystem of related open-source tools and contingents, there are still inherent complexities that remain unsolved for you. For instance, you may find it relatively easy to stand up your first production cluster, deploy your applications to it, and start accepting live traffic to it. But, how is it performing? What are users experiencing when they consume your applications and APIs? Do you have enough capacity for organic growth or seasonal surges related to your apps? When you inevitably need to update your application, how do you do so while minimizing disruption?

## Deployment Patterns

Fortunately, as software development itself has design patterns - repeatable blueprints for successfully and sustainably building applications, there also exist patterns for the deployment and operation of them. You may have heard of "Blue-green", "Canary deployments", and "A/B Testing. These are examples of deployment patterns; designed to route and shape application traffic over time. While these deployment patterns seem promising, until the last several years, you had been left to your own devices (no pun intended) to implement them. However, proxies such as NGINX, Envoy and service meshes such as Istio have been able to assist in the execution of these patterns as application rollout strategies. While these solutions fit neatly into a Kubernetes environment and can route and process traffic into the applications residing in the cluster, many of them have their own specific behaviors and differences in configuration.

Fortunately, the CNCF recognized this flourish of specific implementations to perform these functions natively. As a result, the CNCF's [SIG-NETWORK](https://github.com/kubernetes/community/tree/master/sig-network) community released a common specification for modern application delivery services in Kubernetes called the Gateway API. This new specification has been supported by a number of projects and vendors, most notably the NGINX Gateway Fabric for Kubernetes. At KubeCon 2023, F5 NGINX unveiled 1.0 of NGF, supporting traffic splitting patterns such as Canary and Blue-green intrinsically.

While the capability to execute these application rollouts in a vendor-neutral way has been a desire of the community, what about the observability considerations mentioned earlier? How can we be assured that our applications are working after deploying new versions? What if we want to orchestrate the progressive introduction of application changes rather than a "big-bang" deployment? And if the deployment isn't successful, how might we know if we need to roll back?

## Progressive Delivery

Progressive delivery has emerged as a preferred approach for modern applications for a number of reasons. It can be a means to orchestrate a gradual feature rollout for an application. It can help reduce risk by deploying changes only to a subset or test group of users first. You can also use progressive delivery to gather early feedback of a new application deployment and other use cases. This isn't meant to be an exhaustive list of use cases, but merely an introduction.

## A Solution

NGINX is the most widely used web server and application proxy in the world. It has been very popular in the Kubernetes world, as the de-facto Ingress controller for a number of years. With the aforementioned NGINX Gateway Fabric packaging, the ubiquitous API Gateway, load balancer and security enforcement solution has minted a new lease on seamless integration with Kubernetes.

How do we orchestrate progressive delivery? There is more we need...

Our friends at the Argo Project (famous for their GitOps platform, ArgoCD) released Argo Rollouts back in 2019, focused on this progressive delivery problem. Argo Rollouts enables the orchestration of Canary deployments, Blue-green deployments, and experimentation with traffic splitting. However, the ability to execute the traffic shaping portion of application rollouts with popular proxies and service meshes required in-tree source code which added complexity, and slowed down contributions. As you might imagine, the amount of effort required to develop and maintain all these vendor and implementation-specific extensions has been a chore to say the least.

Fortunately, on April 5, 2023, the Argo Project [announced](https://blog.argoproj.io/argo-rollouts-1-5-release-candidate-2bd93720e411) that a plugin system for Argo Rollouts had been developed, enabling future extensibility of the product without friction to contributing described above. This was a welcome addition, and would set up the Argo Project for yet another important innovation...

With much celebration, the Argo Project [announced](https://blog.argoproj.io/argo-rollouts-now-supports-version-1-0-of-the-kubernetes-gateway-api-acc429729e42) on June 20, 2024 that Argo Rollouts now supports ingresses, gateways and service meshes that implement the Kubernetes Gateway API for progressive delivery. Unsurprisingly, NGINX Gateway Fabric, powered by the most popular proxy in the world, is one of the [supported providers](https://gateway-api.sigs.k8s.io/implementations/).

## Better Together

There are a number of benefits of using NGINX Gateway Fabric with Argo Rollouts:

- **Enhanced Traffic Management:** NGINX Gateway Fabric provides robust traffic management capabilities, allowing dynamic routing adjustments during deployments.
- **Increased Availability and Reliability:** This combination ensures high availability and reliability through intelligent traffic handling and failover mechanisms.
- **Improved User Experience:** Progressive delivery with NGINX Gateway Fabric and Argo Rollouts minimizes downtime and reduces the risk of introducing bugs or issues to end-users.
- **Efficient Rollback Mechanisms:** Ease of rolling back changes in case of issues, ensuring quick recovery and minimal impact on users.

## The Details

Without further ado, let's set this up for a test drive. You are going to need a Kubernetes cluster, ideally [version 1.25 or greater](https://github.com/nginxinc/nginx-gateway-fabric#technical-specifications) so we can take advantage of features in the latest version of the Gateway API. While the installation could be performed entirely by script or using something like Argo CD, we will be performing each step manually for learning purposes.

Here's the overall architecture of the demo environment:

TODO: Diagram here

> Note: It is inadvisable to perform the demo steps in a production Kubernetes cluster without prior validation in a lesser environment.

Local tools needed:

- [Helm](https://helm.sh/docs/intro/install/)
- [kubeconfig](https://kubernetes.io/docs/tasks/tools/#kubectl) (set up with a config valid for your test cluster)
- [git](https://git-scm.com/downloads)

We will begin by installing NGINX Gateway Fabric. We will be using the Plus edition in order to take advantage of its extended metrics and seamless configuration reload capabilities.

### Installation

1. Create the namespace for NGF:

```shell
kubectl create namespace nginx-gateway
```

1. Create a Secret in order to pull the NGF container from the F5 private registry. The secret is based on the contents of the trial JWT from MyF5. If you do not have a trial JWT, you can request one [here](https://www.f5.com/trials/free-trial-connectivity-stack-kubernetes).

```shell
kubectl create secret docker-registry nginx-plus-registry-secret --docker-server=private-registry.nginx.com --docker-username=`cat the_full_path_to_you_jwt_here` --docker-password=none -n nginx-gateway
```

1. The Kubernetes Gateway API isn't included in clusters by default. We need to install its Custom Resource Definitions (CRDs) in order to use it:

```shell
kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.3.0" | kubectl apply -f -
```

1. Next, we will install NGF via its Helm chart:

```shell
helm install ngf oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric --set nginx.image.repository=private-registry.nginx.com/nginx-gateway-fabric/nginx-plus --set nginx.plus=true --set serviceAccount.imagePullSecret=nginx-plus-registry-secret -n nginx-gateway --version 1.3.0
```

1. Run the following command to wait until NGF has been verified as deployed:

```shell
kubectl wait --timeout=5m -n nginx-gateway deployment/ngf-nginx-gateway-fabric --for=condition=Available
```

1. We will be using Prometheus metrics to inform Argo Rollouts of the state of our application's health during rollouts. Add the Prometheus community helm chart, then update the Helm repo:

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

1. Install Prometheus to your cluster in its own namespace:

```shell
helm install prometheus prometheus-community/prometheus -n prometheus --create-namespace --set server.global.scrape_interval=15s
```

1. Create a namespace for Argo Rollouts, and install it using manifests from the project's GitHub repo:

```shell
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

1. Install the Argo Rollouts CLI using the [instructions](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation) for your client platform.

1. Install the Gateway API Plugin for Argo Rollouts:

```shell
kubectl apply -f gateway-plugin.yml -n argo-rollouts
```

1. Restart the Argo Rollouts controller so that it detects the presence of the plugin.

```shell
kubectl rollout restart deployment -n argo-rollouts argo-rollouts
```

1. Then, check the controller logs. You should see a line for loading the plugin:

```shell
time="XXX" level=info msg="Downloading plugin argoproj-labs/gatewayAPI from: https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.2.0/gateway-api-plugin-linux-amd64"
time="YYY" level=info msg="Download complete, it took 7.792426599s"
```

### Demo Setup

1. With git, clone the demo repo:

```shell
git clone <TBD!!!!!!!!!!!!!!!!>
```

1. Change directory to your repo clone's `nginx-gateway-fabric` directory:

```shell
cd rollouts-demo/examples/nginx-gateway-fabric
```

1. Install the Gateway resource:

```shell
kubectl apply -f gateway.yaml
```

## Conclusion

- Summarize the benefits of adopting progressive delivery using NGINX Gateway Fabric and Argo Rollouts.
- Encourage readers to consider these technologies for their Kubernetes deployments to achieve smoother deployments, improved user satisfaction, and faster iteration cycles.

By framing the problem statement around the challenges of modern application deployment and transitioning into the capabilities of NGINX Gateway Fabric and Argo Rollouts for progressive delivery, you can effectively guide your readers towards understanding the benefits and implementation considerations of these technologies.