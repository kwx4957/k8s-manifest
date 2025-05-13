## Prometheus 설치
- kube-Prometheus-stack(그라파나를 포함한 프로메테우스가 제공하는 모든 기능 설치)
- Prometheus(프로메테우스만 설치. expoerter, alertmanager, kube-state-metrics는 제외)
- prometheus-operator-crds(CRD만 필요한 경우)

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/kube-prometheus-stack > values.yaml 

vi values.yaml
---
# 파드 메트릭 설정
kube-state-metrics:
  metricLabelsAllowlist:
    - pods=[*]

# 프로메테우스 볼륨 설정
prometheus:
  prometheusSpec:
    enableRemoteWriteReceiver: true
    storageSpec: 
      volumeClaimTemplate:
        metadata:
          name: prometheus-data
        spec:
          storageClassName: nfs
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:s
              storage: 20Gi

# 그라파나 볼륨 설정
# grafana 설정은 grafana/values.yaml에서 확인할수 있다.
grafana:
  service:
    type: LoadBalancer

  persistence:
    enabled: true
    type: pvc
    storageClassName: nfs
    accessModes:
      - ReadWriteOnce
    size: 20Gi    
    finalizers:
      - kubernetes.io/pvc-protection
---

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
-f values.yaml \
--create-namespace

export GRAFANA_PASSWORD=$(kubectl get secret prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d)
echo $GRAFANA_PASSWORD
```
https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack  
https://malwareanalysis.tistory.com/613
https://newtype.pe.kr/entry/Grafana%EC%99%80-EFS-%EC%97%B0%EB%8F%99  
https://stackoverflow.com/questions/67652819/grafana-pod-is-in-init-error-state-after-adding-an-existing-pvc  

## TroubleShooting 
kube-prxoy에 대한 메트릭을 가져오지 못하는 경우 cm를 수정한다.
```sh
# Proxy Metrcis 설정
kubectl -n kube-system edit cm kube-proxy
metricsBindAddress: 0.0.0.0:10249
kubectl -n kube-system rollout restart daemonset/kube-proxy
```
https://ssnotebook.tistory.com/entry/Prometheus-Kube-Proxy-Metric-%ED%99%9C%EC%84%B1%ED%99%94  

---

ETCD 시크릿 생성
```sh
# ETCD 설정 파일 
cat /etc/etcd.env

# ETCD 통신을 위한 시크릿 생성
kubectl create secret generic etcd-client-cert -n monitoring \
--from-file=etcd-ca=/etc/ssl/etcd/ssl/ca.pem \
--from-file=etcd-client=/etc/ssl/etcd/ssl/master.pem \
--from-file=etcd-client-key=/etc/ssl/etcd/ssl/master-key.pem
```

정적 주소 기반의 ETCD(프로세스) 메트릭 가져오기
```sh
vi values.yaml

kubeEtcd:
  enabled: true
  endpoints:
    - 10.10.10.1
    - 10.10.10.2
    - 10.10.10.3

  serviceMonitor:
    enabled: true
    scheme: https
    insecureSkipVerify: false
    serverName: localhost
    caFile: /etc/prometheus/secrets/etcd-client-cert/etcd-ca
    certFile: /etc/prometheus/secrets/etcd-client-cert/etcd-client
    keyFile: /etc/prometheus/secrets/etcd-client-cert/etcd-client-key
    port: https-metrics

prometheus:
  prometheusSpec:
    secrets:
    - etcd-client-cert
```
https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack/templates/exporters/kube-etcd    
https://www.sktenterprise.com/bizInsight/blogDetail/dev/2851

---

서비스 디스커버리를 통한 ETCD(프로세스) 메트릭 가져오기
```sh
vi values.yaml

# 그라파나의 DasBoard에서 15308 import를 수행한다.
kubeEtcd:
  enabled: false

prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: kube-etcd
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file:  /etc/prometheus/secrets/etcd-client-cert/etcd-ca
          cert_file:  /etc/prometheus/secrets/etcd-client-cert/etcd-client
          key_file:  /etc/prometheus/secrets/etcd-client-cert/etcd-client-key
        relabel_configs:
          - source_labels: [__meta_kubernetes_node_label_master]
            action: keep
            regex: true
          - source_labels: [__address__]
            target_label: __address__
            regex: ([^:;]+):(\d+)
            replacement: ${1}:2379
    secrets:
      - etcd-client-cert
```
https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config  
https://grafana.com/grafana/dashboards/15308-etcd-cluster-overview/  
