options:

tap filter
- https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/tap_filter
- https://medium.com/@mtchkll/solving-microservice-mysteries-with-envoys-tap-filter-fd159c36d0af
- 1k limit details: https://medium.com/@mtchkll/solving-microservice-mysteries-with-envoys-tap-filter-fd159c36d0af



lua filter
- https://themartian.hashnode.dev/http-request-body-logging-with-istio-and-envoy

> need to delete prior RBAC policies

first enable access logging 
```
cat <<EOF | kubectl --context=autopilot-cluster-1 apply -f -
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
kind: ConfigMap
metadata:
  name: istio-asm-managed
  namespace: istio-system
EOF
```

```
cat <<EOF | kubectl --context=autopilot-cluster-2 apply -f -
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
kind: ConfigMap
metadata:
  name: istio-asm-managed
  namespace: istio-system
EOF
```

verify
```
kubectl --context=autopilot-cluster-1 get configmap istio-asm-managed -n istio-system -o yaml
kubectl --context=autopilot-cluster-2 get configmap istio-asm-managed -n istio-system -o yaml
```

```
curl https://frontend.endpoints.mc-e2m-01.cloud.goog/whereami-http/
```

((resource.labels.project_id="mc-e2m-01" AND resource.labels.namespace_name="whereami-http" AND (labels."k8s-pod/service_istio_io/canonical-name"="whereami-http" OR labels."k8s-pod/service.istio.io/canonical-name"="whereami-http")) OR (resource.type="gce_instance" AND labels.mesh_uid="proj-447024719410" AND labels.destination_namespace="whereami-http" AND labels.destination_canonical_service="whereami-http"))

^^ nothing yet 

```
while true
do
   curl https://frontend.endpoints.mc-e2m-01.cloud.goog/whereami-http/
done
```

adding permissions to service account:
```
447024719410-compute@developer.gserviceaccount.com	Compute Engine default service account	
Logs Writer
Monitoring Metric Writer
```

prom setup
```
kubectl --context=autopilot-cluster-1 apply -f payload-logging-filter-example/whereami-http-prom.yaml
kubectl --context=autopilot-cluster-2 apply -f payload-logging-filter-example/whereami-http-prom.yaml
```

trying the demo
```
kubectl -n default apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.5.0/examples/example-app.yaml
kubectl -n default apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.5.0/examples/pod-monitoring.yaml
```

remove the demo 
```
kubectl -n default delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.5.0/examples/example-app.yaml
kubectl -n default delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.5.0/examples/pod-monitoring.yaml
```