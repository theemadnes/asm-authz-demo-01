# asm-authz-demo-01
testing asm authz + rbac 

### rename kube contexts
```
kubectx autopilot-cluster-1=gke_mc-e2m-01_us-central1_autopilot-cluster-1
kubectx autopilot-cluster-2=gke_mc-e2m-01_us-central1_autopilot-cluster-2
```

### create client 1 (w/ sidecar)
```
kubectl --context=autopilot-cluster-1 apply -f client-1-with-sidecar
kubectl --context=autopilot-cluster-2 apply -f client-1-with-sidecar
```

### create client 2 (no sidecar)
```
kubectl --context=autopilot-cluster-1 apply -f client-2-no-sidecar
kubectl --context=autopilot-cluster-2 apply -f client-2-no-sidecar
```

### create service a
```
kubectl --context=autopilot-cluster-1 apply -f service-a
kubectl --context=autopilot-cluster-2 apply -f service-a
```

### create service b
```
kubectl --context=autopilot-cluster-1 apply -f service-b
kubectl --context=autopilot-cluster-2 apply -f service-b
```

### NetworkPolicy: apply networkpolicies, exec into another pod, and attempt to call `service-a` and `service-b`
First apply:
```
kubectl --context=autopilot-cluster-1 apply -f networkpolicy-service-a
kubectl --context=autopilot-cluster-2 apply -f networkpolicy-service-a
kubectl --context=autopilot-cluster-1 apply -f networkpolicy-service-b
kubectl --context=autopilot-cluster-2 apply -f networkpolicy-service-b
```
Then test:
```
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
curl service-a.service-a.svc.cluster.local # works, needed to add CIDR block matching
curl service-b.service-b.svc.cluster.local # doesn't work -> RBAC: access denied
```

### remove network policies and apply Authorization Policies for greater granularity
```
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-b
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-b
kubectl --context=autopilot-cluster-1 apply -f asm-ap-service-a
kubectl --context=autopilot-cluster-1 apply -f asm-ap-service-b
kubectl --context=autopilot-cluster-2 apply -f asm-ap-service-a
kubectl --context=autopilot-cluster-2 apply -f asm-ap-service-b
```

### exec into another pod that's enabled to call `service-a` via authz policy and call `service-a` and `service-b`
```
kubectl --context=autopilot-cluster-1 -n whereami-http exec --stdin --tty deploy/whereami-http -- /bin/sh
curl service-a.service-a.svc.cluster.local # works
curl service-b.service-b.svc.cluster.local # doesn't work -> RBAC: access denied
```

### exec into another pod that's not enabled via authz policy and call `service-a` and `service-b`
```
kubectl --context=autopilot-cluster-1 -n whereami-grpc exec --stdin --tty deploy/whereami-grpc -- /bin/sh
curl service-a.service-a.svc.cluster.local # doesn't work -> RBAC: access denied
curl service-b.service-b.svc.cluster.local # doesn't work -> RBAC: access denied
```

### test deletion of authorization policy `service-a` by `service-a-dev@alexmattson.altostrat.com`
```
$ kubectl --context=autopilot-cluster-1 -n service-a delete authorizationpolicy service-a
Error from server (Forbidden): authorizationpolicies.security.istio.io "service-a" is forbidden: User "service-a-dev@alexmattson.altostrat.com" cannot delete resource "authorizationpolicies" in API group "security.istio.io" in the namespace "service-a": requires one of ["container.thirdPartyObjects.delete"] permission(s).
```

### create role and roleBinding to allow `service-a-dev@alexmattson.altostrat.com` to modify authorization policies
```
kubectl --context=autopilot-cluster-1 apply -f k8s-rbac-service-a
kubectl --context=autopilot-cluster-2 apply -f k8s-rbac-service-a
```

### from Cloud Shell when logged in as `service-a-dev@alexmattson.altostrat.com` attempt to delete authorization policy
```
$ kubectl --context=autopilot-cluster-1 -n service-a delete authorizationpolicy service-a
authorizationpolicy.security.istio.io "service-a" deleted
```

### recreate the authorization policy to get things back to good
```
$ kubectl --context=autopilot-cluster-1 apply -f service-a/authorizationpolicy.yaml
authorizationpolicy.security.istio.io/service-a created
```
