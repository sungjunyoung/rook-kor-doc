# Common Issues

대부분의 이슈들은 간단한 몇 문장으로 요약할 수 없습니다. 몇가지 문제가 아래 리스트에 정리되어 있으며, Rook 설정에 따라 적용할 수 없다는 점을 기억하세요. 

이 페이지에서 찾은 해결방안으로 시도해도 문제가 해결되지 않은 경우, [Rook Slack 가입](https://slack.rook.io/) 후 General 채널에서 도움을 요청하세요. 

## Table of Contents
  - [Table of Contents](#table-of-contents)
  - [트러블슈팅 테크닉](#트러블슈팅-테크닉)
  - [클러스터가 서비스 요청에 응답하지 못함](#클러스터가-서비스-요청에-응답하지-못함)
  - [Mon 만 배포됨](#mon-만-배포됨)
  - [PVC 가 Pending 상태로 지속됨](#pvc-가-pending-상태로-지속됨)
  - [OSD Pod 들이 시작되지 않음](#osd-pod-들이-시작되지-않음)
  - [OSD Pod 들이 디바이스에 생성되지 않음](#osd-pod-들이-디바이스에-생성되지-않음)
  - [노드 리부팅 이후 노드가 Hang 상태로 지속됨](#노드-리부팅-이후-노드가-hang-상태로-지속됨)
  - [여러 Cephfs 를 커널 4.7 미만 버전에서 사용하려고 함](#여러-cephfs-를-커널-47-미만-버전에서-사용하려고-함)
  - [모든 Ceph 데몬들의 로그 레벨을 Debug 로 변경](#모든-ceph-데몬들의-로그-레벨을-debug-로-변경)
  - [특정 Ceph 데몬에 대해 file 로 로그를 쓰도록 변경](#특정-ceph-데몬에-대해-file-로-로그를-쓰도록-변경)
  - [RBD 디바이스를 사용하는 워커 노드가 Hang 상태로 지속됨](#rbd-디바이스를-사용하는-워커-노드가-hang-상태로-지속됨)
  - [Too few PGs per OSD 메시지가 나타남](#too-few-pgs-per-osd-메시지가-나타남)
  - [LV 백엔드 PVC 를 사용하는 OSD 에서 LVM 메타데이터가 손상됨](#lv-백엔드-pvc-를-사용하는-osd-에서-lvm-메타데이터가-손상됨)
  - [OSD Prepare Job 이 to low aio-max-nr 로 실패함](#osd-prepare-job-이-to-low-aio-max-nr-로-실패함)
  - [예상하지 않은 파티션이 생성됨](#예상하지-않은-파티션이-생성됨)


## 트러블슈팅 테크닉

클러스터 내 이슈를 확인하기 위해서 크게 두 가지를 확인해 볼 수 있습니다.
1. [쿠버네티스 상태 및 로그](/common_issues.html)
2. Ceph 클러스터 상태 (다음 섹션의 [Ceph tools](#ceph-tools))

### Ceph Tools
Pod 들이 정상적으로 Running 상태임을 확인했다면, 스토리지 컴포넌트의 상태를 확인하기 위해 Ceph tools 를 사용합니다. Rook toolbox 를 사용하거나, 이미 Running 중인 Rook pod 안에서 확인할 수 있습니다.
- 왜 PVC 가 마운트에 실패했는지에 대한 노드의 로그 확인
- [로그 수집](advanced_configuration.html#log-collection) 섹션에서 로그 확인을 위한 스크립트 확인 
- 다른 방안:
  - mon 이 quorum 내에 있는지 확인: `kubectl -n <cluster-namespace> get configmap rook-ceph-mon-endpoints -o yaml | grep data`

#### Tools in the Rook Toolbox
[rook-ceph-tools pod](toolbox.html) 은 Ceph tools 를 실행하기 위한 환경을 제공합니다. 일단 Pod 이 실행되고 나면, 현재 클러스터 상태를 확인하기 위한 Ceph 커맨드를 실행합니다.

```sh
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[*].metadata.name}') bash
```

### Ceph Commands
아래와 같은 커맨드로 현재 Ceph 클러스터의 상태를 확인할 수 있습니다.
- `ceph status`
- `ceph osd status`
- `ceph osd df`
- `ceph osd utilization`
- `ceph osd pool stats`
- `ceph osd tree`
- `ceph pg stat`

처음 두 커맨드는 전체적인 클러스터 상태를 나타냅니다. 정상적인 상태의 클러스터는 `HEALTH_OK` 이지만, `HEALTH_WARN` 상태에서도 클러스터는 정상적으로 동작합니다. 하지만 이 상태는 모든 디스크 I/O 가 불가능한 상태인 `HEALTH_ERROR` 상태로 빠질 수 있는 상태입니다. `HEALTH_WARN` 상태가 관측된다면, 클러스터가 에러 상태로 빠지지 않도록 조치를 취해야 합니다.

Ceph 의 리소스들을 수정할 수 있는 여러 Ceph 서브커맨드들이 있지만, 이 문서에서는 다루지 않습니다. 클러스터 상태에 대한 더 많은 정보는, [Ceph 문서](https://docs.ceph.com/) 를 확인하세요. [Advanced Configuration 섹션](advanced_configuration.html) 에서, 도움이 될 만한 여러 힌트들을 제공하고 있습니다. 

## 클러스터가 서비스 요청에 응답하지 못함

### 증상
- `ceph` 커맨드가 행 상태로 실행되지 않음
- PersistentVolume 이 생성되지 않음
- 다수의 slow request 
- 다수의 stuck request
- 한 개 이상의 mon 들이 계속 재시작됨

### 확인
현재 Ceph 상태를 확인하기 위해 [rook-ceph-tools pod](toolbox.html) 을 생성합니다. 아래 상화에서, `ceph status` 커맨드는 행 상태로 실행되지 않습니다. CTRL+C 로 멈춰야 합니다.
```sh
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status

ceph status
^CCluster connection interrupted or timed out
```

다른 확인 방법은 mon pod 들이 주기적으로 재시작 되는 것을 확인하는 것입니다. 아래 예시에서 'mon107' pod 은 16분 전 재시작되었습니다.
```sh
kubectl -n rook-ceph get all -o wide --show-all
```
```sh
NAME                                 READY     STATUS    RESTARTS   AGE       IP               NODE
po/rook-ceph-mgr0-2487684371-gzlbq   1/1       Running   0          17h       192.168.224.46   k8-host-0402
po/rook-ceph-mon107-p74rj            1/1       Running   0          16m       192.168.224.28   k8-host-0402
rook-ceph-mon1-56fgm                 1/1       Running   0          2d        192.168.91.135   k8-host-0404
rook-ceph-mon2-rlxcd                 1/1       Running   0          2d        192.168.123.33   k8-host-0403
rook-ceph-osd-bg2vj                  1/1       Running   0          2d        192.168.91.177   k8-host-0404
rook-ceph-osd-mwxdm                  1/1       Running   0          2d        192.168.123.31   k8-host-0403
```

### 해결방법
이 증상은 하나 이상의 Ceph 데몬이 적절한 클러스터 설정으로 구성되지 않았을 떄 발생합니다. 일반적으로 클러스터 CRD 에서 `dataDirHostPath` 값을 지정하지 않은 결과입니다.

`dataDirHostPath` 설정은 호스트에 Ceph 데몬을 위한 설정파일과 데이터를 저장하는 경로를 지정합니다. `/var/lib/rook` 와 같이 지정되며, Cluster CRD 를 다시 배포하거나 Ceph 데몬 (MON, MGR, ODS, RGW) 을 재시작하면 문제 해결에 도움이 됩니다. Ceph 데몬이 재시작 되고 나면, [rook-tool pod](toolbox.html) 를 재시작할 수 있습니다.

## Mon 만 배포됨

### 증상
- Rook 오퍼레이터가 동작 중임
- 하나의 mon 이 시작되거나, 매우 느리게 시작됨 (몇 분 뒤에도)
- crash-collector pod 이 동작하지 않음
- CSI 드라이버 외에 mgr, osd 등 다른 컴포넌트들이 생성되지 않음 

### 확인

오퍼레이터가 클러스터를 시작할 때, 모든 mon 을 실행하기 전에 하나의 mon 을 배포하여 체크합니다. 첫번째 mon 이 정상적이지 않으면, 오퍼레이터는 정상 상태가 될때까지 체크합니다. 첫번째 mon 이 정상적으로 시작되고 나서야 두번째, 세번째 mon 이 시작됩니다. 시작된 mon 들은 바로 쿼럼을 형성하지는 않으며, 오케스트레이션은 블로킹 된 상태가 됩니다.

crash-collector pod 은 mon 들이 처음 쿼럼을 형성할 때 까지 시작되지 않습니다.

mon 이 쿼럼을 형성하지 못하는 이유는 여러가지가 있을 수 있습니다.
- 오퍼레이터 pod 이 mon pod (들)과 연결을 맺지 못함. 네트워크 설정이 잘못되어 있을 수 있습니다.
- 하나 이상의 pod 이 실행 상태이지만, 쿼럼을 형성하지 못한다는 로그가 오퍼레이터에 존재 
- mon 이 이전 설치 떄 사용한 설정을 사용하고 있음. 이전 클러스터를 [클린업 하기 위한 가이드](/ceph_storage/cleanup.html#delete-the-data-on-hosts) 를 참고합니다.
- 방화벽이 mon 이 쿼럼을 형성하기 위한 포트를 막고 있음. 6789, 3300 포트가 열려있는지 확인합니다. 자세한 정보는 [Ceph 네트워크 가이드](https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/)를 참고합니다.
- 컴포넌트간 MTU 가 일치하지 않을 수 있습니다. 쿠버네티스 CNI 플러그인이나 호스트가 점포 프레임 (MTU 9000) 이 활성화 되어 있으면, Ceph 은 네트워크 대역폭을 높이기 위해 더 큰 패킷을 사용합니다. 네트워크를 사용하는 다른 부분이 점포 프레임을 지원하지 않으면, 예상치 못한 패킷 유실이 있을 수 있습니다.

#### 오퍼레이터가 mon 에 연결하지 못함
먼저 오퍼레이터가 mon 들에 접근할 수 있는지 로그를 확인합니다.
```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator
```

오퍼레이터가 mon 에 연결을 시도하다가 타임아웃이 된 로그를 확인할 수 있습니다. 마지막 커맨드는 `ceph mon_status` 였고, 5분 뒤 타임아웃 메시지가 나타납니다.
```bash
2018-01-21 21:47:32.375833 I | exec: Running command: ceph mon_status --cluster=rook --conf=/var/lib/rook/rook-ceph/rook.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/442263890
2018-01-21 21:52:35.370533 I | exec: 2018-01-21 21:52:35.071462 7f96a3b82700  0 monclient(hunting): authenticate timed out after 300
2018-01-21 21:52:35.071462 7f96a3b82700  0 monclient(hunting): authenticate timed out after 300
2018-01-21 21:52:35.071524 7f96a3b82700  0 librados: client.admin authentication error (110) Connection timed out
2018-01-21 21:52:35.071524 7f96a3b82700  0 librados: client.admin authentication error (110) Connection timed out
[errno 110] error connecting to the cluster
```
인증 에러로 로그가 나타날 수 도 있지만, 실제 이슈는 타임아웃일 수 있습니다.

#### 해결방법
오퍼레이터 로그에서 타임아웃 로그를 봤다면, mon pod 들이 Running 상태인지 확인합니다. mon pod 들이 모두 Running 상태라면, 오퍼레이터 pod 과 mon pod 간의 네트워크를 확인합니다. 대부분의 경우 CNI 플러그인이 정상적으로 설정되지 않을 때 나타납니다.

네트워크 연결을 확잏나려면, 
- mon 엔드포인트를 얻습니다.
- 오퍼레이터 pod 에서 curl 툴로 mon 연결을 확인합니다.

예를들어, 아래 커맨드는 오퍼레이터에서 첫번째 mon 을 curl 로 확인합니다.
```bash
kubectl -n rook-ceph exec deploy/rook-ceph-operator -- curl $(kubectl -n rook-ceph get svc -l app=rook-ceph-mon -o jsonpath='{.items[0].spec.clusterIP}'):3300 2>/dev/null
```
```bash
ceph v2
```

만약 "ceph v2" 가 콘솔에 나타나면, 연결이 성공적인 것입니다. 응답이 없다면, 연결이 정상적으로 이루어지지 않는 것입니다.

#### mon pod 비정상
다음으로 mon pod 이 정상적으로 시작되었는 지 확인할 수 있습니다.
```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mon
```
```bash
NAME                                READY     STATUS               RESTARTS   AGE
rook-ceph-mon-a-69fb9c78cd-58szd    1/1       CrashLoopBackOff     2          47s
```

mon pod 이 정상적으로 시작되지 않았다면, 원인 파악을 위해서 mon pod 의 상태나 로그를 확인해야 합니다. pod 이 crash loop backoff 상태라면, pod 을 describe 하여 원인을 확인해볼 수 있습니다.
```bash
# 이미 배포된 키링과 일치하지 않아 pod 이 중단된 경우
kubectl -n rook-ceph describe pod -l mon=rook-ceph-mon0
```
```bash
...
   Last State:    Terminated
     Reason:    Error
     Message:    The keyring does not match the existing keyring in /var/lib/rook/rook-ceph-mon0/data/keyring.
                   You may need to delete the contents of dataDirHostPath on the host from a previous deployment.
...
```
다음 섹션에서 노드의 `dataDirHostPath` 클린업 방법을 확인하세요

#### 해결방법
이 증상은 호스트의 로컬 디렉토리가 정상적으로 삭제되지 않았을 때 다시 Rook 클러스터를 재배포하면 발생합니다. 이 디렉토리는 클러스터 CRD 의 `dataDirHostPath` 이며, `/var/lib/rook` 으로 기본 설정되어 있습니다. 이 이슈를 수정하기 위해서는, Rook 의 모든 컴포넌트를 삭제하고 모든 노드에서 `/var/lib/rook` 디렉토리의 모든 파일들을 삭제합니다. 그러고 나서 새로운 클러스터 CRD 를 배포하게 되면, rook operator 는 모든 pod 들을 예상한대로 배포할 것입니다.

> **중요: `dataDirHostPath` 폴더의 모든 파일을 삭제하면 스토리지 클러스터가 모두 삭제됩니다.**

더 많은 정보는 [클린업 가이드](/ceph_storage/cleanup.html) 를 참고합니다.

## PVC 가 Pending 상태로 지속됨

### 증상
- Rook 스토리지 클래스로 PVC 를 생성했을떄, Pending 상태로 계속 남아있음

Wordpress 예제에서, PVC 가 Pending 상태로 남아있던 것을 이미 보았을 것입니다.
```bash
kubectl get pvc
```
```bash
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Pending                                      rook-ceph-block   8s
wp-pv-claim      Pending                                      rook-ceph-block   16s
```

### 확인
PVC 가 Pending 상태로 남는 주요 원인은 아래 두가지입니다.
- 클러스터에 OSD 가 없음
- CSI 프로비저너 Pod 이 동작하지 않고 있거나, 스토리지를 프로비저닝하도록 요청을 받고 응답하지 않음

#### OSD 가 있는지 확인
클러스터에 OSD 가 있는지 확인하기 위해서, [Rook Toolbox](/ceph_tools/toolbox.html) 에 연결하여 `ceph status` 커맨드를 입력합니다. 적어도 하나 이상의 OSD 가 `up` 그리고 `in` 상태로 존재해야 합니다. 최소 OSD 갯수는 스토리지 클래스를 위해 만들어진 pool 의 `replicated.size` 설정에 따라 다릅니다. "test" 클러스터에서는, 하나의 OSD 만 필요합니다(`storageclass-test.yaml` 참고). 프로덕선 스토리지 클래스에서는 (`storageclass.yaml`) 세개의 OSD 가 필요합니다.

```bash
ceph status
```
```bash
 cluster:
   id:     a0452c76-30d9-4c1a-a948-5d8405f19a7c
   health: HEALTH_OK

 services:
   mon: 3 daemons, quorum a,b,c (age 11m)
   mgr: a(active, since 10m)
   osd: 1 osds: 1 up (since 46s), 1 in (since 109m)
```

#### OSD 준비 로그
예상한 갯수의 OSD 가 보이지 않는다면, 왜 OSD 가 생성되지 못했는지 확인해 봅니다. Rook 이 OSD 를 설정하는 각 노드에서, "osd prepare" pod 이 생성되는 것을 확인했을 것입니다.

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-osd-prepare
```
```bash
NAME                                 ...  READY   STATUS      RESTARTS   AGE
rook-ceph-osd-prepare-minikube-9twvk   0/2     Completed   0          30m
```
[osd pod 들이 시작되지 않음](#osd-pod-들이-시작되지-않음) 섹션을 참고합니다.

#### CSI 드라이버
CSI 드라이버가 요청에 응답하지 못했을 수 있습니다. 프로비저닝 하는 동안 CSI 프로비저너 Pod 에 어떤 에러 로그가 있었는지 확인합니다.

두개의 프로비저너 Pod 이 존재할 것입니다:
```bash
kubectl -n rook-ceph get pod -l app=csi-rbdplugin-provisioner
```

각 Pod 들의 로그를 확인하세요, 이중 하나가 리더이고, 실제로 요청에 대한 응답을 수행합니다.
```bash
kubectl -n rook-ceph logs csi-cephfsplugin-provisioner-d77bb49c6-q9hwq csi-provisioner
```

더 많은 이슈는 [CSI 트러블슈팅 가이드](/ceph_tools/csi_common_issues.html) 을 참고하세요.

#### 오퍼레이터가 응답하지 않음
OSD 가 `up`, `in` 상태인 것을 확인했다면, 다음 절차는 오퍼레이터가 응답하는지 확인하는 것입니다. PVC 가 생성되고 난 후에 오퍼레이터 pod 에 로그가 발생하는지 확인합니다. 오퍼레이터가 블록 이미지를 프로비저닝 하는 데 요청을 보여주지 않는다면, 다른 작업 때문에 stuck 되었을 수 있습니다. 이럴때는, 오퍼레이터 pod 을 재시작 해줍니다.

### 해경방법
"osd prepare" 로그가 왜 OSD 가 생성되지 않았는지에 대한 실마리를 주지 않는다면, `cluster.yaml` 설정을 확인하세요. 주된 설정 실수는 아래와 같습니다:
- `useAllDevices: true` 설정이 되어 있으면, Rook 은 노드의 로컬 디바이스를 찾습니다. 디바이스가 노드에 없으면, OSD 는 생성되지 않습니다.
- `useAllDevices: fales` 설정이 되어 있으면, OSD 는 `deviceFilter` 가 명시되어 있어야 생성됩니다.
- 노드의 로컬 디바이스만 Rook 에 의해 설정됩니다. 다시말해, `/dev` 경로 아래에 디바이스가 보여야 합니다.
  - 디바이스는 파티션이나 파일시스템을 가지고 있어서는 안됩니다. Rook 은 raw 디바이스만 설정합니다. 파티션은 아직 지원되지 않습니다.


## OSD Pod 들이 시작되지 않음
### 증상
- OSD 이 시작에 실패함
- 클러스터를 클린업 한 후 다시 클러스터를 시작함

### 확인
OSD 가 시작할 때, 디바이스나 디렉토리가 설정됩니다. 설정에 에러가 있으면, pod 은 CrashLoopBackoff 상태로 시작되지 않습니다. 실패 원인을 조회하려면 osd pod 로그를 확인합니다.
```bash
$ kubectl -n rook-ceph logs rook-ceph-osd-fl8fs
...
```

주된 실패의 원인은 테스트 클러스터를 다시 배포했을 때, 기존 배포 떄의 상태가 남아있어 생깁니다. 만약 클러스터의 노드가 많다면, monitor 들이 쿼럼을 형성해 따시 시작할 수 있을 수도 있습니다. 하지만, osd pod 들은 기존 상태 때문에 시작하지 못할 것입니다. OSD pod 로그를 확인한다면, 그에 대한 로그를 확인할 수 있을 것입니다.
```bash
$ kubectl -n rook-ceph logs rook-ceph-osd-fl8fs
```
```bash
...
2017-10-31 20:13:11.187106 I | mkfs-osd0: 2017-10-31 20:13:11.186992 7f0059d62e00 -1 bluestore(/var/lib/rook/osd0) _read_fsid unparsable uuid
2017-10-31 20:13:11.187208 I | mkfs-osd0: 2017-10-31 20:13:11.187026 7f0059d62e00 -1 bluestore(/var/lib/rook/osd0) _setup_block_symlink_or_file failed to create block symlink to /dev/disk/by-partuuid/651153ba-2dfc-4231-ba06-94759e5ba273: (17) File exists
2017-10-31 20:13:11.187233 I | mkfs-osd0: 2017-10-31 20:13:11.187038 7f0059d62e00 -1 bluestore(/var/lib/rook/osd0) mkfs failed, (17) File exists
2017-10-31 20:13:11.187254 I | mkfs-osd0: 2017-10-31 20:13:11.187042 7f0059d62e00 -1 OSD::mkfs: ObjectStore::mkfs failed with error (17) File exists
2017-10-31 20:13:11.187275 I | mkfs-osd0: 2017-10-31 20:13:11.187121 7f0059d62e00 -1  ** ERROR: error creating empty object store in /var/lib/rook/osd0: (17) File exists
```

### 해결방법
기존 설정이 존재해서 에러가 나타났다면, 기존에 사용하던 로컬 디렉토리가 제대로 삭제되지 않은 것입니다. 이 디렉토리는 `dataDirHostPath` 설정을 통해 설정되는 디렉토리이며, 기본값은 `/var/lib/rook` 입니다. 이 이슈를 해결하기 위해서는, 모든 호스트에서 `/var/lib/rook` 디렉토리의 파일들을 삭제해야 합니다. (혹은 `dataDirHostPath` 에 설정된 디렉토리) 이후 신규 클러스터갑 배포된다면, rook 오퍼레이터는 모든 pod 들을 예상한대로 실행할 것입니다.

## OSD Pod 들이 디바이스에 생성되지 않음
### 증상
- 클러스터에서 OSD pod 이 시작되지 않음
- 클러스터 CRD 에 명시 되었으나 OSD 와 함께 디바이스가 설정되지 않음
- 개별 디바이스별로 여러 Pod 이 생성되지 않고 각 노드별로 하나의 OSD 만 생성됨 

### 확인
먼저, CRD 에 디바이스가 올바르게 지정되어 있는지 확인합니다. [Cluster CRD](/ceph_storage/cluster_crd.html) 는 Rook 에 의해 디바이스가 어떻게 사용될 것인지 명시하는 여러 방법을 제공합니다.
- `useAllDevices: true`: 가능한 모든 디바이스를 사용합니다.
- `deviceFilter`: 정규 표현식에 매치되는 모든 디바이스를 사용합니다.
- `devices`: 각 노드별로 사용할 디바이스를 명시적으로 지정합니다.

두번째로, Rook 이 가용한 디바이스가 없다고 판단하면, (이미 파티션이나 파일시스템이 있음), Rook 은 해당 디바이스를 사용하지 않습니다. OSD 가 시작되지 않으면, 이 이슈일 가능성이 있습니다. 디바이스가 스킵되었는지 확인하려면, OSD prepare 로그를 확인합니다. osd prepare pod 이 `completed` 상태가 된 것은 정상적인 상태입니다. job 이 완료되고 나면, 로그를 확인해야 할 케이스를 대비하여 pod 을 남겨놓습니다.

```bash
# 클러스터 안에서 osd prepare pod 조회
kubectl -n rook-ceph get pod -l app=rook-ceph-osd-prepare
```
```bash
NAME                                   READY     STATUS      RESTARTS   AGE
rook-ceph-osd-prepare-node1-fvmrp      0/1       Completed   0          18m
rook-ceph-osd-prepare-node2-w9xv9      0/1       Completed   0          22m
rook-ceph-osd-prepare-node3-7rgnv      0/1       Completed   0          22m
```
```bash
# "provision" 컨테이너에서 노드의 로그를 조회합니다.
kubectl -n rook-ceph logs rook-ceph-osd-prepare-node1-fvmrp provision
[...]
```
로그에서 유용한 라인들:
```bash
# Rook 이 파티션이나 파일시스템을 발견하면 디바이스는 스킵됩니다.
2019-05-30 19:02:57.353171 W | cephosd: skipping device sda that is in use
2019-05-30 19:02:57.452168 W | skipping device "sdb5": ["Used by ceph-disk"]

# ceph 이 사용할 수 없는 디스크에 대한 메시지:
Insufficient space (<5GB) on vgs
Insufficient space (<5GB)
LVM detected
Has BlueStore device label
locked
read-only

# 디바이스가 설정됨
2019-05-30 19:02:57.535598 I | cephosd: device sdc to be configured by ceph-volume

# 각 디바이스마다 로그에 리포트가 프린트됩니다.
2019-05-30 19:02:59.844642 I |   Type            Path                                                    LV Size         % of device
2019-05-30 19:02:59.844651 I | ----------------------------------------------------------------------------------------------------
2019-05-30 19:02:59.844677 I |   [data]          /dev/sdc                                                7.00 GB         100%
```

### 해결방법
커스텀 리소스를 올바른 방법으로 수정하거나, 디바이스에서 파티션이나 파일시스템을 클린업합니다. 이전 설치로부터 디바이스를 클린업하려면 [가이드](/ceph_storage/cleanup.html#zapping-devices)를 참고하세요.

설정이 업데이트되거나 디바이스가 클린업 되고 나면, 오퍼레이터를 재시작함으로서 디바이스를 다시 분석하도록 트리거합니다. 오퍼레이터가 재시작 될때마다, 원하는 상태로 디바이스가 설정되었는지 확인합니다. 대부분의 시나리오에서 오퍼레이터는 OSD 를 자동으로 배포하지만, 오퍼레이터가 자동으로 감지하지 못하는 시나리오에 대해서는 재시작 시 대부분 해결됩니다.

```bash
# 디바이스가 설정되었는지 확인하기 위해서 오퍼레이터를 재시작합니다. 새로운 pod 은 자동으로 다시 시작됩니다.
kubectl -n rook-ceph delete pod -l app=rook-ceph-operator
[...]
```

## 노드 리부팅 이후 노드가 Hang 상태로 지속됨

이 이슈는 Rook v1.3 이후 버전에서 수정되었습니다.

### 증상
- `reboot` 명령 이후에 노드가 온라인 상태로 돌아오지 않음
- 전원 버튼만이 해결책

### 확인
Ceph persistent volume 을 사용중인 pod 의 노드에서 아래 명령을 입력합니다.
```bash
mount | grep rbd
```
```bash
# _netdev 마운트 옵션이 없고, cephfs 도 동일
# OS 는 PV 가 네트워크를 통해 마운트 되었는지 알지 못합니다.
/dev/rbdx on ... (rw,relatime, ..., noquota)
```

리부팅 커맨드가 실행되면, 네트워크 인터페이스가 디스크 unmount 전에 종료됩니다. 이 때문에 Ceph persistent volume 의 umount 는 계속 실패하며 아래의 에러와 함께 계속 재시도합니다.
```bash
libceph: connect [monitor-ip]:6789 error -101
```

### 해결방법
노드가 리부팅되기 전에 [drain](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) 되어야 합니다. drain 이 완료된 이후에야 노드가 리부팅 될 수 있습니다.

`kubectl drain` 커맨드는 자동으로 노드를 스케줄링 되지 않게 하기 때문에 (`kubectl cordon`), 온라인으로 돌아오고 난 이후에는 uncordon 되어야 합니다.

#### 노드 Drain:
```bash
$ kubectl drain <node-name> --ignore-daemonsets --delete-local-data
```

#### 노드 Uncordon
```bash
$ kubectl uncordon <node-name>
```

## 여러 Cephfs 를 커널 4.7 미만 버전에서 사용하려고 함

### 증상
- 하나 이상의 공유 파일시스템 (Cephfs) 를 클러스터에 만듬
- pod 이 **처음** 생성된 공유 파일시스템 외에 다른 공유 파일시스템을 마운트하려 합니다.
- pod 이 의도한 파일시스템 대신 마운트된 첫 번째 파일시스템을 잘못 가져옵니다.

### 해결방법
이 문제를 해결할 유일한 방법은 커널을 `4.7` 이상으로 업그레이드 하는 것입니다. 이름으로 파일시스템을 선택하는 마운트 플래그가 `4.7` 에서 추가되었습니다.

다중 공유 파일시스템에 대한 더 자세한 커널 버전 요구사항은 [파일시스템 - 커널 버전 요구사항](/ceph_storage/shared_filesystem.html#kernel-version-requirement) 섹션을 참고하세요.

## 모든 Ceph 데몬들의 로그 레벨을 Debug 로 변경

모든 Ceph 데몬에 대해 동시에 로그 레벨을 변경할 수 있습니다. 이를 위해서, toolbox pod 이 동작 중인지 확인하고, 0-20 까지 원하는 로그 레벨을 정합니다. 모든 서비시스템의 로그레벨과 기본값을 [다음](https://docs.ceph.com/en/latest/rados/troubleshooting/log-and-debug/#ceph-subsystems) 가이드에서 확인 가능합니다. 로그 레벨을 높일수록 더 많은 로그를 발생시킵니다.

로그 레벨 1을 원하면, 아래 커맨드를 입력합니다.
```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- set-ceph-debug-level 1
```

결과:
```bash
ceph config set global debug_context 1
ceph config set global debug_lockdep 1
...
...
```

디버깅이 완료되었다면, 아래 커맨드로 기본값으로 돌릴 수 있습니다.
```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- set-ceph-debug-level default
```

## 특정 Ceph 데몬에 대해 file 로 로그를 쓰도록 변경

쿠버네티스 로그만으로는 다양한 원인을 파악하기에는 한계가 있을 수 있습니다.
- 모든 사람들에게 쿠버네티스의 로깅 방법이 로그를 찾는 데 적절하지는 않으며, 전통적인 디렉토리 내에 로그를 남기는 방법이 편리할 수 있습니다.
- 로그 엔진의 버퍼를 넘어서면 로그는 로테이트 되어 사라집니다.

그래서 각각의 데몬에서, 로깅이 활성화 되어 있다면 `dataDirHostPath` 설정이 로그를 저장하기 위해서 사용됩니다. Rook 는 `dataDirHostPath` 를 각 pod 마다 bind mount 합니다. 여러분이 `mon.a` 데몬만 로깅을 활성화 한다고 가정한다면, toolbox 내에서 아래 커맨드를 사용합니다. 
```bash
ceph config set mon.a log_to_file true
```

이 커맨드는 파일시스템에 로깅하도록 활성화시키며, 로그를 `${dataDirHostPath}/$NAMESPACE/log` 디렉토리에서 확인할 수 있습니다. 기본 설정을 사용한다면, `/var/lib/rook/rook-ceph/log` 에서 확인 가능합니다. pod 을 재시작하지 않아도 즉시 반영됩니다.

파일로의 로깅을 비활성화하려면, `log_to_file` 을 `false` 로 설정합니다.


## RBD 디바이스를 사용하는 워커 노드가 Hang 상태로 지속됨
### 증상
- RBD 디바이스 중 하나에 I/O 가 없음 (`/dev/rbd*` 혹은 `/dev/nbd*`)
- 모든 워커 노드가 hang 됨

### 확인
위 현상은 아래와 같은 상황에서 발생합니다.
- 문제가 있는 RBD 디바이스와 OSD 가 동시에 있을 떄 
- 디바이스가 XFS 파일시스템일 떄 

문제가 발생했을 때, `dmesg` 커맨드로 로그를 확인할 수 있습니다.
```bash
dmesg
```
```bash
...
[51717.039319] INFO: task kworker/2:1:5938 blocked for more than 120 seconds.
[51717.039361]       Not tainted 4.15.0-72-generic #81-Ubuntu
[51717.039388] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
...
```

이 문제는 `hung_task` 문제라고도 불리며, 커널 내에 데드락이 발생했다는 것을 의미합니다. 더 많은 정보는, [해당 이슈 코멘트](https://github.com/rook/rook/issues/3132#issuecomment-580508760) 를 확인하세요.

### 해결방법
아래 두 방법을 통해 문제를 해결할 수 있습니다.
- Linux Kernel: [다음 커밋](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8d19f1c8e1937baf74e1962aae9f90fa3aeab463) 에서 해당 마이너 피처가 소개되었고, 커널 v5.6 에 포함되었습니다.
- Ceph: 커널 v5.6 릴리즈 이후에 위에서 언급된 커널 피처를 사용하는 수정사항에 대한 논의가 이루어질 것입니다.

XFS 파일시스템 대신 ext4 파일시스템을 사용하여 위 문제를 회피할 수 있습니다. 스토리지 클래스 리소스에 `csi.storage.k8s.io/fstype` 에 파일시스템을 명시할 수 있습니다.

## Too few PGs per OSD 메시지가 나타남
### 증상
- `ceph status` 커맨드를 입력했을 때, "too few PGs per OSD" 경고 메시지가 나타남
```bash
ceph status
```
```bash
 cluster:
   id:     fd06d7c3-5c5c-45ca-bdea-1cf26b783065
   health: HEALTH_WARN
           too few PGs per OSD (16 < min 30)
```

### 해결방법
[이 문서](https://docs.ceph.com/docs/master/rados/operations/health-checks#too-few-pgs) 에서 경고의 의미를 조회할 수 있습니다. 더 많은 정보는 [이 블로그](https://ceph.com/community/new-luminous-pg-overdose-protection/) 를 참고하세요. pool 의 적절한 `pg_num` 을 알고 싶거나 이 값을 변경하고 싶으면 [Configuring Pools](/ceph_tools/advanced_configuration.html#configuring-pools) 섹션을 참고하세요.

## LV 백엔드 PVC 를 사용하는 OSD 에서 LVM 메타데이터가 손상됨
### 증상
LV 기반 PVC 에는 OSD 에 심각한 결함이 있습니다. 호스트와 OSD 컨테이너에서 동시에 LVM 메타데이터를 수정하면 메타데이터가 꺠질 수 있습니다. 예를 들어, 운영자가 호스트에서 LVM 메타데이터를 수정하는 와중에, OSD 초기화 프로세스가 컨테이너 내에서 일어날 수 있습니다. 또한, `lvmetad` 가 동작 중이라면, 더 자주 발생할 수 있습니다. 이 경우, OSD 컨테이너의 LVM 메타데이터 변경은 한동안 호스트의 LVM 메타데이터 캐시에 반영되지 않습니다.

LVM 위에서 OSD 를 동작시키기로 결정했다면, 이슈 발생 가능성을 줄이기 위해 아래 사항들을 유의해주세요.

### 해결방법
- `lvmetad` 를 비활성화합니다.
- 호스트로부터 LV 를 설정하지 않습니다. 그리고, LV 뒷단의 VG 들과 물리적 볼륨들을 수정하지 않습니다.
- `storageClassDeviceSets` 의 `count` 필드가 증가하지 않도록 하고, OSD 를 동시에 지원하는 새 LV 를 생성합니다.

`sudo lvs -o lv_name,lv_tags` 명령을 사용해서 위의 태그가 존재하는지 여부를 알 수 있습니다. 만약 OSD lv_tags 에 해당하는 LV 에서 `lv_tag` 필드가 비어있다면 이 OSD 에 문제가 발생한 것입니다. 이 경우, OSD 를 시작하기 전에 이 [OSD 를 제거](/osd_management.html#remove-an-osd) 하거나 교체하세요.

이 문제는 OSD 컨테이너가 LVM 메타데이터를 더이상 건드리지 않기 떄문에 신규로 생성된 LV 백엔드 PVC 에서는 발생하지 않습니다. 이미 lvm 모드로 만들어진 OSD 는 Rook 가 업그레이드 되더라도 계속 동작합니다. 그러나, 위의 이슈들 떄문에 raw 모드 OSD 가 권장됩니다. 이미 존재하는 OSD 들을 raw 모드로 변경할 수 있습니다. [OSD 제거](/osd_management.html#remove-an-osd) 와 [OSD 추가](/osd_management.html#add-an-osd-on-a-pvc) 문서를 확인하세요

## OSD Prepare Job 이 low aio-max-nr 설정으로 실패함
만약 커널이 low [aio-max-nr 설정](https://www.kernel.org/doc/Documentation/sysctl/fs.txt) 으로 구성되어 있다면, OSD prepare job 이 아래 에러와 함께 실패할 것입니다.
```bash
exec: stderr: 2020-09-17T00:30:12.145+0000 7f0c17632f40 -1 bdev(0x56212de88700 /var/lib/ceph/osd/ceph-0//block) _aio_start io_setup(2) failed with EAGAIN; try increasing /proc/sys/fs/aio-max-nr
```

이 에러를 해결하기 위해서는, sysctl 설정에서 `fs.aio-max-nr` 값을 늘려 주어야 합니다. (`/etc/sysctl.conf`) 여러분이 가장 선호하는 설정 관리 시스템으로 변경할 수 있습니다.

대안으로, [DaemonSet](https://github.com/rook/rook/issues/6279#issuecomment-694390514) 을 사용하여 모든 노드에 설정을 적용할 수 있습니다.


## 예상하지 않은 파티션이 생성됨
### 증상
**Rook 버전 v1.6.0-v1.6.7 을 사용하는 사용자들은 예기치 않게 무작위로 나타나는 파티션에 원하지 않는 OSD 를 확인할 수도 있으며, 이로 인해 기존의 OSD 가 손상될 수 있습니다.**

Ceph OSD 에 의해 사용되는 호스트의 디스크에 예기치 못한 파티션이 만들어 질 수 있습니다. SSD 보다 HDD 에 더 많이 나타나며, 875GB 보다 높은 디스크에서 더 발생합니다. `lsblk`, `blkid`, `udevadm`, `parted` 같은 툴들은 이런 파티션에 대해 파티션 타입을 보여주지 않을 수 있습니다. `blkid` 신규 버전은 타입을 "atari" 로 인식합니다.

이런 이슈는 리눅스 커널의 Atari 파티션 (AHDI 로 구분되는) 지원입니다. Atari 파티션은 다른 파티션 유형에 비해 매우 완화된 사양을 가지고 있으며, 디스크에 작성된 임의의 데이터가 리눅스 커널의 Atari 파티션으로 표시될 수 있습니다. Ceph 의 블루스토어 OSD 는 커널에 Atari 파티션으로 나타날 수 있는 디스크에 데이터를 쓸 확률이 높습니다.

아래는 Atari 파티션이 나타난 특정 노드의 `lsblk` 아웃풋을 나타냅니다. `sdX1` 은 팬텀 파티션으로 나타나지 않으며, 모든 디스크에는 `sdX2` 가 48GB 로 나타납니다. `sdX3` 는 가변 크기이며 항상 존재하지는 않을 수 있습니다. 드물지만 `sdX4` 가 나타날 수 있습니다.

```bash
# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sdb      8:16   0     3T  0 disk
├─sdb2   8:18   0    48G  0 part
└─sdb3   8:19   0   6.1M  0 part
sdc      8:32   0     3T  0 disk
├─sdc2   8:34   0    48G  0 part
└─sdc3   8:35   0   6.2M  0 part
sdd      8:48   0     3T  0 disk
├─sdd2   8:50   0    48G  0 part
└─sdd3   8:51   0   6.3M  0 part
```

더 많은 정보는 https://github.com/rook/rook/issues/7940 에서 확인할 수 있습니다. 

### 해결방법

#### corruption 으로부터 복구하기
여러분이 Rook v1.6 을 사용중이라면, Atari 파티션으로 인해 발생하는 OSD corruption 사고를 피하기 위해 v1.6.8 이상 버전을 사용해야 합니다.

기존에는 `deviceFilter: ^sd[a-z]+$` 를 사용하는 것으로 해결방안을 제시했지만, 그래도 예기치 못한 파티션이 발생합니다. Rook 은 새 OSD 를 만드는 것을 멈출 뿐이며, Atari 파티션 문제를 인식하지 못하는 `ceph-volume` 과 관련된 문제는 해결되지 않았습니다. 이 해결 방법을 사용한 사용자는 앞으로도 OSD 오류가 발생할 위험이 있습니다.

이 이슈를 해결하기 위해서, 즉시 v1.6.8 로 업데이트를 해야 합니다. 업데이트 이후, 미래에 생성되는 OSD 들에 대해서는 corruption 이 발생해선 안됩니다. 이후 클러스터가 Healty 상태로 돌아가면, 이미 손상된 디스크에 대해서 하나씩 [OSD 제거](/osd_management.html#remove-an-osd) 를 진행합니다.

예시와 같이, 여러분은 두개의 예상치 못한 파티션(`/dev/sdb2`, `/dev/sdb3`)을 가진 `/dev/sdb` 디바이스를 가지고 있고, 두번째 손상 디스크 `/dev/sde` 는 하나의 예상하지 못한 파티션 (`/dev/sde2`) 를 가지고 있습니다.

1. 먼저, `/dev/sdb`, `/dev/sdb2`, `/dev/sdb3` 와 관련된 OSD 를 삭제합니다. OSD 하나일 수 도 있고, 세 개의 OSD 가 연관이 있을 수 도 있습니다. [OSD Management](/osd_management.html) 섹션을 참고합니다.
2. 디스크 파티션의 첫번째 섹터를 삭제하기 위해 `dd` 명령을 사용합니다.
    - `dd if=/dev/zero of=/dev/sdb2 bs=1M`
    - `dd if=/dev/zero of=/dev/sdb3 bs=1M`
    - `dd if=/dev/zero of=/dev/sdb bs=1M`
3. 그러고 나서, 신규 OSD 를 준비하기 위해 `/dev/sdb` 를 클린업합니다. [teardown 문서](/ceph_storage/cleanup.html#zapping-devices)를 참고하세요.
4. `/dev/sdb` 디바이스에 신규 OSD 를 배포하기 위해 Rook 오퍼레이터를 스케일업합니다. 이를 통해 Ceph 은 다음 OSD 가 제거되는 동안 데이터 복구 및 복제를 위해 `/dev/sdb` 를 사용할 수 있습니다.
5. 1-4 스텝을 `/dev/sde` 와 `/dev/sde2` 에 대해 반복합니다. 

Rook-Ceph 이 중요한 데이터를 가지고 있지 않다면, Rook 을 완전히 삭제하고 v1.6.8 이상으로 재설치 하는 것이 더 간단할 수 있습니다.