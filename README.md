# Getting started with Knative and Istio

# Introduction
Serverless and Service Mesh are two popular cloud-native technologies that customers are exploring how to create value from. [Knative](https://knative.dev/docs/) and [Istio](https://istio.io/) respectively are the prominent open source communities surrounding these two spaces. As we dive deeper into each of these solutions with our customers, the question often comes up on the intersection between the two popular technologies and how they can compliment eachother. Can we leverage Istio to secure, observe, and expose our Knative serverless applications? 

When exploring this area you will find that while there is a lot of great information and concepts floating around, it is more difficult in practice to implement even in development without some trial-and-error. (But isn't that the fun in the process?!) The aim of this blog is to assist with the following by providing a guide to walk through the installation of both technologies and exploring the integration between them

# Purpose/Outcomes
At the end of part 1 of this tutorial we will have completed the following:
- Installed knative-serving
- Installed Istio
- Configured knative and Istio together
- Deployed our first serverless knative app
- Triggered our serverless app through the service mesh externally and internally (using default PERMISSIVE mTLS)
- Set up a knative revision
- Split traffic between two revisions of a knative service

In part 2 of this tutorial we will expand further on this with a second example:
- Setting up STRICT mtls for knative-serving
- Deploy a second serverless app `httpbin`
- Confirm mTLS is enforced
- Explore using Istio `AuthorizationPolicy` to further secure our services

### OpenShift Installations
For OpenShift users, this guide will also provide the additional configuration components necessary to walk through the steps on an OpenShift cluster. Additional configuration has been summarized into a single section to make the tutorial easier to follow along for both OpenShift and non-OpenShift users

[See this link](https://istio.io/latest/docs/setup/platform-setup/openshift/) from the Istio documentation for more detail on the additional configuration required to run knative + istio on OpenShift. For the purposes of this repo, all general-purpose commands are led with `kubectl` and all additional OpenShift specific instructions are commands led with `oc` to provide more clarity.

## Table of Contents
- [Tutorial #1: Deploying your first Knative Service on Istio](https://github.com/ably77/knative-istio-tutorial#tutorial-1-deploying-your-first-knative-service-on-istio)
- [Next Steps: Setting up STRICT mtls for knative-serving](https://github.com/ably77/knative-istio-tutorial#next-steps---setting-up-strict-mtls-for-knative-serving)
- [Additional Next Steps: Install Gloo Mesh](https://github.com/ably77/knative-istio-tutorial#additional-next-steps---install-gloo-mesh)

## Tutorial #1: Deploying your first Knative Service on Istio

### First step - install Istio!
The first step that we should do is install Istio on our cluster. The commands below will guide us through both default and OpenShift install processes

#### Default Istio Installation
Default Istio install will leverage the [default](https://istio.io/latest/docs/setup/additional-setup/config-profiles/) Istio configuration profile
```
kubectl create ns istio-system
istioctl install --set profile=default -y
```

#### Istio Installation for OpenShift users
```
oc new-project istio-system
oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-system
istioctl install --set profile=openshift -y
```
**Note:** As we can see, there is extra configuration necessary for OpenShift Istio deployments because of the use of Security Context Constraints (SCCs) in OpenShift. Istio components require the use of UID 1337 which is reserved for the sidecar proxy component. For this reason in this tutorial we will need to allow the `anyuid` SCC to be used anywhere Istio is used, rather than the default `restricted` SCC.

### Install knative-serving
For this tutorial we will be deploying knative using the [YAML method](https://knative.dev/docs/admin/install/serving/install-serving-with-yaml/). In the future we can explore using the knative operator in order to deploy and manage knative components

Create knative-serving namespace:
```
kubectl create ns knative-serving
```

Enable istio sidecar container injection on `knative-serving` system namespace.
```
kubectl label namespace knative-serving istio-injection=enabled
```

### Additional Prep for OpenShift deployments
This section will cover all of the additional prep necessary for this tutorial to work on OpenShift in detail

#### NetworkAttachmentDefinitions and OpenShift
For OpenShift users, the istio-cni NetworkAttachment must be added to **every** namespace where we plan to deploy istio-enabled services. This is because CNI on OpenShift is managed by Multus, and it requires a NetworkAttachmentDefinition to be present in the application namespace in order to invoke the istio-cni plugin. The [Istio CNI plugin](https://istio.io/latest/docs/setup/additional-setup/cni/) is a replacement for the istio-init container that performs the same networking functionality but without requiring Istio users to enable elevated Kubernetes RBAC permissions.

Deploy NetworkAttachmentDefinition to the knative-serving namespace:
```
cat <<EOF | oc -n knative-serving create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: istio-cni
EOF
```

Deploy NetworkAttachmentDefinition to the default namespace (where we will be deploying our knative service later):
```
cat <<EOF | oc -n default create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: istio-cni
EOF
```

#### Leveraging the anyuid SCC
For OpenShift users, the `anyuid` policy must be added to every namespace where we plan to deploy istio-enabled services as well. This is because the Istio sidecar injected into each application pod runs with user ID 1337, which is not allowed by the default Security Context Constraint (SCC) in OpenShift. [see link](https://istio.io/latest/docs/setup/platform-setup/openshift/#security-context-constraints-for-application-sidecars)


Set the `anyuid` policy on the `default` and `knative-serving` namespaces:
```
oc adm policy add-scc-to-group anyuid system:serviceaccounts:default
oc adm policy add-scc-to-group anyuid system:serviceaccounts:knative-serving
```

Thats it! These two steps adding the `NetworkAttachmentDefinition` and assigning the `anyuid` SCC to each istio enabled namespace is all you need to get started with Istio and Knative on OpenShift!


### Deploy knative-serving components
First deploy the knative CRDs
```
kubectl create -f https://github.com/knative/serving/releases/download/v0.24.0/serving-crds.yaml
```

Next lets deploy the knative-serving components
```
kubectl create -f https://github.com/knative/serving/releases/download/v0.24.0/serving-core.yaml
```

Check to see that the components have been deployed
```
kubectl get pods -n knative-serving
```

Output should look similar to below, as you can see the istio sidecar has been injected into each component in the knative-serving namespace, denoted by the 2/2 in each pod. If you want to drill down you can do a `kubectl describe` on the pod to see more details.
```
% kubectl get pods -n knative-serving
NAME                                     READY   STATUS    RESTARTS   AGE
activator-dfc4f7578-62c9f                2/2     Running   0          4m37s
autoscaler-756797655b-tz4c8              2/2     Running   0          2m31s
controller-7bccdf6fdb-kx5gj              2/2     Running   0          2m53s
domain-mapping-65fd554865-94kkg          2/2     Running   0          2m3s
domainmapping-webhook-7ff8f59965-ljqb6   2/2     Running   0          88s
webhook-568c4d697-hzh55                  2/2     Running   0          53s
```

### Deploy the knative istio controller to integrate knative with istio
By default, the `net-istio` controller example in the Knative docs creates a shared ingress Gateway named `knative-ingress-gateway` located in the `knative-serving` namespace to serve all incoming traffic to Knative. By default, the `net-istio` controller integrates with the default Istio gateway `istio-ingressgateway` in the `istio-system` namespace as its underlying gateway. This can be configured to [use a non-default local gateway](https://knative.dev/docs/admin/install/installing-istio/#updating-the-config-istio-configmap-to-use-a-non-default-local-gateway) but for the purposes of this tutorial we will keep the default.

Deploy the knative-istio-controller to integrate istio and knative
```
kubectl apply -f https://github.com/knative/net-istio/releases/download/v0.24.0/net-istio.yaml
```

#### Validate
To validate, you can see that the Istio sidecar was injected in the net-istio-webhook pod and would be showing as `2/2` 
```
% kubectl get pods -n knative-serving
NAME                                     READY   STATUS    RESTARTS   AGE
activator-dfc4f7578-62c9f                2/2     Running   0          7m32s
autoscaler-756797655b-tz4c8              2/2     Running   0          5m26s
controller-7bccdf6fdb-kx5gj              2/2     Running   0          5m48s
domain-mapping-65fd554865-94kkg          2/2     Running   0          4m58s
domainmapping-webhook-7ff8f59965-ljqb6   2/2     Running   0          4m23s
net-istio-controller-799fb59fbf-xtmwh    1/1     Running   0          59s
net-istio-webhook-5d97d48d5b-g7n6p       2/2     Running   0          59s
webhook-568c4d697-hzh55                  2/2     Running   0          3m48s
```

If you do a describe on the net-istio-webhook pod you can see more detail in the events
```
% k describe pod net-istio-webhook-5d97d48d5b-mwkgv -n knative-serving
<...>
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       12m   default-scheduler  Successfully assigned knative-serving/net-istio-webhook-5d97d48d5b-mwkgv to crc-txps5-master-0
  Normal  AddedInterface  12m   multus             Add eth0 [10.217.0.78/23] from openshift-sdn
  Normal  AddedInterface  12m   multus             Add net1 [] from knative-serving/istio-cni
  Normal  Pulled          12m   kubelet            Container image "docker.io/istio/proxyv2:1.10.2" already present on machine
  Normal  Created         12m   kubelet            Created container istio-validation
  Normal  Started         12m   kubelet            Started container istio-validation
  Normal  Pulled          12m   kubelet            Container image "gcr.io/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:49bf045db42aa0bfe124e9c5a5c511595e8fc27bf3a77e3900d9dec2b350173c" already present on machine
  Normal  Created         12m   kubelet            Created container webhook
  Normal  Started         12m   kubelet            Started container webhook
  Normal  Pulled          12m   kubelet            Container image "docker.io/istio/proxyv2:1.10.2" already present on machine
  Normal  Created         12m   kubelet            Created container istio-proxy
  Normal  Started         12m   kubelet            Started container istio-proxy
```

### Deploying knative-serving apps
For this demo we will be following the [Deploying your first Knative Service](https://knative.dev/docs/getting-started/first-service/) guide from the official knative docs.

#### Enable sidecar injection on default namespace
In order for Istio to recognize workloads, it is necessary to label the namespace (in this case the default namespace) with `istio-injection=enabled`
```
kubectl label namespace default istio-injection=enabled
```

#### Create first knative hello-world service
To deploy our first knative service, run the following command
```
kn service -n default create hello \
--image gcr.io/knative-samples/helloworld-go \
--port 80 \
--env TARGET=World \
--revision-name=world 
```

The output should look similar to below
```
% kn service -n default create hello \
--image gcr.io/knative-samples/helloworld-go \
--port 80 \
--env TARGET=World \
--revision-name=world
Creating service 'hello' in namespace 'default':

  0.037s The Route is still working to reflect the latest desired specification.
  0.094s Configuration "hello" is waiting for a Revision to become ready.
 45.769s ...
 45.861s Ingress has not yet been reconciled.
 46.076s Waiting for load balancer to be ready
 46.281s Ready to serve.

Service 'hello' created to latest revision 'hello-world' is available at URL:
http://hello.default.example.com
```

#### Explore the deployment

List available `kn` services
```
kn service list
```

Example output
```
% kn service list
NAME    URL                                LATEST        AGE   CONDITIONS   READY   REASON
hello   http://hello.default.example.com   hello-world   38m   3 OK / 3     True    
```

#### Trigger our knative service
There are multiple methods to triggering our knative-service through the istio-ingressgateway. Below will give a few examples

##### Trigger knative service directly through external LB
Get our istio-ingressgateway IP
```
kubectl get svc -n istio-system
```

The output should look similar to below. The value we are looking for is the `EXTERNAL-IP` of the `istio-ingressgateway` service. In our case this is the `104.154.165.58`
```
% kubectl get svc -n istio-system
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                      AGE
istio-ingressgateway    LoadBalancer   10.35.253.134   104.154.165.58   15021:30144/TCP,80:30485/TCP,443:31656/TCP   45m
istiod                  ClusterIP      10.35.254.69    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP        45m
knative-local-gateway   ClusterIP      10.35.242.59    <none>           80/TCP                                       44m
```

curl the ingress gateway with the correct host match:
```
curl -v 104.154.165.58 -H "Host: hello.default.example.com"
```

The output should look similar to below. Here we can see that the request was served by envoy `x-envoy-upstream-service-time: 2677`
```
% curl -v 104.154.165.58 -H "Host: hello.default.example.com"
*   Trying 104.154.165.58...
* TCP_NODELAY set
* Connected to 104.154.165.58 (104.154.165.58) port 80 (#0)
> GET / HTTP/1.1
> Host: hello.default.example.com
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain; charset=utf-8
< date: Wed, 28 Jul 2021 18:47:01 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 2677
< 
Hello World!
* Connection #0 to host 104.154.165.58 left intact
* Closing connection 0
```

We can double check that the pod has the istio sidecar proxy attached by checking `kubectl get pods` and using `kubectl describe` to check the pod events:
```
% k get pods                                                 
NAME                                      READY   STATUS    RESTARTS   AGE
hello-world-deployment-676674dc86-mk2bj   3/3     Running   0          5s
sleep-6fb84cbcf-57z96                     2/2     Running   0          47m
```

```
k describe pods hello-world-deployment-676674dc86-mk2bj
<...>
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  43s   default-scheduler  Successfully assigned default/hello-world-deployment-676674dc86-mk2bj to gke-ly-cluster-default-pool-e75444fc-wvzr
  Normal  Started    42s   kubelet            Started container user-container
  Normal  Created    42s   kubelet            Created container istio-init
  Normal  Started    42s   kubelet            Started container istio-init
  Normal  Pulled     42s   kubelet            Container image "gcr.io/knative-samples/helloworld-go@sha256:5ea96ba4b872685ff4ddb5cd8d1a97ec18c18fae79ee8df0d29f446c5efe5f50" already present on machine
  Normal  Created    42s   kubelet            Created container user-container
  Normal  Pulled     42s   kubelet            Container image "docker.io/istio/proxyv2:1.10.2" already present on machine
  Normal  Pulled     42s   kubelet            Container image "gcr.io/knative-releases/knative.dev/serving/cmd/queue@sha256:6c6fdac40d3ea53e39ddd6bb00aed8788e69e7fac99e19c98ed911dd1d2f946b" already present on machine
  Normal  Created    42s   kubelet            Created container queue-proxy
  Normal  Started    42s   kubelet            Started container queue-proxy
  Normal  Pulled     42s   kubelet            Container image "docker.io/istio/proxyv2:1.10.2" already present on machine
  Normal  Created    42s   kubelet            Created container istio-proxy
  Normal  Started    42s   kubelet            Started container istio-proxy
```

#### Trigger knative service internally
If triggering the knative service through the external LB is not an option, below will guide us through how to do so internally

Deploy sleep app to run curl commands from:
```
kubectl create -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml -n default
```

Get our knative-local-gateway CLUSTER-IP
```
kubectl get svc -n istio-system
```

The output should look similar to below. The value we are looking for in this case is the `CLUSTER-IP` of the `knative-local-gateway` service. In our case this is the `10.35.242.59`
```
% kubectl get svc -n istio-system
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                      AGE
istio-ingressgateway    LoadBalancer   10.35.253.134   104.154.165.58   15021:30144/TCP,80:30485/TCP,443:31656/TCP   45m
istiod                  ClusterIP      10.35.254.69    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP        45m
knative-local-gateway   ClusterIP      10.35.242.59    <none>           80/TCP                                       44m
```

Exec into sleep container and curl the knative-local-gateway
```
kubectl exec deploy/sleep -n default -- curl -v -H "Host: hello.default.example.com" 10.35.242.59
```

The output should look similar to below. Here we can see that the request was served by envoy `x-envoy-upstream-service-time: 2237`
```
% kubectl exec deploy/sleep -- curl -v -H "Host: hello.default.example.com" 10.35.242.59
*   Trying 10.35.242.59:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.35.242.59 (10.35.242.59) port 80 (#0)
> GET / HTTP/1.1
> Host: hello.default.example.com
> User-Agent: curl/7.69.1
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0Hello World!
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain; charset=utf-8
< date: Wed, 28 Jul 2021 18:57:36 GMT
< server: envoy
< x-envoy-upstream-service-time: 2237
< 
{ [13 bytes data]
100    13  100    13    0     0      5      0  0:00:02  0:00:02 --:--:--     5
* Connection #0 to host 10.35.242.59 left intact
```

#### Watch your service scale to zero
```
kubectl get pod -l serving.knative.dev/service=hello -w
```

It may take up to 2 minutes for your Pods to scale down. Pinging your service again will reset this timer. Output should look similar to below
```
% kubectl get pod -l serving.knative.dev/service=hello -w
NAME                                      READY   STATUS    RESTARTS   AGE
hello-world-deployment-676674dc86-kl8mk   3/3     Running   0          8s
hello-world-deployment-676674dc86-kl8mk   3/3     Terminating   0          64s
hello-world-deployment-676674dc86-kl8mk   2/3     Terminating   0          67s
hello-world-deployment-676674dc86-kl8mk   0/3     Terminating   0          97s
hello-world-deployment-676674dc86-kl8mk   0/3     Terminating   0          103s
hello-world-deployment-676674dc86-kl8mk   0/3     Terminating   0          103s
```

#### Traffic Splitting
See [Traffic Splitting](https://knative.dev/docs/getting-started/first-traffic-split/) knative docs for reference

Create a revision
```
kn service update hello \
--env TARGET=Knative \
--revision-name=knative
```

Now you can curl our knative service through the istio gateway again and see that the output has changed from `Hello World!` to `Hello Knative!`:
```
% curl 104.154.165.58 -H "Host: hello.default.example.com" 
Hello Knative!
```

List our revisions
```
kn revisions list
```

As you can see both of our revisions are there, but 100% of traffic is routing to our latest revision
```
% kn revisions list
NAME            SERVICE   TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
hello-knative   hello     100%             2            2m44s   4 OK / 4     True    
hello-world     hello                      1            144m    3 OK / 4     True    
```

Let's split traffic 50/50
```
kn service update hello \
--traffic hello-world=50 \
--traffic @latest=50
```

Verify traffic split
```
% kn revisions list        
NAME            SERVICE   TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
hello-knative   hello     50%              2            4m19s   3 OK / 4     True    
hello-world     hello     50%              1            145m    3 OK / 4     True    
```

Now we can curl our knative service again:
```
% curl 104.154.165.58 -H "Host: hello.default.example.com"
Hello World!
% curl 104.154.165.58 -H "Host: hello.default.example.com"
Hello Knative!
```

### End of Part 1
Congrats! At this point we have successfully
- Installed knative-serving
- Installed Istio
- Configured knative and Istio together
- Deployed our first serverless knative app
- Triggered our serverless app through the service mesh externally and internally (using default PERMISSIVE mTLS)
- Set up a knative revision
- Split traffic between two revisions of a knative service

## Next Steps - Setting up STRICT mtls for knative-serving
At this point, we have run through our first knative-serving example without explicitly specifying an Istio `PeerAuthentication` policy for mtls. If not defined, Istio by default will configure the destination workloads using `PERMISSIVE` mode. When `PERMISSIVE` mode is enabled, a service can accept both **plain text and mutual TLS traffic**. In order to only allow mutual TLS traffic, the configuration needs to be changed to `STRICT` mode.

- [Tutorial #2 - Setting up STRICT mtls for knative-serving](https://github.com/ably77/knative-istio-tutorial/blob/main/strict-mtls.md)

In part 2 of this tutorial we will expand further on this with a second example:
- Setting up STRICT mtls for knative-serving
- Deploy a second serverless app `httpbin`
- Confirm mTLS is enforced
- Explore using Istio `AuthorizationPolicy` to further secure our services

## Additional Next Steps - Install Gloo Mesh
[Gloo Mesh](https://docs.solo.io/gloo-mesh/latest/) is a Kubernetes-native management plane that enables configuration and operational management of multiple heterogeneous service meshes across multiple clusters through a unified API. The Gloo Mesh API integrates with the leading service meshes and abstracts away differences between their disparate API's, allowing users to configure a set of different service meshes through a single API. Gloo Mesh is engineered with a focus on its utility as an operational management tool, providing both graphical and command line UIs, observability features, and debugging tools.

Follow the [Gloo mesh Enterprise Docs](https://docs.solo.io/gloo-mesh/latest/setup/installation/enterprise_installation/) to deploy Gloo Mesh with `meshctl` CLI or Helm!

Once gloo-mesh is deployed and the cluster(s) registered, we can see our hello-world instance workloads and destinations in the mesh updated and removed as the serverless function is scaled up and then scaled down to zero.

![](https://github.com/ably77/knative-istio-tutorial/raw/main/images/gm1.png)

![](https://github.com/ably77/knative-istio-tutorial/raw/main/images/gm2.png)









