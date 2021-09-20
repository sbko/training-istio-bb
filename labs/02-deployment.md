# Creating a deployment and using a Gateway to expose it

In this lab, we will deploy a Hello World application to the cluster. We will then deploy a Hello World application, a Service resource and a VirtualService that binds to the ingress gateway `istio-system/public` to expose the application on the external IP address.


Let's enable automatic sidecar injection on the default namespace by adding  the label istio-injection=enabled:

```bash
kubectl label namespace default istio-injection=enabled
```

Check that the `default` namespace contains the label for Istio proxy injection.

```bash
kubectl get namespace -L istio-injection
```

```bash
default             Active   19h   enabled
kube-system         Active   19h   
kube-public         Active   19h   
kube-node-lease     Active   19h   
flux-system         Active   19h   
bigbang             Active   16h   
jaeger              Active   16h   enabled
gatekeeper-system   Active   16h   
istio-operator      Active   16h   disabled
logging             Active   16h   enabled
monitoring          Active   16h   
kiali               Active   16h   enabled
istio-system        Active   16h   
eck-operator        Active   16h   
```

## Deploying the Hello-World app

> To execute the following steps in a Big Bang deployment it is necessary to make modifications in the contrains allowed-docker-registries, that initially includes only ["registry1.dso.mil", "registry.dso.mil"]
> In the dev/configmap.yaml make the following modifications:
> gatekeeper:
> ```yaml
>  values:
>    violations:
>        allowedDockerRegistries:
>          parameters:
>            exemptContainers: []
>            repos:
>            - registry1.dso.mil
>            - registry.dso.mil
>            - gcr.io/tetratelabs
>            - docker.io/istio
> ```

The next step is to create the Hello World deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - image: gcr.io/tetratelabs/hello-world:1.0.0
          imagePullPolicy: Always
          name: svc
          ports:
            - containerPort: 3000
---
kind: Service
apiVersion: v1
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      name: http
      targetPort: 3000
```

Save the above YAML to hello-world.yaml and create the deployment and service using `kubectl apply -f hello-world.yaml`. If we look at the created Pods, we will notice  in pod `hello-world`, two containers running. One is the Envoy sidecar proxy, and the second one is the application. We have also created a Kubernetes service called hello-world:

```bash
kubectl get po,svc -l=app=hello-world

NAME                              READY   STATUS    RESTARTS   AGE
pod/hello-world-85c8685dd-7n2dw    2/2    Running      0       7m38s

NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/hello-world   ClusterIP   10.43.4.118   <none>        80/TCP    7m38s

```

The next step is to create a VirtualService for the hello-world service and bind it to the Gateway resource:

```yaml                                                                                                      
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-world
spec:
  hosts:
    - 'hello.bigbang.dev'
  gateways:
    - istio-system/public
  http:
    - route:
        - destination:
            host: hello-world.default.svc.cluster.local
            port:
              number: 80
```

We are matching the value of the hosts field with the hosts defined in the Gateway resource. We have also added the Gateway resource `istio-system/public` to the gateways array. Finally, we are specifying a single route with a destination that points to the Kubernetes service hello-world.default.svc.cluster.local.

Save the above YAML to vs-hello-world.yaml and create the VirtualService using `kubectl apply -f vs-hello-world.yaml`. If you look at the deployed VirtualService, you should see a similar output:

```bash
kubectl get vs
NAME          GATEWAYS                  HOSTS                   AGE
hello-world   ["istio-system/public"]   ["hello.bigbang.dev"]   80m
```

To reach the host `hello.bigbang.dev`, it is necessary to add the following line in /etc/hosts:

```bash
<public-ip> hello.bigbang.dev
```


If we run cURL against `hello.bigbang.dev` or open it in the browser, we will get back a response of Hello World:

```bash
curl -v  https://hello.bigbang.dev/
*   Trying 18.222.24.147:443...
* Connected to hello.bigbang.dev (18.222.24.147) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.bigbang.dev
*  start date: Jun 30 08:41:48 2021 GMT
*  expire date: Sep 28 08:41:47 2021 GMT
*  subjectAltName: host "hello.bigbang.dev" matched cert's "*.bigbang.dev"
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x55ae9fff8960)
> GET / HTTP/2
> Host: hello.bigbang.dev
> user-agent: curl/7.78.0
> accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 200 
< date: Mon, 16 Aug 2021 19:48:15 GMT
< content-length: 11
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 23
< 
* Connection #0 to host hello.bigbang.dev left intact
Hello World
```

## Clean-up

The following commands will clean-up your cluster.

Delete the nginx app. Be sure to run the command from the directory `hello-world.yaml` file is located.
```bash
kubectl delete -f hello-world.yaml
```

Delete the `hello-world` Virtual Service.

```bash
kubectl delete -f vs-hello-world.yaml 
```
