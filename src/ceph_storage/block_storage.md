# Block Storage

블록 스토리지는 단일 Pod 에 스토리지 마운트를 지원합니다. 이 가이드는 Rook 으로 활성화된 PersistentVolume 을 어떻게 만들고 구성하는지 보여줍니다. 

## Prerequisites

[Quickstart](/quickstart.html) 가이드에 따라 Rook 클러스터가 구축된 상황을 가정합니다.

## Provision Storage

Rook 이 스토리지를 프로비저닝 하기 전에, `StorageClass` 와 `CephBlockPool` 커스텀 리소스가 생성되어 있어야 합니다. 이렇게 하면 PersistentVolume 을 프로비저닝 할 때 쿠버네티스가 Rook 과 상호작용 할 수 있습니다.

> **NOTE**: 예제는 세개의 다른 노드에 있는 OSD 와 함께 노드당 적어도 1개 이상의 OSD 가 필요합니다. 

아래의 `stoargeclass.yaml` 에 정의되어 있는 `StorageClass` 를 저장합니다. 

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    clusterID: rook-ceph
    # Ceph pool into which the RBD image shall be created
    pool: replicapool

    # (optional) mapOptions is a comma-separated list of map options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # mapOptions: lock_on_read,queue_depth=1024

    # (optional) unmapOptions is a comma-separated list of unmap options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # unmapOptions: force

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
    imageFeatures: layering

    # The secrets contain Ceph admin credentials.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

    # Specify the filesystem type of the volume. If not specified, csi-provisioner
    # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
    # in hyperconverged settings where the volume is mounted on the same node as the osds.
    csi.storage.k8s.io/fstype: ext4

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete

# Optional, if you want to add dynamic resize for PVC. Works for Kubernetes 1.14+
# For now only ext3, ext4, xfs resize support provided, like in Kubernetes itself.
allowVolumeExpansion: true
```

"rook-ceph" 외에 다른 네임스페이스에 Rook 오퍼레이터를 배포했다면, 프로비저너의 프리픽스를 해당 네임스페이스로 변경해야 합니다. 예를 들어, Rook 오퍼레이터가 설치된 네임스페이스가 "my-namespace" 라면 프로비저너 스펙은 "my-namespace.rbd.csi.ceph.com" 이 됩니다.

storage class 를 생성합니다.

```sh
kubectl create -f deploy/examples/csi/rbd/storageclass.yaml
```

> **NOTE**: [쿠버네티스 문서](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#retain) 에서 언급되었다시피, `Retain` reclaim policy 를 사용하게 되면, `PersistentVolume` 이 삭제되더라도 Ceph RBD 볼륨은 삭제되지 않습니다. `rbd rm` 을 통해 클린업이 필요합니다.

## Consume the storage: Wordpress sample

Rook 에 의해 프로비저닝 된 블록 스토리지를 사용하여 wordpress 와 mysql 어플리케이션을 생성합니다. 
두 어플리케이션은 Rook 으로 프로비저닝 된 블록 볼륨을 사용합니다.

`deploy/examples` 폴더 내의 mysql 과 wordpress 를 배포합니다.

```sh
kubectl create -f mysql.yaml
kubectl create -f wordpress.yaml
```

두 어플리케이션은 각각 블록 볼륨을 마운트하며, 아래 커맨드로 쿠버네티스 PersistentVolumeClaim 이 생성된 것을 확인할 수 있습니다.

```sh
kubectl get pvc
```
```sh
NAME             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
mysql-pv-claim   Bound     pvc-95402dbc-efc0-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
wp-pv-claim      Bound     pvc-39e43169-efc1-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
```

두 Pod 이 `Running` 상태가 되면, 여러분의 브라우저에서 접근할 포트를 확인합니다.

```sh
kubectl get svc wordpress
```
```sh
NAME        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
wordpress   10.3.0.155   <pending>     80:30841/TCP   2m
```

이제 wordpress 어플리케이션이 실행하는 것을 확인할 수 있습니다.
minikube 를 사용한다면, 아래 커맨드로 wordpress URL 을 확인할 수 있습니다.

```sh
echo http://$(minikube ip):$(kubectl get service wordpress -o jsonpath='{.spec.ports[0].nodePort}')
```

> **NOTE**: vagrant 환경에서 실행한다면, wordpress 어플리케이션을 확인할 external ip 가 존재하지 않습니다. 쿠버네티스 클러스터 안에서 `CLUSTER-IP` 를 통해서만 wordpress 에 접근할 수 있습니다.

## Consume the storage: Toolbox

위에서 만든 pool 로, 블록 이미지를 만들고 pod 에 직접적으로 마운트 할 수 있습니다. [Direct Block Tools](/ceph_tools/direct_tools.html) 를 참고하세요.

## Teardown

위 데모에서 생성한 모든 리소스를 삭제하려면 아래 커맨드를 입력합니다.

```sh
kubectl delete -f wordpress.yaml
kubectl delete -f mysql.yaml
kubectl delete -n rook-ceph cephblockpools.ceph.rook.io replicapool
kubectl delete storageclass rook-ceph-block
```

## Advanced Example: Erasure Coded Block Storage

RBD 를 erasure code pool 로 사용하고 싶다면, OSD 의 `storageType` 이 `bluestore` 로 설정되어 있어야 합니다. 또한 마운트가 가능하려면 커널이 `4.11` 이상이어야 합니다.

**NOTE**: 예제는 세 개 이상의 OSD 가 필요하며, 각 OSD 는 서로 다른 노드에 존재해야 합니다.

OSD 는 `failureDomain` 이 `host` 로 설정되어 있고, `erasureCoded` 청크 설정이 적어도 세 개의 다른 OSD (2 `dataChunks` + 1 `codingChunks`) 가 필요하기 문에, OSD 는 서로 다른 노드에 존재하고 있어야 합니다.

erasure coded pool 의 사용을 위해서는 두개의 pool 을 생성해야 합니다.

### Erasure Coded CSI Driver

Erasure coded pool 을 사용하려면 `storageclass-ec.yaml` 에 `dataPool` 파라미터가 설정되어 있어야 합니다. RBD 이미지의 데이터로 사용됩니다.

### Erasure Coded Flex Driver

Erasure coded pool 을 사용하려면 `storageclass-ec.yaml` 에 `dataBlockPool` 파라미터가 설정되어 있어야 합니다. RBD 이미지의 데이터로 사용됩니다.