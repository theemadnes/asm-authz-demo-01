# asm-authz-demo-01
testing asm authz + rbac 

### service a
```
kubectl --context=autopilot-cluster-1 apply -f service-a
kubectl --context=autopilot-cluster-2 apply -f service-a
```

### service b
```
kubectl --context=autopilot-cluster-1 apply -f service-b
kubectl --context=autopilot-cluster-2 apply -f service-b
```

### exec into another pod that's enabled via authz policy and call `service-a` and `service-b`
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