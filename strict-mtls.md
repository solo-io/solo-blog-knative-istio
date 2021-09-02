# Setting up STRICT mtls for knative-serving
For this tutorial we will be using a `httpbin` service in the form of a knative service. This will help us more clearly identify whether mtls is working as expected by looking for the envoy `X-Forwarded-Client-Cert` in the headers.

## create httpbin kn service with CLI
```
kn service -n default create httpbin \
--image docker.io/kennethreitz/httpbin \
--port 80 \
--env TARGET=httpbin \
--revision-name=httpbin
```

## get svc
```
% k get svc -n istio-system
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
istio-ingressgateway    LoadBalancer   10.104.186.58    34.136.77.209   15021:31110/TCP,80:32068/TCP,443:31988/TCP   140m
istiod                  ClusterIP      10.104.180.235   <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP        140m
knative-local-gateway   ClusterIP      10.104.184.118   <none>          80/TCP                                       72m
```

## curl external LB to see if a cert is passed (should be no)
```
% curl -v 34.136.77.209/headers -H "Host: httpbin.default.example.com"
*   Trying 34.136.77.209...
* TCP_NODELAY set
* Connected to 34.136.77.209 (34.136.77.209) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-length: 423
< content-type: application/json
< date: Mon, 09 Aug 2021 15:50:21 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 7
< 
{
  "headers": {
    "Accept": "*/*", 
    "Forwarded": "for=10.60.1.1;proto=http, for=127.0.0.6", 
    "Host": "httpbin.default.example.com", 
    "K-Proxy-Request": "activator", 
    "User-Agent": "curl/7.64.1", 
    "X-B3-Parentspanid": "15da8977d43b5280", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "e4cc1206d44dabd8", 
    "X-B3-Traceid": "2a410fe45174597880146c4b34f9a7a3", 
    "X-Envoy-Attempt-Count": "1"
  }
}
* Connection #0 to host 34.136.77.209 left intact
* Closing connection 0
```

## curl external LB from inside the mesh see if a cert is passed (should be no)
```
% kubectl exec deploy/sleep -n default -- curl -v -H "Host: httpbin.default.example.com" 34.136.77.209/headers
*   Trying 34.136.77.209:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 34.136.77.209 (34.136.77.209) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.78.0-DEV
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-length: 430
< content-type: application/json
< date: Mon, 09 Aug 2021 15:52:06 GMT
< server: envoy
< x-envoy-upstream-service-time: 2555
< 
{ [430 bytes data]
100   430  100   430    0     0    166      0  0:00:02  0:00:02 --:--:--   166
* Connection #0 to host 34.136.77.209 left intact
{
  "headers": {
    "Accept": "*/*", 
    "Forwarded": "for=10.128.0.110;proto=http, for=127.0.0.6", 
    "Host": "httpbin.default.example.com", 
    "K-Proxy-Request": "activator", 
    "User-Agent": "curl/7.78.0-DEV", 
    "X-B3-Parentspanid": "1007ea949424b775", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "9b3ba429867d9890", 
    "X-B3-Traceid": "4a76ff0376ad40cf013ddc9582083092", 
    "X-Envoy-Attempt-Count": "1"
  }
}
```

