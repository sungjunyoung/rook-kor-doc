# Quickstart

Rook 에 오신 것을 환영합니다! 여러분의 쿠버네티스 클러스터에 확장 가능하고 견고한 Ceph 스토리지를 **cloud-native 스토리지 오케스트레이터 플랫폼** 을 사용하여 설치하는 최상의 경험을 제공하길 희망합니다.

질문이 있다면 주저하지 말고 [Slack channel](https://rook-io.slack.com/) 에 문의하세요. 

이 가이드는 Ceph 클러스터의 기본 셋업 및 여러분의 클러스터에서 Pod 들이 블록, 오브젝트, 파일 스토리지를 사용할 수 있도록 도와줄 것입니다.

**Rook 를 테스트 하기 위해서는 항상 VM 을 사용하세요. 호스트 시스템의 로컬 디바이스가 실수로 사용되지 않도록 해야 합니다.**

## Minimum Version

Rook 지원을 위해서 쿠버네티스 v1.16 혹은 그 이상이 필요합니다.

## Prerequisites

[Prerequisites](./prerequisites.html) 에 따라 쿠버네티스 클러스터가 `Rook` 을 설치하기 위해 준비가 되었는지 확인해야 합니다.

Ceph 스토리지 클러스터를 설정하기 위해서, 아래 중 적어도 하나의 스토리지 옵션이 필요합니다:

- Raw 디바이스 (파티션이 없고, 포맷되어있지 않아야함)
  - 이 요구사항은 호스트에 `lvm2` 가 설치되어 있어야 합니다. 이 디펜던시를 피하기 위해서는, 디스크에 single full-disk 파티션 (아래 참고) 을 만들어야 합니다.
- Raw 파티션 (포맷되어 있지 않아야 함)
- StorageClass 로 PersistentVolume 이 `block` 모드로 생성 가능해야함

## TL;DR

간단한 Rook 클러스터는 아래와 같이 kubectl 커맨드와 [example manifests](https://github.com/rook/rook/blob/release-1.8/deploy/examples) 로 만들 수 있습니다.

```sh
$ git clone --single-branch --branch v1.8.5 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```

클러스터가 Running 상태가 되면, [블록, 오브젝트 혹은 파일](#storage) 스토리지를 여러분의 어플리케이션에서 사용할 수 있습니다.

## Deploy the Rook Operator

첫번째 스텝은 Rook operator 를 배포하는 것입니다. 원하는 Rook 릴리즈 버전의 [example yaml files](https://github.com/rook/rook/blob/release-1.8/deploy/examples) 을 사용하는지 확인해야 합니다. 더 많은 옵션은, [examples 문서](/ceph_storage/examples.html) 를 확인하세요.

```sh
cd deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml

# verify the rook-ceph-operator is in the `Running` state before proceeding
kubectl -n rook-ceph get pod
```

[Rook Helm Chart](/helm_charts/ceph_operator.html) 로 배포할 수도 있습니다.

프로덕션에 배포하기 전에 고려해야 할 사항이 몇가지 존재합니다.
1. 디폴트로 비활성화 되어 있는 Rook 기능들을 활성화하기를 원하다면, 추가 설정을 위해 [operator.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/operator.yaml) 을 확인하세요.
   1. Device discovery: `ROOK_ENABLE_DISCOVERY_DAEMON` 설정이 활성화 되어 있다면, 일반적으로 베어메탈 클러스터에서 사용되어 신규 디바이스를 감지하고 설정합니다.
   2. Node affinity and tolerations: CSI 드라이버는 기본적으로 어떤 노드에나 스케줄링 될 수 있습니다. CSI 드라이버 affinity 를 지정하기 위한 여러 설정들이 존재합니다.

`rook-ceph` 네임스페이스 외에 다른 네임스페이스에 설치하기 원하다면, [Ceph advanced configuration 섹션](/ceph_tools/advanced_configuration.html#using-alternate-namespaces) 을 참고하세요.


## Cluster Environments

Rook 문서는 프로덕션 환경에서 Rook 을 시작하는 것에 초점을 맞추고 있습니다. 
물론 테스트 환겨을 위한 예제들도 제공하고 있으며, 가이드 외에 다른 환경에 클러스터를 구축하길 원한다면, 아래 예제 클러스터 manifest 들을 참고하세요.

- [cluster.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/cluster.yaml): 베어메탈, 프로덕션 클러스터에서 설정. 적어도 세개의 워커 노드가 있어야 합니다.
- [cluster-on-pvc.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/cluster-on-pvc.yaml): 여러 클라우드 환경에서 프로덕션 클러스터를 설치하기 위한 설정
- [cluster-test.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/cluster-test.yaml): minikube 와 같은 테스트 환경에서 설치하기 위한 설정

더 자세한 정보는 [Ceph examples](/ceph_storage/examples.html) 를 참고하세요.

## Create a Ceph Cluster

이제 Rook operator 가 클러스터에 설치되고, Ceph 클러스터를 만들 수 있게 되었습니다. 클러스터가 리부팅으로부터 온전하기 위해서, `dataDirHostPath` 프로퍼티가 호스트에서 잘 설정되어 있는지 확인하세요. 더 많은 설정은, [configuring the cluster](/ceph_storage/cluster_crd.html) 를 참고하세요.

Ceph 클러스터를 생성합니다.
```sh
kubectl create -f cluster.yaml
```

`rook-ceph` 네임스페이스에 Pod 리스트를 확인하기 위해서 `kubectl` 커맨드를 사용하세요. 모든 Pod 들이 Running 상태라면 아래와 같은 결과를 확인할 수 있습니다. osd pod 들의 갯수는 클러스터 내의 노드와 디바이스 갯수에 따라 다릅니다. `cluster.yaml` 파일을 수정하지 않았다면, OSD 는 노드의 갯수만큼 만들어집니다. 

> 만약 `rook-ceph-mon`, `rook-ceph-mgr`, `rook-ceph-osd` pod 이 생성되지 않았다면, [Ceph common issues](/ceph_tools/common_issues.html) 을 확인하세요.

```sh
kubectl -n rook-ceph get pod
```
```sh
NAME                                                 READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-provisioner-d77bb49c6-n5tgs         5/5     Running     0          140s
csi-cephfsplugin-provisioner-d77bb49c6-v9rvn         5/5     Running     0          140s
csi-cephfsplugin-rthrp                               3/3     Running     0          140s
csi-rbdplugin-hbsm7                                  3/3     Running     0          140s
csi-rbdplugin-provisioner-5b5cd64fd-nvk6c            6/6     Running     0          140s
csi-rbdplugin-provisioner-5b5cd64fd-q7bxl            6/6     Running     0          140s
rook-ceph-crashcollector-minikube-5b57b7c5d4-hfldl   1/1     Running     0          105s
rook-ceph-mgr-a-64cd7cdf54-j8b5p                     1/1     Running     0          77s
rook-ceph-mon-a-694bb7987d-fp9w7                     1/1     Running     0          105s
rook-ceph-mon-b-856fdd5cb9-5h2qk                     1/1     Running     0          94s
rook-ceph-mon-c-57545897fc-j576h                     1/1     Running     0          85s
rook-ceph-operator-85f5b946bd-s8grz                  1/1     Running     0          92m
rook-ceph-osd-0-6bb747b6c5-lnvb6                     1/1     Running     0          23s
rook-ceph-osd-1-7f67f9646d-44p7v                     1/1     Running     0          24s
rook-ceph-osd-2-6cd4b776ff-v4d68                     1/1     Running     0          25s
rook-ceph-osd-prepare-node1-vx2rz                    0/2     Completed   0          60s
rook-ceph-osd-prepare-node2-ab3fd                    0/2     Completed   0          60s
rook-ceph-osd-prepare-node3-w4xyz                    0/2     Completed   0          60s
```

클러스터가 healty 상태임이 확인되면, [Rook tollbox](/ceph_tools/toolbox.html) 에 연결하여 `ceph status` 커맨드를 입력합니다.
- 모든 mon 들이 quorum 내에 존재함
- mgr 이 active 상태
- 하나 이상의 OSD 가 active 상태
- health 가 `HEALTH_OK` 상태가 아니라면, warning 혹은 error 가 나타납니다.

```sh 
ceph status
```
```sh
 cluster:
   id:     a0452c76-30d9-4c1a-a948-5d8405f19a7c
   health: HEALTH_OK

 services:
   mon: 3 daemons, quorum a,b,c (age 3m)
   mgr: a(active, since 2m)
   osd: 3 osds: 3 up (since 1m), 3 in (since 1m)
...
```
클러스터가 healty 상태가 아니라면, [Ceph common issues](/ceph_tools/common_issues.html) 를 참고하세요.

## Storage

Rook 에 의해서 세 타입의 스토리지를 사용할 수 있습니다. 각각의 가이드 링크를 참고하세요.
- [Block](/ceph_storage/block_storage.html): Pod 에 의해 사용되는 블록 스토리지를 생성합니다. (RWO)
- [Shared Filesystem](/ceph_storage/shared_filesystem.html): 여러 Pod 에 걸쳐 공유되는 파일시스템을 생성합니다. (RWX)
- [Object](/ceph_storage/object_storage.html): 쿠버네티스 클러스터 안밖에서 사용가능한 오브젝트 스토어를 만듭니다.

## Ceph Dashboard

Ceph 은 클러스터 상태를 확인할 수 있는 대시보드를 제공합니다. [dashboard guide](/ceph_storage/ceph_dashboard.html) 를 참고하세요.

## Tools

toolbox pod 은 Rook 클러스터를 디버깅하고 트러블슈팅 할 수 있는 ceph admin client 권한을 가집니다. 셋업하고 사용하기 위해서 [toolbox documentation](/ceph_tools/toolbox.html) 를 참고하세요. 유지보수와 튜닝을 위해서는 [advanced configuration](/ceph_tools/advanced_configuration.html) 을 참고하세요.

## Monitoring

Rook 클러스터는 [Prometheus](https://prometheus.io/) 를 통해 매트릭을 제공합니다. Rook 클러스터에 모니터링을 세팅하기 위해서는, [monitoring guide](/ceph_storage/prometheus_monitoring.html) 를 참고하세요.

## Teardown

클러스터 테스트가 완료되었다면, 클러스터 클린업을 위해서 [다음의](/ceph_storage/cleanup.html) 가이드를 참고하세요.