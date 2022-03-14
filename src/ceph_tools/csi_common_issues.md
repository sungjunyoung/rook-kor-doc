# CSI Common Issues

Ceph CSI 드라이버를 사용해 볼륨을 프로비저닝 할 때, 다음과 같은 여러 이유로 이슈가 발생할 수 있습니다:
- CSI pod 와 ceph 간의 통신 이슈
- 클러스터 health 이슈
- Slow operation
- 쿠버네티스 이슈
- Ceph-CSI 설정이나 버그

아래의 트러블슈팅 스텝이 이슈들을 확인하는 데 도움을 줄 것입니다.

### Block (RBD)
블록 볼륨 (RWO) 을 마운트 한다면, Ceph 에서 `RBD` 볼륨이라고 불립니다. 블록 볼륨에 대한 이슈가 있다면 아래의 RBD 섹션을 참고하세요.

### Shared Filesystem (Cephfs)
공유 파일시스템 (RWX) 를 마운트 한다면, Ceph 에서는 `Cephfs` 라고 부릅니다. 공유 파일시스템에 대한 이슈는 CephFS 섹션을 참고하세요.

## Network Connectivity
mon 은 가장 먼저 확인해야 할 중요한 컴포넌트입니다. Service 리소스로부터 mon 엔드포인트를 확인합니다.
```bash
kubectl -n rook-ceph get svc -l app=rook-ceph-mon
```
```bash
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mon-a   ClusterIP   10.104.165.31   <none>        6789/TCP,3300/TCP   18h
rook-ceph-mon-b   ClusterIP   10.97.244.93    <none>        6789/TCP,3300/TCP   21s
rook-ceph-mon-c   ClusterIP   10.99.248.163   <none>        6789/TCP,3300/TCP   8s
```

CephCluster 커스텀 리소스에서 호스트 네트워크가 활성화 되어 있다면, mon 이 동작하고 있는 노드의 IP 를 확인해야 합니다.

`clusterIP` 는 mon IP 이고, `3300` 은 Ceph-CSI 가 ceph 클러스터에 연결하기 위한 포트입니다. 이 엔드포인트는 CSI 드라이버 뿐만 아니라 모든 클라이언트에서 접근이 가능해야 합니다.

만약 PVC 를 프로비저닝 하는 데 이슈가 발생했다면, 프로비저너 pod 으로부터 네트워크 연결을 확인해야 합니다.
- CephFS PVC 는, `csi-cephfsplugin-provisioner` pod 의 `csi-cephfsplugin` 컨테이너를 확인합니다.
- Block PVC 는, `csi-rbdplugin-provisioner` pod 의 `csi-rbdplugin` 컨테이너를 확인합니다.

이중화를 위해 유형별로 두 개의 프로버지너 Pod 이 있습니다. 모든 Pod 의 연결을 테스트 해야 합니다.

프로비저너 Pod 에 연겨하고, 아래와 같이 mon 엔드포인트로의 연결을 확인합니다.
```bash
# Connect to the csi-cephfsplugin container in the provisioner pod
kubectl -n rook-ceph exec -ti deploy/csi-cephfsplugin-provisioner -c csi-cephfsplugin -- bash

# Test the network connection to the mon endpoint
curl 10.104.165.31:3300 2>/dev/null
ceph v2
```

"ceph v2" 응답을 확인했다면, 연결은 성공적인 것입니다. 응답이 없다면, ceph 클러스터 연결에 문제가 있는 것입니다.

모든 monitor IP 와 포트에 대해 연결을 확인해야 합니다.

## Ceph Health
Ceph 클러스터의 Health 상태가 PVC 를 생성하거나 마운트 하는 데 이슈를 야기할 수 있습니다. [Toolbox](/ceph_tools/toolbox.html) 를 통해 Ceph 클러스터에 연결하여 클러스터 상태를 확인합니다.
```bash
ceph health detail
```
```bash
HEALTH_OK
```

## Slow Operation
클러스터의 slow ops 상태도 이슈를 발생시킬 수 있습니다. toolbox 에서 ceph 클러스터에 slow ops 가 발생하는지 확인합니다.
```bash
ceph -s
```
```bash
cluster:
 id:     ba41ac93-3b55-4f32-9e06-d3d8c6ff7334
 health: HEALTH_WARN
         30 slow ops, oldest one blocked for 10624 sec, mon.a has slow ops
```

Ceph 클러스터가 healty 상태가 아니라면, 다음을 확인합니다.
- Ceph mon 에러 로그
- OSD 에러 로그
- 디스크 상태
- 네트워크 상태

