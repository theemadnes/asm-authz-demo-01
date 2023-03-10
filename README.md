# asm-authz-demo-01
testing asm authz + rbac 
![rough idea of the demo setup](/assets/arch_example.png)

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

### test calls to make sure everything works at first
`client-1`
```
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
curl service-a.service-a.svc.cluster.local # works
curl service-b.service-b.svc.cluster.local # works
```

`client-2`
```
kubectl --context=autopilot-cluster-2 -n client-2 exec --stdin --tty deploy/client-2 -- /bin/sh
curl service-a.service-a.svc.cluster.local # works, but note that responses only come from autopilot-cluster-2 as no sidecar to allow cross-cluster service discovery
curl service-b.service-b.svc.cluster.local # works, but note that responses only come from autopilot-cluster-2 as no sidecar to allow cross-cluster service discovery
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
curl service-b.service-b.svc.cluster.local # fails: upstream connect error or disconnect/reset before headers. retried and the latest reset reason: connection failure
```

### remove network policies
```
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-b
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-b
```

### enable peer authentication == `strict` to require mTLS 
> This will break `client-2`, as it has no sidecar
```
kubectl --context=autopilot-cluster-1 apply -f asm-peer-authentication-meshwide
kubectl --context=autopilot-cluster-2 apply -f asm-peer-authentication-meshwide

# test from client-1
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
curl service-a.service-a.svc.cluster.local # works
curl service-b.service-b.svc.cluster.local # works

# test from client-2
kubectl --context=autopilot-cluster-1 -n client-2 exec --stdin --tty deploy/client-2 -- /bin/sh
curl service-a.service-a.svc.cluster.local # fails: Recv failure: Connection reset by peer
curl service-b.service-b.svc.cluster.local # fails: Recv failure: Connection reset by peer
```

### apply `allow nothing` Authorization Policy to entire mesh
```
kubectl --context=autopilot-cluster-1 apply -f asm-authorizationpolicy-allow-nothing
kubectl --context=autopilot-cluster-2 apply -f asm-authorizationpolicy-allow-nothing
```

### exec into a pod to verify `allow nothing` is in place
Try from `client-1` only, as `client-2` won't work as there's no sidecar
```
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
curl service-a.service-a.svc.cluster.local # doesn't work -> RBAC: access denied
curl service-b.service-b.svc.cluster.local # doesn't work -> RBAC: access denied
```


### apply service-specific Authorization Policies for request-level granularity
```
kubectl --context=autopilot-cluster-1 apply -f asm-authorizationpolicy-service-a
kubectl --context=autopilot-cluster-1 apply -f asm-authorizationpolicy-service-b
kubectl --context=autopilot-cluster-2 apply -f asm-authorizationpolicy-service-a
kubectl --context=autopilot-cluster-2 apply -f asm-authorizationpolicy-service-b
```

### exec into a pod that's enabled to call `service-a` via authz policy and call `service-a` and `service-b`
Try from `client-1` only, as `client-2` won't work as there's no sidecar
```
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
curl service-a.service-a.svc.cluster.local # works
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

### clear out all authn/z & network policies to reset back to beginning
```
kubectl --context=autopilot-cluster-1 delete -f asm-authorizationpolicy-service-a
kubectl --context=autopilot-cluster-1 delete -f asm-authorizationpolicy-service-b
kubectl --context=autopilot-cluster-2 delete -f asm-authorizationpolicy-service-a
kubectl --context=autopilot-cluster-2 delete -f asm-authorizationpolicy-service-b
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-a
kubectl --context=autopilot-cluster-1 delete -f networkpolicy-service-b
kubectl --context=autopilot-cluster-2 delete -f networkpolicy-service-b
kubectl --context=autopilot-cluster-1 delete -f asm-peer-authentication-meshwide
kubectl --context=autopilot-cluster-2 delete -f asm-peer-authentication-meshwide
kubectl --context=autopilot-cluster-1 delete -f asm-authorizationpolicy-allow-nothing
kubectl --context=autopilot-cluster-2 delete -f asm-authorizationpolicy-allow-nothing
kubectl --context=autopilot-cluster-1 delete -f k8s-rbac-service-a
kubectl --context=autopilot-cluster-2 delete -f k8s-rbac-service-a
kubectl --context=autopilot-cluster-1 delete -f k8s-rbac-service-b
kubectl --context=autopilot-cluster-2 delete -f k8s-rbac-service-b
```

### other notes
exec shortcuts
```
kubectl --context=autopilot-cluster-1 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
kubectl --context=autopilot-cluster-2 -n client-1 exec --stdin --tty deploy/client-1 -- /bin/sh
kubectl --context=autopilot-cluster-1 -n client-2 exec --stdin --tty deploy/client-2 -- /bin/sh
kubectl --context=autopilot-cluster-2 -n client-2 exec --stdin --tty deploy/client-2 -- /bin/sh
```

curl indefinitely
```
while true
do
   curl service-a.service-a.svc.cluster.local
done
```

```
while true
do
   curl service-b.service-b.svc.cluster.local
done
```
