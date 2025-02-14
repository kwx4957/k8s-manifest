## Prometheus 설치
- kube-Prometheus-stack(그라파나를 포함한 프로메테우스가 제공하는 모든 기능 설치)
- Prometheus(프로메테우스만 설치. expoerter, alertmanager, kube-state-metrics는 제외)
- prometheus-operator-crds(CRD만 필요한 경우)

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/kube-prometheus-stack > values.yaml 

kubectl create namespace monitoring

vi values.yaml
---
prometheus:
  prometheusSpec:
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

# 기존에 없던 값들은 추가한다.
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

helm install prometheus-stack prometheus-community/kube-prometheus-stack --version 69.2.4 -f values.yaml -n monitoring

export GRAFANA_PASSWORD=$(kubectl get secret prometheus-stack-grafana -n monitoring -o jsonpath="{.data.username}" | base64 -d)
echo $GRAFANA_PASSWORD
```
https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack  
https://newtype.pe.kr/entry/Grafana%EC%99%80-EFS-%EC%97%B0%EB%8F%99  
https://stackoverflow.com/questions/67652819/grafana-pod-is-in-init-error-state-after-adding-an-existing-pvc
