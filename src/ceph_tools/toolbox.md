# Toolbox

Rook toolbox 는 Rook 의 디버깅과 테스트를 위한 툴 컨테이너입니다. toolbox 는 CentOS 를 기반으로 만들어져 있어, `yum` 패키지 매니저를 사용하여 손쉽게 추가 패키지 설치가 가능합니다.

toolbox 는 두가지 모드로 실행할 수 있습니다.
1. [Interactive](toolbox.html#interactive-toolbox): Shell 에서 Ceph 에 연결하여 커맨드를 실행할 수 있는 toolbox pod 을 시작합니다.
2. [One-time job](toolbox.html#toolbox-job): Ceph 커맨드를 실행하여 job 로그를 가져옵니다.

> Prerequisite: toolbox 를 실행하기 전에, Rook 클러스터가 배포되어 있어야 합니. ([Quickstart Guide](/quickstart.html) 를 참고하세요.)

## Interactive Toolbox

Rook toolbox 는 쿠버네티스 클러스터 안에서 Deployment 로 배포할 수 있으며, Pod 내에서 Ceph 커맨드를 실행할 수 있습니다.

rook-ceph-tools pod 을 배포합니다.
```sh
kubectl create -f deploy/examples/toolbox.yaml
```

toolbox pod 이 `Runinng` 상태가 될 때까지 기다립니다.
```sh
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
```

rook-ceph-tools 가 `Running` 상태이면, Pod 에 연결해 커맨드를 실행할 수 있습니다.

**Example**
- `ceph status`
- `ceph osd status`
- `ceph df`
- `rados df`

작업이 끝낫다면, Deployment 를 삭제합니다.
```sh
kubectl -n rook-ceph delete deploy/rook-ceph-tools
```

## Toolbox Job

커맨드를 한번만 실행해서 결과를 얻고 싶다면, 쿠버네티스 Job 을 사용할 수 있습니다. toolbox job 은 job 스펙에 명시되어 있는 스크립트를 실행합니다. 스크립트는 Bash 스크립트로 작성합니다.

아래 예제에서, job 이 생성되면 `ceph status` 커맨드가 실행됩니다.
```sh
kubectl create -f deploy/examples/toolbox-job.yaml
```

job 이 완료되면, 아래 커맨드로 스크립트의 결과를 화인합니다.
```sh
kubectl -n rook-ceph logs -l job-name=rook-ceph-toolbox-job
```
