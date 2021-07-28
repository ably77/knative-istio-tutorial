# knative-istio-tutorial
The purpose of this tutorial is to walk through the steps to install knative-serving and Istio on Kubernetes. This guide will also provide the extra configuration components necessary to walk through the steps on OpenShift as well.

## Prerequisites
- Kubernetes cluster up and authenticated to kubectl
- `kn` CLI installed [see Installing kn](https://knative.dev/docs/client/install-kn/)

## Demo Instructions

### Install knative-serving
For this tutorial we will be deploying knative using the [YAML method](https://knative.dev/docs/admin/install/serving/install-serving-with-yaml/). In the future we can explore using the knative operator in order to deploy and manage knative components

Install knative CRDs
```
kubectl apply -f https://github.com/knative/serving/releases/download/v0.24.0/serving-crds.yaml
```

Install knative-serving
```
kubectl apply -f https://github.com/knative/serving/releases/download/v0.24.0/serving-core.yaml
```

Check to see that the components have been deployed
```
kubectl get pods -n knative-serving
```

Output should look similar to below
```
% kubectl get pods -n knative-serving
NAME                                     READY   STATUS    RESTARTS   AGE
activator-dfc4f7578-gr72l                1/1     Running   0          15m
autoscaler-756797655b-qnpc2              1/1     Running   0          15m
controller-7bccdf6fdb-tjcxz              1/1     Running   0          15m
domain-mapping-65fd554865-8sbwc          1/1     Running   0          14m
domainmapping-webhook-7ff8f59965-zwxlf   1/1     Running   0          14m
net-istio-controller-799fb59fbf-fglzv    1/1     Running   0          13m
net-istio-webhook-5d97d48d5b-pzk4m       2/2     Running   0          13m
webhook-568c4d697-bzcdz                  1/1     Running   0          14m
```

### Install Istio
Now that knative-serving components are up, we can move on to deploying Istio. The commands below will guide you through both default and OpenShift install processes

Default Istio install
```
kubectl create ns istio-system
istioctl install --set profile=default -y
```

For OpenShift users
```
oc new-project istio-system
oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-system
istioctl install --set profile=openshift -y
```
As you can see, there is extra configuration necessary for OpenShift Istio deployments because of the use of Security Context Constraints (SCCs) in OpenShift. Istio components require the use of UID 1337 which is reserved for the sidecar proxy component. For this reason we will need to allow the `anyuid` SCC to be used anywhere Istio is used, rather than the default `restricted` SCC.

#### Using Istio mTLS capabilities
Since there are some networking communications between knative-serving namespace and the namespace where your services running on, you need additional preparations for mTLS enabled environment.

Enable sidecar container on `knative-serving` system namespace.
```
kubectl label namespace knative-serving istio-injection=enabled
```

#### Set `PeerAuthentication` to `PERMISSIVE` on knative-serving namespace
```
cat <<EOF | kubectl apply -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "knative-serving"
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

#### Install the knative istio controller
Knative uses a shared ingress Gateway to serve all incoming traffic within Knative service mesh, which is the knative-ingress-gateway Gateway under the knative-serving namespace. By default, we use Istio gateway service istio-ingressgateway under istio-system namespace as its underlying service.

Deploy the knative-istio-controller to integrate istio and knative
```
kubectl apply -f https://github.com/knative/net-istio/releases/download/v0.24.0/net-istio.yaml
```

### Deploying knative-serving apps
For this demo we will be following the [Deploying your first Knative Service](https://knative.dev/docs/getting-started/first-service/) guide from the official knative docs.

#### Enable sidecar injection on default namespace
In order for Istio to recognize workloads, it is necessary to label the namespace (in this case the default namespace) with `istio-injection=enabled`
```
kubectl label namespace default istio-injection=enabled
```

Additionally for OpenShift users, the istio-cni NetworkAttachment must be added to each namespace where you plan to deploy istio-enabled services
```
cat <<EOF | oc -n default create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: istio-cni
EOF
```

#### Create first knative hello-world service
To deploy your first knative service, run the following command
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

#### Trigger your knative service
There are multiple methods to triggering your knative-service through the istio-ingressgateway. Below will give a few examples

##### Trigger knative service directly through external LB
Get your istio-ingressgateway IP
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

The output should look similar to below. Here you can see that the request was served by envoy `x-envoy-upstream-service-time: 2677`
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

You can double check that the pod has the istio sidecar proxy attached by checking `kubectl get pods` and using `kubectl describe` to check the pod events:
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

##### Trigger knative service internally
If triggering the knative service through the external LB is not an option, below will guide you through how to do so internally

Deploy sleep app to run curl commands from:
```
kubectl create -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml -n default
```

Get your knative-local-gateway CLUSTER-IP
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
kubectl exec deploy/sleep -- curl -v -H "Host: hello.default.example.com" 10.35.242.59
```

The output should look similar to below. Here you can see that the request was served by envoy `x-envoy-upstream-service-time: 2237`
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

### Conclusion
Congrats! At this point you have successfully
- Installed knative-serving
- Installed Istio
- Configured knative and Istio together
- Deployed your first serverless knative app
- Triggered your serverless app through Istio externally and internally!







