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
[Otel 공식문서]  
https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md   
https://github.com/open-telemetry/opentelemetry-operator/tree/main/tests/e2e-instrumentation/instrumentation-nginx-multicontainer  

[Tempo 공식문서]  
https://grafana.com/blog/2021/02/03/auto-instrumenting-a-java-spring-boot-application-for-traces-and-logs-using-opentelemetry-and-grafana-tempo/  

[블로그]  
https://joeunvit.tistory.com/15    
https://wlsdn3004.tistory.com/49#c4      
https://luvstudy.tistory.com/237#minio-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0    
https://velog.io/@utcloud/OpenTelemetry-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EA%B5%AC%EC%84%B1#opentelemetry-operator-%EC%84%A4%EC%B9%98    



## TroubleShooting
1. 연결 오류  
otel의 기본 프로토클은 grpc으로 4317으로 통신한다. 4318으로 통신하기 위해서는 프로토콜을 http으로 변경한다.

```sh
[otel.javaagent 2025-02-24 05:43:18:563 +0000] [OkHttp http://tempo-distributor.default.svc.cluster.local:4318/...] WARN io.opentelemetry.exporter.internal.grpc.GrpcExporter - Failed to export spans. Server responded with gRPC status code 2. Error message: FRAME_SIZE_ERROR: 4740180
[otel.javaagent 2025-02-24 05:43:33:571 +0000] [OkHttp http://tempo-distributor.default.svc.cluster.local:4318/...] WARN io.opentelemetry.exporter.internal.grpc.GrpcExporter - Failed to export spans. Server responded with gRPC status code 2. Error message: closed
```

2. Tempo가 분산 모드인 경우 Ingrester 에러  
Ingrester의 파드 개수를 2개 이상으로 설정함으로써 해결했다.

```sh
Error (error querying ingesters in Querier.Search: forIngesterRings: error getting replication set for ring (0): too many unhealthy instances in the ring ). Please check the server logs for more details.
```  
https://community.grafana.com/t/tempo-ingester-ring-not-forming/52137/4  

3. nginx 자동 계측 실패   
go와 nginx의 경우 자동 계측이 false로 되어있다. 따라서 해당 설정을 true 변경해줘야 한다.

```sh
{"level":"ERROR","timestamp":"2025-02-24T08:47:06Z","message":"support for Nginx auto instrumentation is not enabled","namespace":"default","generateName":"otel-nginx-66f996bd78-","stacktrace":"github.com/open-telemetry/opentelemetry-operator/pkg/instrumentation."}

helm upgrade my-opentelemetry-operator open-telemetry/opentelemetry-operator \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true \
  --set manager.extraArgs[0]="--enable-nginx-instrumentation=true"
```
https://github.com/open-telemetry/opentelemetry-operator/blob/db1598095161ea75389165ee374f05fb46ee4bf3/main.go#L153

4. sidecar 주입 에러     
기존 어노테이션 중 `sidecar.opentelemetry.io/inject: "true"`를 주석 처리했다. 해당 경우는 콜렉터가 사이드카 모드일 때 파드에 주입하는 것으로 보인다.  
```sh
{"level":"ERROR","timestamp":"2025-02-25T01:29:00Z","message":"failed to select an OpenTelemetry Collector instance for this pod's sidecar","namespace":"default","name":"","error":"no OpenTelemetry Collector instances available","stacktrace":"github.com/open-telemetry/opentelemetry-operator/pkg/sidecar."}
```

5. nginx 버전 문제  
1.26.x 버전에 대해서는 자동 계측이 동작하지 않는다. nginx 버전을 1.25.3으로 낮춘 결과 동작한다. 공식 문서에서는 1.26.0과 1.25.5에 대해서 지원한다고 한다. 하지만 해당 버전은  sdk에 대해서만 지원하고 agnet에 대해서는 아직 지원하지 않는 것으로 보인다.  
```sh
nginx: [emerg] dlopen() "/opt/opentelemetry-webserver/agent/WebServerModule/Nginx/1.26.1/ngx_http_opentelemetry_module.so" 
failed (/opt/opentelemetry-webserver/agent/WebServerModule/Nginx/1.26.1/ngx_http_opentelemetry_module.so: cannot open shared object file: No such file or directory) in /etc/nginx/nginx.conf:2

# sdk에 대해서는 기능이 지원되지만 agent에 대해서는 지원이 지원되지 않는듯 하다.
SDK:   /opt/opentelemetry-webserver-sdk/WebServerModule/Nginx/1.26.0/ngx_http_opentelemetry_module.so
AGNET: /opt/opentelemetry-webserver/agent/WebServerModule/Nginx/1.26.0/ngx_http_opentelemetry_module.so
```
https://github.com/open-telemetry/opentelemetry-cpp-contrib/pull/436/files   
https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module  
