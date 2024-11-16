```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
```

```bash
helm install istio-base istio/base -n istio-system --create-namespace
```

```bash
helm install istiod istio/istiod -n istio-

#NAME                          READY   STATUS    RESTARTS   AGE
#pod/istiod-7cb48bbd59-8r864   1/1     Running   0          24s
#
#NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
#       AGE
#service/istiod   ClusterIP   172.20.171.111   <none>        15010/TCP,15012/TCP,443/TCP,15014
#/TCP   24s
#
#NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
#deployment.apps/istiod   1/1     1            1           24s
#
#NAME                                DESIRED   CURRENT   READY   AGE
#replicaset.apps/istiod-7cb48bbd59   1         1         1       24s
#
#NAME                                         REFERENCE           TARGETS         MINPODS   MA
#XPODS   REPLICAS   AGE
#horizontalpodautoscaler.autoscaling/istiod   Deployment/istiod   <unknown>/80%   1         5
#        1          24s

kubectl get mutatingwebhookconfigurations -n istio-system
istio-sidecar-injector          4          38s
```

```bash
helm install istio-ingressgateway istio/gateway -n istio-system

#NAME                                        READY   STATUS    RESTARTS   AGE
#pod/istio-ingressgateway-5dbfd468d7-wztcl   1/1     Running   0          8s
#pod/istiod-7cb48bbd59-8r864                 1/1     Running   0          78s
#
#NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)
#                             AGE
#service/istio-ingressgateway   LoadBalancer   172.20.77.140    <pending>     15021:31125/TCP,
#80:31421/TCP,443:30538/TCP   8s
#service/istiod                 ClusterIP      172.20.171.111   <none>        15010/TCP,15012/
#TCP,443/TCP,15014/TCP        78s
#
#NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
#deployment.apps/istio-ingressgateway   1/1     1            1           9s
#deployment.apps/istiod                 1/1     1            1           79s
#
#NAME                                              DESIRED   CURRENT   READY   AGE
#replicaset.apps/istio-ingressgateway-5dbfd468d7   1         1         1       9s
#replicaset.apps/istiod-7cb48bbd59                 1         1         1       79s
#
#NAME                                                       REFERENCE
#TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
#horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway
#<unknown>/80%   1         5         0          9s
#horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod
#<unknown>/80%   1         5         1          79s
```

```bash
export DOMAIN_NAME="cloudclub-istio.site"
export CERT_ARN=$(aws acm list-certificates --query "CertificateSummaryList[?DomainName=='${DOMAIN_NAME}'].CertificateArn" --output text)
echo "Certificate ARN for domain ${DOMAIN_NAME}: ${CERT_ARN}"
Certificate ARN for domain cloudclub-istio.site: arn:aws:acm:ap-northeast-1:xxxx0165xxxx:certificate/xxxxa97d-dcxx-45xx-87xx-350ceca5xxxx
```

```bash
cat <<EOF > ingress-alb.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-alb
  namespace: istio-system
  annotations:
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/certificate-arn: ${CERT_ARN}
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 80
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 443
EOF
```

```bash
k apply -f ingress-alb.yaml

k get all -n istio-system
#NAME                                        READY   STATUS    RESTARTS   AGE
#pod/istio-ingressgateway-5dbfd468d7-wztcl   1/1     Running   0          4m43s
#pod/istiod-7cb48bbd59-8r864                 1/1     Running   0          5m53s
#
#NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
#service/istio-ingressgateway   LoadBalancer   172.20.77.140    <pending>     15021:31125/TCP,80:31421/TCP,443:30538/TCP   4m43s
#service/istiod                 ClusterIP      172.20.171.111   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        5m53s
#
#NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
#deployment.apps/istio-ingressgateway   1/1     1            1           4m43s
#deployment.apps/istiod                 1/1     1            1           5m53s
#
#NAME                                              DESIRED   CURRENT   READY   AGE
#replicaset.apps/istio-ingressgateway-5dbfd468d7   1         1         1       4m43s
#replicaset.apps/istiod-7cb48bbd59                 1         1         1       5m53s
#
#NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
#horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          4m43s
#horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 <unknown>/80%   1         5         1          5m53s

helm upgrade istio-ingressgateway istio/gateway -n istio-system --set service.type=NodePort

#NAME                                        READY   STATUS    RESTARTS   AGE
#pod/istio-ingressgateway-5dbfd468d7-wztcl   1/1     Running   0          4m58s
#pod/istiod-7cb48bbd59-8r864                 1/1     Running   0          6m8s
#
#NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
#service/istio-ingressgateway   NodePort    172.20.77.140    <none>        15021:31125/TCP,80:31421/TCP,443:30538/TCP   4m58s
#service/istiod                 ClusterIP   172.20.171.111   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        6m8s
#
#NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
#deployment.apps/istio-ingressgateway   1/1     1            1           4m58s
#deployment.apps/istiod                 1/1     1            1           6m8s
#
#NAME                                              DESIRED   CURRENT   READY   AGE
#replicaset.apps/istio-ingressgateway-5dbfd468d7   1         1         1       4m59s
#replicaset.apps/istiod-7cb48bbd59                 1         1         1       6m9s
#
#NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
#horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          4m59s
#horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 <unknown>/80%   1         5         1          6m9s

```