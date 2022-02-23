# Common Issues

Rook 클러스터를 운영하면서 트러블슈팅을 돕기 위한 몇몇 팁들입니다. 이 페이지에서 찾은 해결방안으로 시도해도 문제가 해결되지 않은 경우, [Rook Slack 가입](https://slack.rook.io/) 후 General 채널에서 도움을 요청하세요. 

## Ceph

Ceph 에 관련된 이슈들은 [Ceph Common Issues](/ceph_tools/common_issues.html) 페이지를 참고하세요.

## Troubleshooting Techniques

쿠버네티스 상태와 로그는 Rook 클러스터 이슈를 확인하는데 필요한 중요 리소스입니다.

## Kubernetes Tools

쿠버네티스 상태는 클러스터가 정상 동작하지 않을 때 가장 처음 확인해야할 리소스입니다. 아래 방법들이 상태를 확인하는데 도움을 줄 수 있습니다.

- Rook pod status
  - `kubectl get po -n <cluster-namespace> -o wide`
- Logs for Rook pods
  - 오퍼레이터 로그: `kubectl logs -n <cluster-namespace> -l app=<storage-backend-operator>`
  - 특정 Pod 에 대한 로그: `kubectl logs -n <cluster-namespace> <pod-name>`, 혹은 pod label (ex. mon1) 로 조회: `kubectl logs -n <cluster-namespace> -l <label-matcher>`
  - PVC 가 왜 마운트에 실패했는지에 대한 로그: 
    - 노드에 연결하고, kubelet 로그를 확인합니다: `journalctl -u kubelet`
  - 여러 컨테이너가 동작하는 Pod
    - 모든 컨테이너에 대해서: `kubectl -n <cluster-namespace> logs <pod-name> --all-containers`
    - 특정 컨테이너에 대해서: `kubectl -n <cluster-namespace> logs <pod-name> -c <container-name>`
  - 더 이상 running 상태가 아닌 Pod 로그 확인: `kubectl -n <cluster-namespace> logs --previous <pod-name>`

몇몇 Pod 들은 initContainers 스펙을 가지고 있으며, pod 내에서 다른 컨테이너의 로그를 확인해야 할 수 도 있습니다.
- `kubectl -n <namespace> logs <pod-name> -c <container-name>`
- 다른 Rook 컴포넌트들: `kubectl -n <cluster-namespace> get all`