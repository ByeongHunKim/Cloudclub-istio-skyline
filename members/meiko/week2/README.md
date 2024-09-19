- Data Plane
  - The Proxies are collectively called the `Data Plane` in Istio
  - helper container

---

```bash
k label namespace default istio-injection=enabled

k describe ns default

k get ns default -o yaml

# sidecar container debug mode
k exec -it <pod name> -c istio-proxy -- curl -X POST http://localhost:15000/logging\?level\=debug

# watch sidecar container's log
k logs <pod name> -c istio-proxy -f
```