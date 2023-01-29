# asm-authz-demo-01
testing asm authz + rbac 

# service a
```
kubectl --context=autopilot-cluster-1 apply -f service-a
kubectl --context=autopilot-cluster-2 apply -f service-a
```

# service b
```
kubectl --context=autopilot-cluster-1 apply -f service-b
kubectl --context=autopilot-cluster-2 apply -f service-b
```

# exec into another pod and call `service-a` and `service-b`
```
kubectl --context=autopilot-cluster-1 -n whereami-http exec --stdin --tty whereami-http-549c466796-k4lls -- /bin/sh
curl service-a.service-a.svc.cluster.local
curl service-b.service-b.svc.cluster.local
```