## Ceph Troubleshooting
### Check if the RBD Pool exists
ceph 클러스터에 `storageclass.yaml` 에 정의한 pool 이 만들어 졌는지 확인해야 합니다.

`storageclass.yaml` 에 지정한 pool 이름이 `replicapool` 이라고 합시다. toolbox 에서 아래와 같이 존재 여부를 확인할 수 있습니다.
```bash
ceph osd lspools
```
```bash
1 device_health_metrics
2 replicapool
```

pool 이 리스트에 없다면, `CephBlockPool` 커스텀 리소스를 생성합니다. 이미 pool 을 생성했다면, Rook 오퍼레이터의 에러 로그를 확인합니다.

### Check if the Filesystem exists
공유 파일시스템 (CephFS) 에서, `storageclass.yaml` 에 정의한 파일시스템과 pool 이 Ceph 클러스터에 존재하는지 확인합니다.

`storageclass.yaml` 에 정의한 `fsName` 이 `myfs` 라고 가정해 봅시다. toolbox에서 아래와 같이 확인할 수 있습니다.
```bash
ceph fs ls
```
```bash
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data0 ]
```

다음으로 `storageclass.yaml` 에 정의된 `pool` 이 있는지 확인합니다. 여기서는 `myfs-data0` 입니다.
```bash
ceph osd lspools
```
```bash
1 device_health_metrics
2 replicapool
3 myfs-metadata0
4 myfs-data0
```

CephFilesystem 커스텀 리소스로 생성된 pool 은 `-data0` suffix 를 가집니다.

### subvolumegroups
ceph-csi configmap 에 subvolumegroup 이 명시되어 있지 않다면, (ceph mon 정보를 기입한 곳) Ceph-CSI 는 csi 이름으로 기본 subvolumegroup 을 생성합니다. subvolumegroup 이 존재하는지 확인합니다.
```bash
ceph fs subvolumegroup ls myfs
```
```bash
[
   {
       "name": "csi"
   }
]
```

Ceph 클러스터에 이상이 없다면, CSI 측 이슈를 디버깅 하기 위해 다음 섹션을 참고합니다.

## Provisioning Volumes
이슈는 Ceph-CSI 나 Ceph-CSI 에서 사용되는 사이드카 컨테이너에서 발생합니다.

Ceph-CSI 는 provisioner pod 에 여러 사이드카 컨테이너를 포함합니다: `csi-attacher`, `csi-resizer`, `csi-provisioner`, `csi-cephfsplugin`, `csi-snapshotter`, `liveness-promethues`.

CephFS 의 핵심 컨테이너는 `csi-cephfsplugin` 이고, RBD 프로비저너는 `csi-rbdplugin` 입니다.

사이드카 컨테이너들에 대한 설명입니다.

### csi-provisioner
external-provisioner 는 CSI 드라이버의 `ControllerCreateVolume()` 과 `ControllerDeleteVolume()` 함수를 호출함으로서 동적으로 볼륨을 프로비저닝하는 사이드카 컨테이너입니다. 

만약 PVC 를 생성, 삭제하는 데 이슈가 있다면 `csi-provisioner` 사이드카 컨테이너의 로그를 확인하세요.
```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner -c csi-provisioner
```

### csi-resizer
CSI `external-resizer` 는 사용자가 PersistentVolumeClaim 에 더 큰 용량을 요청할 때의 업데이트 트리거를 받아 `ControllerExpandVolume` 을 호출합니다. 

PVC 용량 확장에 이슈가 있다면 `csi-resizer` 사이드카의 로그를 확인합니다.
```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner -c csi-resizer
```

