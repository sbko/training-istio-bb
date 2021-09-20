# Envoy Filters

Let's deploy the web frontend workload first using `kubectl apply -f envoy-demo-apps.yaml`.

An type of envoy filter is to write filters using Lua scripts. An existing filter called HTTP Lua filter allows you to include a Lua script inline with the configuration. Here’s an example of what a configured Lua HTTP filter that adds a new header to the response would look like:
We will create an Envoy filter that adds a header `api-version` to the HTTP response:


```yaml                                                                                                    
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: api-header-filter
  namespace: default
spec:
  workloadSelector:
    labels:
      app: web-frontend
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        portNumber: 8080
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "envoy.filters.http.router"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.lua
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
          inlineCode: |
            function envoy_on_response(response_handle)
              response_handle:headers():add("api-version", "v1")
            end

```

Save the above file to `envoy-header-filter.yaml` and deploy it using `kubectl apply -f envoy-header-filter.yaml`.

To see the header added, you can send a request to the Ingress gateway with the following command:

```bash
curl -s -I -X HEAD  https://frontend.bigbang.dev/
HTTP/2 200 
x-powered-by: Express
content-type: text/html; charset=utf-8
content-length: 2471
etag: W/"9a7-hEXE7lJW5CDgD+e2FypGgChcgho"
date: Thu, 19 Aug 2021 14:38:12 GMT
x-envoy-upstream-service-time: 496
api-version: v1
server: istio-envoy
```



## Cleanup

To cleanup, delete the workloads and the EnvoyFilter:

```
kubectl delete -f envoy-demo-apps.yaml
kubectl delete -f envoy-header-filter.yaml
```
