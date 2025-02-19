## krew
kubectl 플러그인 관리 도구
```sh
# macOS/Linux
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

## kubectl 자동완성
```sh
echo "source <(kubectl completion bash)" >>~/.bashrc
echo "alias k=kubectl" >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

# 적용
source ~/.bashrc
```

## kubecolor
kubectl 결과에 색깔을 입힘
```sh
# 다운로드
wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_x86_64.tar.gz
tar -xvf kubecolor_0.0.25_Linux_x86_64.tar.gz

# 복사
sudo cp ./kubecolor /usr/local/bin
alias kubectl="kubecolor"

# bash 영구저장
echo "alias kubectl=kubecolor" >>~/.bashrc
echo "complete -o default -F __start_kubectl kubecolor" >>~/.bashrc

# 적용
source ~/.bashrc
```

## neat
불필요한 필드 값을 제거하고, yaml 출력값을 읽기 좋게 필터링한다.
```sh
kubectl krew install neat
kubectl get pod busbox-neat -o yaml | kubectl neat
```

## lineage
k8s 리소스 관계도 출력
```sh
kubectl krew install lineage
k lineage pod nginx -D

echo 'source <(kubectl lineage completion bash)' >> ~/.bashrc

```

## stern
pod log 조회
```sh
# default namespace 모든 pod 로그 조회
kubectl stern .

# 현재시간 기준으로 10분전, 모든 pod 로그 조회
kubectl stern . --since 10m

# 현재시간 기준으로 6시간 전, 모든 pod 로그 조회
kubectl stern . --since 6h

# 모든 pod 로그에서 404만 조회
kubectl stern . --include "404"
kubectl stern . --include "first"

# 모든 pod 로그에서 first만 제외
kubectl stern -n default . --exclude "first"
```

https://github.com/choisungwook/kubectl_plugins?tab=readme-ov-file#%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%A6%AC%EC%86%8C%EC%8A%A4-%EB%B0%B0%ED%8F%AC%EA%B4%80%EB%A0%A8-%EB%8F%84%EA%B5%AC
