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