### csi-snapshotter
CSI external-snapshotter 사이드카는 `VolumeSnapshotContent` 리소스의 생성/삭제/업데이트 이벤트를 watch 하고, ceph-csi 컨테이너에게 스냅샷을 생성하거나 삭제하도록 명령합니다. external-snapshotter 에 대한 자세한 설명은 [이 곳](https://github.com/kubernetes-csi/external-snapshotter) 을 참고합니다.

**쿠버네티스 1.17 부터 볼륨 스냅샷 피처는 beta 로 승격되었습니다. 1.20 부터, 기본으로 활성화 되어 있으며 비활성화 할 수 없습니다.**

올바른 snapshotter CRD 버전을 설치했는지 확인하세요. 아직 snapshotter controller 를 설치하지 않았다면, [Snapshots 가이드](/ceph_storage/snapshots.html) 를 참고합니다.

```bash
kubectl get crd | grep snapshot
```
```bash
volumesnapshotclasses.snapshot.storage.k8s.io    2021-01-25T11:19:38Z
volumesnapshotcontents.snapshot.storage.k8s.io   2021-01-25T11:19:39Z
volumesnapshots.snapshot.storage.k8s.io          2021-01-25T11:19:40Z
```

위의 CRD 들은 `snapshotclass.yaml` 이나 `snapshot.yaml` 과 동일한 버전이어야 합니다. 그렇지 않다면, `VolumeSnapshot` 과 `VolumesnapshotContent` 리소스는 만들어지지 않습니다.

snapshot controller 는 `VolumeSnapshot` 과 `VolumeSnapshotContent` 리소스들을 만드는 데 책임이 있습니다. 만약 이 리소스들이 만들어지지 않는다면, snapshot-controller 컨테이너의 로그를 확인해야 합니다.

Rook 은 snapshotter 사이드카 컨테이너만 배포하며, controller 는 배포하지 않습니다. controller 는 CSI 드라이버 종류와 상관 없이 동작하기 때문에, 쿠버네티스 클러스터에 배포해 두는 것을 권장합니다.

만약 쿠버네티스 클러스터에 snapshot controller 가 설치되지 않았다면, 수동으로 설치가 가능합니다.

snapshot 을 생성/삭제 하는데 이슈가 있다면 csi-snapshotter 사이드카 컨테이너의 로그를 확인합니다.
```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner -c csi-snapshotter
```

아래와 같은 로그를 확인할 수 있을 것입니다.
```bash
GRPC error: rpc error: code = Aborted desc = an operation with the given Volume ID
0001-0009-rook-ceph-0000000000000001-8d0ba728-0e17-11eb-a680-ce6eecc894de already >exists.
```

보통 Ceph 클러스터의 자체 이슈나 통신 이슈일 수 있습니다. PVC 프로비저닝 이슈라면, 프로비저너를 재시작 하는 것이 도움이 될 수 있습니다. PVC 를 마운팅하는 데 이슈가 있다면, `csi-rbdplugin-xxxx` pod 을 재시작 (RBD) 하거나 CephFS 이슈에서는 `csi-cephfsplugin-xxxx` pod 을 재시작합니다.

## Mounting the volume to application pods

유저가 PVC 를 사용해 어플리케이션 Pod 을 생성하려고 한다면, 세 단계를 거치게 됩니다.
- CSI 드라이버 등록
- volume attchment 리소스 생성
- 볼륨 stage, publish

### csi-driver registration
`csi-cephfsplugin-xxxx` 이나 `csi-rbdplugin-xxxx` pod 은 어플리케이션이 스케줄링 되는 모든 노드에서 동작하는 daemonset pod 입니다. 만약 어플리케이션이 스케줄링 되는 노드에 플러그인 pod 이 없다면 이슈가 생길 수 있습니다. 항상 플러그인 pod 이 동작 중인지 확인해야 합니다.

각 플러그인 pod 은 두 가지 중요한 컨테이너를 포함하고 있습니다. `driver-registrar` 와 `csi-rbdplugin` / `csi-cephfsplugin` 컨테이너 입니다. `liveness-prometheus` 컨테이너도 포함될 수 있습니다.

### driver-registrar
node-driver-registrar 는 Kubelet 에 CSI 드라이버를 등록하는 사이드카 컨테이너입니다. 더 많은 정보는 [이 곳](https://github.com/kubernetes-csi/node-driver-registrar)을 참고합니다. 

PVC 을 어플리케이션 Pod 에 attach 하는 데 이슈가 있다면 해당 어플리케이션 Pod 이 동작중인 노드의 driver-registrar 사이드카 컨테이너의 로그를 확인합니다.
```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin -c driver-registrar
```
```bash
I0120 12:28:34.231761  124018 main.go:112] Version: v2.0.1
I0120 12:28:34.233910  124018 connection.go:151] Connecting to unix:///csi/csi.sock
I0120 12:28:35.242469  124018 node_register.go:55] Starting Registration Server at: /registration/rook-ceph.rbd.csi.ceph.com-reg.sock
I0120 12:28:35.243364  124018 node_register.go:64] Registration Server started at: /registration/rook-ceph.rbd.csi.ceph.com-reg.sock
I0120 12:28:35.243673  124018 node_register.go:86] Skipping healthz server because port set to: 0
I0120 12:28:36.318482  124018 main.go:79] Received GetInfo call: &InfoRequest{}
I0120 12:28:37.455211  124018 main.go:89] Received NotifyRegistrationStatus call: &RegistrationStatus{PluginRegistered:true,Error:,}
E0121 05:19:28.658390  124018 connection.go:129] Lost connection to unix:///csi/csi.sock.
E0125 07:11:42.926133  124018 connection.go:129] Lost connection to unix:///csi/csi.sock.
```

로그에서 `RegistrationStatus{PluginRegistered:true,Error:,}` 를 확인했다면, kubelet 에 플러그인이 성공적으로 등록된 것입니다.

driver not found 에러가 확인된다면, `csi-xxxxplugin-xxx` pod 을 재시작 하는 것이 도움될 수 있습니다.

## Volume Attachment
각 프로비저너 pod 은 `csi-attacher` 라는 사이드카 컨테이너를 포함합니다.

### csi-attacher
external-attacher 는 CSI 드라이버에 `ControllerPublish` 와 `ControllerUnpublish` 함수를 호출함으로서 노드에 볼륨을 attach 합니다. 쿠버네티스 controller-manager 의 Attach/Detach 컨트롤러는 CSI 드라이버와 직접적인 인터페이스가 없기 때문에 이 external-attacher 는 필수적인 컴포넌트입니다. [이 곳](https://github.com/kubernetes-csi/external-attacher) 에서 더 많은 정보를 확인하세요.

PVC 를 어플리케이션 pod 에 attach 하는 데 이슈가 있다면, 생성된 volumeattachment 리소스를 확인하고, provisioner pod 에서 csi-attacher 사이드카 컨테이너의 로그를 확인합니다.
```bash
kubectl get volumeattachment
```
```bash
NAME                                                                   ATTACHER                        PV                                         NODE       ATTACHED   AGE
csi-75903d8a902744853900d188f12137ea1cafb6c6f922ebc1c116fd58e950fc92   rook-ceph.cephfs.csi.ceph.com   pvc-5c547d2a-fdb8-4cb2-b7fe-e0f30b88d454   minikube   true       4m26s
```
```bash
kubectl logs po/csi-rbdplugin-provisioner-d857bfb5f-ddctl -c csi-attacher
```

## CephFS Stale operations
어플리케이션 pod 이 스케줄링 된 노드의 `csi-cephfsplugin-xxxx` pod 에 stale (오래된) 마운트 커맨드가 있는지 확인합니다. 

stale 마운트를 확인하기 위해서는 `csi-cephfsplugin-xxxx` pod 에 exec 로 접근해야 합니다.

`kubectl get po -o wide` 커맨드로 어플리케이션이 동작 하고 있는 노드를 확인하고, 그 노드에 동작 중인 `csi-cephfsplugin-xxxx` pod 에 명령을 수행합니다.
```bash
kubectl exec -it csi-cephfsplugin-tfk2g -c csi-cephfsplugin -- sh
ps -ef |grep mount

root          67      60  0 11:55 pts/0    00:00:00 grep mount
```
```bash
ps -ef |grep ceph

root           1       0  0 Jan20 ?        00:00:26 /usr/local/bin/cephcsi --nodeid=minikube --type=cephfs --endpoint=unix:///csi/csi.sock --v=0 --nodeserver=true --drivername=rook-ceph.cephfs.csi.ceph.com --pidlimit=-1 --metricsport=9091 --forcecephkernelclient=true --metricspath=/metrics --enablegrpcmetrics=true
root          69      60  0 11:55 pts/0    00:00:00 grep ceph
```

커맨드가 stuck 되어 있는 것을 확인했다면, 노드에서 `dmesg` 로그를 확인합니다. `csi-cephfsplugin` pod 을 재시작하면 해결될 수 있습니다.

stuck 메시지를 찾지 못했다면, 통신 상태, Ceph 클러스터 상태, slow ops 를 확인하세요.

## RBD Stale operations
어플리케이션 Pod 이 스케줄링 된 노드의 `csi-rbdplugin-xxxx` pod 의 `map/mkfs/mount` 커맨드 stale 상태를 확인합니다.

`csi-rbdplugin-xxxx` pod 에 exec 로 접근하여 stale 오퍼레이션 (`rbd map`, `rbd unmap`, `mkfs`, `mount`, `umount`) 을 확인해야 합니다.

마찬가지로 어플리케이션이 동작하고 있는 노드의 `csi-rbdplugin-xxxx` pod 에 접근합니다.
```bash
kubectl exec -it csi-rbdplugin-vh8d5 -c csi-rbdplugin -- sh
```
```bash
ps -ef |grep map

root     1297024 1296907  0 12:00 pts/0    00:00:00 grep map
```
```bash
ps -ef |grep mount

root        1824       1  0 Jan19 ?        00:00:00 /usr/sbin/rpc.mountd
ceph     1041020 1040955  1 07:11 ?        00:03:43 ceph-mgr --fsid=ba41ac93-3b55-4f32-9e06-d3d8c6ff7334 --keyring=/etc/ceph/keyring-store/keyring --log-to-stderr=true --err-to-stderr=true --mon-cluster-log-to-stderr=true --log-stderr-prefix=debug  --default-log-to-file=false --default-mon-cluster-log-to-file=false --mon-host=[v2:10.111.136.166:3300,v1:10.111.136.166:6789] --mon-initial-members=a --id=a --setuser=ceph --setgroup=ceph --client-mount-uid=0 --client-mount-gid=0 --foreground --public-addr=172.17.0.6
root     1297115 1296907  0 12:00 pts/0    00:00:00 grep mount
```
```bash
ps -ef |grep mkfs

root     1297291 1296907  0 12:00 pts/0    00:00:00 grep mkfs
```
```bash
ps -ef |grep umount

root     1298500 1296907  0 12:01 pts/0    00:00:00 grep umount
```
```bash
ps -ef |grep unmap

root     1298578 1296907  0 12:01 pts/0    00:00:00 grep unmap
```

커맨드가 stuck 되어 있는 것을 확인했다면, 노드의 `dmesg` 로그를 확인합니다. `csi-rbdplugin` pod 을 재시작하면 해결 될 수 있습니다.

stuck 메시지를 확인하지 못했다면, 통신 상태, Ceph 클러스터 상태, slow ops 를 확인합니다.

## dmesg logs
`csi-rbdplugin-xxxx` pod 의 `csi-rbdplugin` 컨테이너가 동작 중인 PVC 마운트가 실패하는 노드에 대해 dmesg 로그를 확인합니다.
```bash
dmesg
```

## RBD Commands
위 방법으로 해결이 되지 않는다면, ceph-csi pod 로그로부터 마지막 실행된 커맨드를 provisioner 나 플러그인 pod 에서 수동으로 실행시키고, 에러가 있는지 확인합니다. 
```bash
$ rbd ls --id=csi-rbd-node -m=10.111.136.166:6789 --key=AQDpIQhg+v83EhAAgLboWIbl+FL/nThJzoI3Fg==
```

`-m` 옵션은 mon 엔드포인트이고, `--key` 옵션은 CSI 드라이버가 Ceph 클러스터에 접근하기 위한 키 입니다.

## Node Loss
노드가 죽으면, 해당 노드의 어플리케이션 pod 이 `Terminating` 상태로 남고 리스케줄링된 pod 은 `ContainerCreating` 상태가 됩니다.

이 pod 이 다른 노드에 정상적으로 동작하게 하기 위해서, `Terminating` 상태의 pod 을 강제 삭제합니다.

### Force deleting the pod
`Terminating` 상태로 stuck 된 pod 을 강제 삭제합니다:
```bash
$ kubectl -n rook-ceph delete pod my-app-69cd495f9b-nl6hf --grace-period 0 --force
```

강제 삭제 이후, 8-10 분정도 대기합니다. pod 이 running 상태가 되지 않으면, 노드를 blacklist 에 추가하기 위해 다음 섹션을 참고합니다.

### Blocklisting a node
타임아웃을 짧게 하고 failover 를 빠르게 하기 위해, 노드를 "blocklisted" 로 등록할 수 있습니다. 

Ceph 버전이 Pacific(v16.2.x) 이상이라면, 아래 커맨드를 실행합니다.
```bash
$ ceph osd blocklist add <NODE_IP> # blocklist 등록할 노드의 IP 를 조회해야 함
blocklisting <NODE_IP>
```

Ceph 버전이 Octopus(v15.2.x) 이하라면, 아래 커맨드를 실행합니다.
```bash
$ ceph osd blacklist add <NODE_IP> # blocklist 등록할 노드의 IP 를 조회해야 함
blacklisting <NODE_IP>
```

위 커맨드를 실행한 후 pod 이 running 이 될 때까지 몇 분 대기합니다.

### Removing a node blocklist
노드가 완전히 오프라인 상태이고 더이상 blocklist 될 필요가 없다면, 노드를 blocklist 로부터 삭제합니다.

Ceph 버전이 Pacific(v16.2.x) 이상이라면, 아래 커맨드를 실행합니다.
```bash
$ ceph osd blocklist rm <NODE_IP>
un-blocklisting <NODE_IP>
```

Ceph 버전이 Octopus(v15.2.x) 이하라면, 아래 커맨드를 실행합니다.
```bash
$ ceph osd blacklist rm <NODE_IP> # get the node IP you want to blacklist
un-blacklisting <NODE_IP>
```

