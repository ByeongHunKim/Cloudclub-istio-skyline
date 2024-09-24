# Istio와 Envoy

## sidecar injection

manual과 automatic이 있지만 두 방식 모두 동일한 주입 원리를 따름.

istio-sidecar-injector configmap, 사이드카 주입 템플릿을 사용

<pre>
<code>
#아래의 명령을 사용하여 istio-sidecar-injector configmap 확인 가능

kubectl -n istio-system get configmap istio-sidecar-injector

#data.config와 data.values로 이루어져있음
#내용을 보는 방법은 아래와 같다.
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}'
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}'
</code>
</pre>

### manual

istioctl로 주입하며 istio-sidecar-injector configmap을 수정할 수 있다.

#### 기본적인 설정
<pre>
<code>
istioctl kube-inject -f application.yaml | kubectl apply -f -

#또는

kubectl apply -f <(istioctl kube-inject -f application.yaml)
</code>
</pre>

주입 시 기본 설정은 Kubernetes configmap의 istio-sidecar-injector를 사용

#### istio-sidecar-injector configmap 수정하여 설정

위에 설명했다시피 istio는 istio-sidecar-injector configmap를 사용한다. 

istioctl kube-inject는 아래의 플레그로 원하는 설정을 할 수 있다.
<pre>
<code>
--injectConfigFile string    Injection configuration filename. Cannot be used with --injectConfigMapName
--meshConfigFile string      Mesh configuration filename. Takes precedence over --meshConfigMapName if set
--meshConfigMapName string   ConfigMap name for Istio mesh configuration, key should be "mesh" (default "istio")
--injectConfigMapNam string  ConfigMap name for Istio sidecar injection, key should be "config" (default "istio-sidecar-injector")
</code>
</pre>

#injectConfigMapNam flag는 기존 istio-sidecar-injector에 Override된다.

따라서 아래의 코드를 이용해 기존 Configmap을 추출 후 재설정한다.
<pre>
<code>
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
</code>
</pre>

그 후 istioctl kube-inject명령어로 application.yaml에 해당되는 pods에 원하는 설정으로 envoy를 배포할 수 있다.
<pre>
<code>
istioctl kube-inject \
    --injectConfigFile inject-config.yaml \
    --meshConfigFile mesh-config.yaml \
    --valuesFile inject-values.yaml \
    --filename application.yaml \
    | kubectl apply -f -
</code>
</pre>


### automatic
<img src="https://www.solo.io/wp-content/uploads/2021/10/Istio-Sidecar-Injector-Webhook-1536x593.png" width="100%"/>

이 방법은 쿠버네티스가 Admission Controller를 활성화 했는지 여부에 따라 사용 가능한지 좌우된다.

1. Istio 설치 과정에서 주입된 istio-sidecar-injector로 변형된 설정은 모든 pod 정보와 함께 웹훅 요청을 istiod 컨트롤러에 전송한다.
2. 컨트롤러는 런타임에 pod 사양을 수정하여 실제 pod 사양에 init 및 sidecar container 를 주입한다.
3. 컨트롤러는 수정된 객체를 객체 검증을 위해 Admission Webhook에 반환한다.
4. 검증 후, 수정된 pod 사양이 모든 sidecar container와 함께 배포된다.


참고로 Admission Controller가 무엇인지 모른다면 아래의 링크를 참조

https://coffeewhale.com/kubernetes/admission-control/2021/04/28/opa1/

참고 내용: https://www.solo.io/blog/istios-networking-in-depth/