## Set STRICT PeerAuthentication policy for whole mesh
```
kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

**NOTE**: It may take a minute for the `PeerAuthentication` policy to propagate across the entire cluster before moving on

## now curl the external loadbalancer directly
```
% curl -v 34.136.77.209/headers -H "Host: httpbin.default.example.com"                                        
*   Trying 34.136.77.209...
* TCP_NODELAY set
* Connected to 34.136.77.209 (34.136.77.209) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-length: 652
< content-type: application/json
< date: Mon, 09 Aug 2021 15:54:04 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 8
< 
{
  "headers": {
    "Accept": "*/*", 
    "Forwarded": "for=10.128.0.110;proto=http, for=127.0.0.6", 
    "Host": "httpbin.default.example.com", 
    "K-Proxy-Request": "activator", 
    "User-Agent": "curl/7.64.1", 
    "X-B3-Parentspanid": "fc9bdde9d5cd2a0a", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "daa90fb122374314", 
    "X-B3-Traceid": "83977095f25a11875380a324f1de0623", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=764a17b2e2662930c0887e4b947f58dd512a89b700f75c20f9ce5c30a74f5f7e;Subject=\"\";URI=spiffe://cluster.local/ns/knative-serving/sa/controller"
  }
}
* Connection #0 to host 34.136.77.209 left intact
* Closing connection 0
```

## If not deployed, deploy sleep container to run curl commands from
Deploy sleep app to run curl commands from:
```
kubectl create -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml -n default
```

## curl from container inside mesh
```
% k exec deploy/sleep -n default -c sleep -- curl -v 34.136.77.209/headers -H "Host: httpbin.default.example.com"
*   Trying 34.136.77.209:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 34.136.77.209 (34.136.77.209) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.78.0-DEV
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-length: 656
< content-type: application/json
< date: Mon, 09 Aug 2021 15:55:40 GMT
< server: envoy
< x-envoy-upstream-service-time: 4162
< 
{ [656 bytes data]
100   656  100   656    0     0    157      0  0:00:04  0:00:04 --:--:--   157
* Connection #0 to host 34.136.77.209 left intact
{
  "headers": {
    "Accept": "*/*", 
    "Forwarded": "for=10.128.0.110;proto=http, for=127.0.0.6", 
    "Host": "httpbin.default.example.com", 
    "K-Proxy-Request": "activator", 
    "User-Agent": "curl/7.78.0-DEV", 
    "X-B3-Parentspanid": "8087936530de64a0", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "5d7734de789dd179", 
    "X-B3-Traceid": "ab4b808a994789d1cb709985407d4c78", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=764a17b2e2662930c0887e4b947f58dd512a89b700f75c20f9ce5c30a74f5f7e;Subject=\"\";URI=spiffe://cluster.local/ns/knative-serving/sa/controller"
  }
}
```

## Confirm
As you can see from the `X-Forwarded-Client-Cert` that the mtls is working as expected
```
"X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=764a17b2e2662930c0887e4b947f58dd512a89b700f75c20f9ce5c30a74f5f7e;Subject=\"\";URI=spiffe://cluster.local/ns/knative-serving/sa/controller"
```

## Testing - Lock down your service further

### Set default deny all Istio AuthorizationPolicy
```
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}
EOF
```

If you re-run the curl command it should hang, this is because we have by default denied all connections with the `AuthorizationPolicy` above
```
% kubectl exec deploy/sleep -n default -- curl -v -H "Host: httpbin.default.example.com" 10.217.5.142/headers
*   Trying 10.217.5.142:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.217.5.142 (10.217.5.142) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.78.0-DEV
> Accept: */*
> 
  0     0    0     0    0     0      0      0 --:--:--  0:00:27 --:--:--     0
```

### Allow default namespace to be accessed by source namespaces
```
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: allow
 namespace: default
spec:
 action: ALLOW
 rules:
 - from:
   - source:
      namespaces: ["knative-serving", "default"]
EOF
```

If you re-run the curl command we should be successful
```
% kubectl exec deploy/sleep -n default -- curl -v -H "Host: httpbin.default.example.com" 10.217.5.142/headers
*   Trying 10.217.5.142:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.217.5.142 (10.217.5.142) port 80 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.default.example.com
> User-Agent: curl/7.78.0-DEV
> Accept: */*
> 
{
  "headers": {
    "Accept": "*/*", 
    "Forwarded": "for=10.217.0.88;proto=http, for=127.0.0.6", 
    "Host": "httpbin.default.example.com", 
    "K-Proxy-Request": "activator", 
    "User-Agent": "curl/7.78.0-DEV", 
    "X-B3-Parentspanid": "04900f328faebbfa", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "c19c81e9a174cb61", 
    "X-B3-Traceid": "853b5d52d5f5ff99cdfe2d296db779ce", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=048a2894dc7e3917d1658e208404892069d19e083bf6cd7b6f8f4942bebfa94b;Subject=\"\";URI=spiffe://cluster.local/ns/knative-serving/sa/controller"
  }
}
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< access-control-allow-credentials: true
< access-control-allow-origin: *
< content-length: 655
< content-type: application/json
< date: Mon, 09 Aug 2021 18:08:49 GMT
< server: envoy
< x-envoy-upstream-service-time: 5
< 
{ [655 bytes data]
100   655  100   655    0     0  94845      0 --:--:-- --:--:-- --:--:--  106k
* Connection #0 to host 10.217.5.142 left intact
```