- Data Plane
  - The Proxies are collectively called the `Data Plane` in Istio
  - helper container

---

```bash
k apply -f 1-istio-init.yaml

k apply -f 2-istio-minikube.yaml

k apply -f 3-kiaic-secret.yaml 

k label namespace default istio-injection=enabled

k describe ns default

k get ns default -o yaml

k apply -f 4-application-full-stack.yaml

# sidecar container debug mode
k exec -it <pod name> -c istio-proxy -- curl -X POST http://localhost:15000/logging\?level\=debug

# watch sidecar container's log
k logs <pod name> -c istio-proxy -f
```