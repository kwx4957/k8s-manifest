## Alloy 설치
```sh
helm repo update
helm install alloy grafana/alloy --version 0.12.0
helm show values grafana/alloy --version 0.12.0 > values.yaml

// alloy는 하단 아래에 있다.
vi config.alloy

kubectl create configmap alloy-config "--from-file=config.alloy=./config.alloy" -n monitoring

vi values.yaml
---
alloy:
  configMap:
    create: false
    name: alloy-config
    key: config.alloy
---

helm upgrade alloy grafana/alloy -f alloy-values.yaml -n monitoring
```
### Config.alloy
```
// 3가지로 구성되어 있다.
// 1. 노드에 대한 syslog
// 2. k8s 이벤트 로그
// 3. pod 로그

// system log 수집 
// local.file_match discovers files on the local filesystem using glob patterns and the doublestar library. It returns an array of file paths.
local.file_match "node_logs" {
  path_targets = [{
      __path__  = "/var/log/syslog",
      job       = "node/syslog",
      node_name = sys.env("HOSTNAME"),
      cluster   = "<원하는 클러스터 이름>",
  }]
}

// loki.source.file reads log entries from files and forwards them to other loki.* components.
// You can specify multiple loki.source.file components by giving them different labels.
loki.source.file "node_logs" {
  targets    = local.file_match.node_logs.targets
  forward_to = [loki.write.default.receiver]
}

//k8s 이벤트
// loki.source.kubernetes_events tails events from the Kubernetes API and converts them
// into log lines to forward to other Loki components.
loki.source.kubernetes_events "cluster_events" {
  job_name   = "integrations/kubernetes/eventhandler"
  log_format = "logfmt"
  forward_to = [
    loki.process.cluster_events.receiver,
  ]
}

// loki.process receives log entries from other loki components, applies one or more processing stages,
// and forwards the results to the list of receivers in the component's arguments.
loki.process "cluster_events" {
  forward_to = [loki.write.default.receiver]

  stage.static_labels {
    values = {
      cluster = "<원하는 클러스터 이름>",
    }
  }

  stage.labels {
    values = {
      kubernetes_cluster_events = "job",
    }
  }
}

//수집 대상 pod log로 k8s api를 통한 로그를 수집한다.
discovery.kubernetes "pod" {
  role = "pod"
	selectors {
		role  = "pod"
		field = "spec.nodeName=" + coalesce(env("HOSTNAME"), constants.hostname)
	}
}

// discovery.relabel rewrites the label set of the input targets by applying one or more relabeling rules.
// If no rules are defined, then the input targets are exported as-is.
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pod.targets

  // Label creation - "namespace" field from "__meta_kubernetes_namespace"
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    action = "replace"
    target_label = "namespace"
  }

  // Label creation - "pod" field from "__meta_kubernetes_pod_name"
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    action = "replace"
    target_label = "pod"
  }

  // Label creation - "container" field from "__meta_kubernetes_pod_container_name"
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    action = "replace"
    target_label = "container"
  }

  // Label creation -  "app" field from "__meta_kubernetes_pod_label_app_kubernetes_io_name"
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
    action = "replace"
    target_label = "app"
  }

  // Label creation -  "job" field from "__meta_kubernetes_namespace" and "__meta_kubernetes_pod_container_name"
  // Concatenate values __meta_kubernetes_namespace/__meta_kubernetes_pod_container_name
  rule {
    source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_container_name"]
    action = "replace"
    target_label = "job"
    separator = "/"
    replacement = "$1"
  }

  // Label creation - "container" field from "__meta_kubernetes_pod_uid" and "__meta_kubernetes_pod_container_name"
  // Concatenate values __meta_kubernetes_pod_uid/__meta_kubernetes_pod_container_name.log
  rule {
    source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
    action = "replace"
    target_label = "__path__"
    separator = "/"
    replacement = "/var/log/pods/*$1/*.log"
  }

  // Label creation -  "container_runtime" field from "__meta_kubernetes_pod_container_id"
  rule {
    source_labels = ["__meta_kubernetes_pod_container_id"]
    action = "replace"
    target_label = "container_runtime"
    regex = "^(\\S+):\\/\\/.+$"
    replacement = "$1"
  }
}

//수집 된 파드 로그 -> relabel -> 하단의 프로세스 전달 -> loki write
//loki.source.kubernetes tails logs from Kubernetes containers using the Kubernetes API.
loki.source.kubernetes "pod_logs" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.process.pod_logs.receiver]
}

//loki.process receives log entries from other Loki components, applies one or more processing stages,
//and forwards the results to the list of receivers in the component's arguments.
loki.process "pod_logs" {
  stage.static_labels {
      values = {
        cluster = "<원하는 클러스터 이름>",
      }
  }

  forward_to = [loki.write.default.receiver]
}

//Loki 백엔드 저장소 쓰기 작업
loki.write "default" {
  endpoint {
    url = "http://loki-write.default.svc.cluster.local:3100/loki/api/v1/push"
  }
}
```

[alloy 설치]      
https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/    

[alloy 로그 설정]      
https://grafana.com/docs/alloy/latest/collect/metamonitoring/#meta-monitoring-logs  
https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/  

[alloy 문제 해결]   
https://github.com/grafana/alloy/issues/1217  

## TroubleShooting  
1. Loki failed to create fsnotify watcher: too many open files  
그라파나에서 Loki에 대한 로그 쿼리를 날린 순간 다음 에러가 발생한다. 이러한 원인이 발생한 파드에 대한 선택기가 없기 떄문에 발생한 문제이다. alloy.config에서 파드 셀렉터를 추가함으로써 문제를 해결했다. 
```sh 
vi alloy.config
discovery.kubernetes "pods" {
	role = "pod"

	selectors {
		role  = "pod"
		field = "spec.nodeName=" + coalesce(env("HOSTNAME"), constants.hostname)
	}
}
```
