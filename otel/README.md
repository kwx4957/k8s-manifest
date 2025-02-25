## Otel 설치

설치 과정
1. otel operator
2. otel Collector(선택 사항으로 제외)
3. Instrumentation 생성
4. APP에 주석을 통한 자동 계측 

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Otel Operator
helm upgrade --install my-opentelemetry-operator open-telemetry/opentelemetry-operator \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true \
  --set manager.extraArgs[0]="--enable-nginx-instrumentation=true"

# Instrumentation 생성
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: nginx-instrumentation
spec:
  exporter:
    endpoint: http://tempo-distributor.default.svc.cluster.local:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  nginx:
    attrs:
    - name: NginxModuleOtelMaxQueueSize
      value: "4096"
EOF

# APP 배포
# init-Container로 otel 모듈을 자동으로 설치한다.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        instrumentation.opentelemetry.io/inject-nginx: "true" 
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```
https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md   
https://github.com/open-telemetry/opentelemetry-operator/tree/main/tests/e2e-instrumentation/instrumentation-nginx-multicontainer  
