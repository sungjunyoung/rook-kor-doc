# Examples

블록 디바이스, 공유 파일시스템, 그리고 오브젝트 스토리지를 쿠버네티스 네임스페이스에 제공하기 위해 Rook 과 Ceph 을 설정하는 방법은 다양합니다. 스토리지 셋업을 간단하게 하기 위한 여러가지 예제를 제공했지만, 여러분의 환경에 따라 다양한 튜닝과 설정이 필요할 수 있습니다.

rook/ceph 셋업 스펙 확인을 위해 [example yaml files](https://github.com/rook/rook/blob/release-1.8/deploy/examples) 를 확인하세요.

## Common Resources

Rook 을 배포하기 위한 첫번째 스텝은 공통 리소스와 CRD 들을 생성하는 것입니다. 이 리소스들을 위한 설정은 대부분의 배포에서 동일합니다. [crds.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/crds.yaml) 과 [common.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/common.yaml) 을 사용합니다.

```sh
kubectl create -f crds.yaml -f common.yaml
```

위 예제는 오퍼레이터와 Ceph 데몬들이 모두 같은 네임스페이스에 배포되는 것을 가정합니다. 만약 여러분이 다른 네임스페이스 배포를 원한다면, `common.yaml` 파일의 주석을 확인하세요.

## Operator

공통 리소스가 만들어졌다면, 다음 스텝은 오퍼레이터 Deployment 를 만드는 읿입니다. [이 디렉토리](https://github.com/rook/rook/blob/release-1.8/deploy/examples/)에 여러 파일 스펙 예제들이 포함되어 있습니다.
- `operator.yaml`: 가장 공통된 세팅의 프로덕션 배포
  - `kubectl create -f operator.yaml`
- `operator-openshft.yaml`: Openshift 환경 배포에 최적화
  - `oc create -f operator-openshift.yaml`

오퍼레이터를 위한 설정들은 오퍼레이터 Deployment 에 환경변수로 설정됩니다. 각 세팅은 [operator.yaml](https://github.com/rook/rook/blob/release-1.8/deploy/examples/operator.yaml) 에 기술되어 있습니다.

## Cluster CRD

오퍼레이터가 동작하기 시작하면, Ceph 스토리지 클러스터를 만들 수 있습니다. 이 커스텀 리소스는 오퍼레이터가 스토리지를 어떻게 운영할 것인지에 대한 크리티컬한 설정들이 포함됩니다. 클러스터를 설정하기 위한 여러가지 방법들을 충분히 이해하는 것이 중요합니다. 아래 예제들은 스토리지를 구성하기 위한 간단한 방법들을 보여줍니다.
- `cluster.yaml`: 프로덕선 스토리지 클러스터의 공통 설정을 포함합니다. 적어도 세개 이상의 워커 노드가 필요합니다.
- `cluster-test.yaml`: 오직 하나의 노드만 필요한 테스트용 클러스터 설정입니다.
- `cluster-on-pvc.yaml`: Ceph mon 과 osd 들을 PV 로 사용하는 설정을 포함하고 있습니다. 클라우드 환겨이나 local PV 가 만들어져 있는 환경에 적합합니다.
- `cluster-external.yaml`: 외부 Ceph 클러스터를 연결하기 위한 설정입니다.
- `cluster-external-management.yaml`: 어드민 키를 사용해 원격으로 pool 을 만들고 오브젝트 스토어, 공유 파일시스템 등을 설정할 수 있는 설정입니다.
- `cluster-stretched.yaml`: 클러스터를 "stretched" 모들 생성할 수 있는 설정입니다. 세 개의 존에 5개의 노드, 그리고 두개의 존에 OSD 가 펼쳐집니다. [Stretch documentation](/ceph_storage/cluster_crd.html#stretch-cluster) 문서를 참고하세요.

더 많은 정보는 [Cluster CRD](/ceph_storage/cluster_crd.html) 를 참고하세요.

## Setting up consumable storage

이제 Rook Ceph 클러스터 내에 블록, 오브젝트 스토리지 및 공유 파일 시스템이 사용 가능한 상태가 되었습니다. 이 스토리지들은 각각 CephBlockPool, CephFilesystem, CephObjectStore 리소스로 참조됩니다.

### Block Devices
Ceph 은 pod 에 raw 블록 디바이스를 제공할 수 있습니다. 아래의 각 예제들은 쿠버네티스 pod 에 블록 디바이스를 제공하는 storage class 를 셋업합니다. 이 storage class 는 [pool](http://docs.ceph.com/docs/master/rados/operations/pools/)과 함께 저장 방식에 따라 정의됩니다.
- `storageclass.yaml`: 이 예제는 3개의 레플리케이션을 가진 프로덕션 시나리오에서 사용되며, 적어도 세개의 워커 노드를 필요로 합니다. 쿠버네티스의 세 워커 노드에 레플리케이션되어 하나의 노드가 죽어도 데이터에 손실이 생기지 않습니다.
- `storageclass-ec.yaml`: Erasure coding 방식으로 설정합니다. Erasure coding 방식은 레플리케이션 방식보다 더 적은 데이터 공간을 차지하지만, 더 많은 컴퓨팅 파워를 요구합니다. 마찬가지로 적어도 세개의 워커 노드가 필요합니다. 문서를 참고하세요.
- `storageclass-test.yaml`: 1개의 레플리케이션을 요구하는 테스트용 시나리오를 위해 사용됩니다. 한개의 노드가 죽으면 데이터 손실로 이어지므로 프로덕션 레벨에서 사용해서는 안됩니다.

storage class 는 드라이버에 따라 다른 서브디렉토리에 존재합니다.
- `csi/rbd`: 블록 디바이스를 위한 CSI Driver 입니다. 

더 많은 정보는 [Ceph Pool CRD](/ceph_storage/block_pool_crd.html) 문서를 참고하세요.

### Shared Filesystem

Ceph 파일시스템 (Cephfs) 는 하나 이상의 호스트(Pod)에 POSIX 호환의 폴더 마운트를 가능하게 해줍니다. NFS 나 CIFS 와 비슷합니다.

파일 스토리지는 여러 pool 을 포함하며, 여러 시나리오에서 아래 파일들을 사용합니다.
- `filesystem.yaml`: 프로덕션 시나리오를 위한 3개의 레플리케이션, 세개 이상의 노드가 필요함
- `filesystem-ec.yaml`: 프로덕션 시나리오를 위한 Erasure coding, 세개 이상의 노드가 필요함
- `filesystem-test.yaml`: 테스트 시나리오를 위한 1개의 레플리케이션, 한개 노드가 필요

CSI Driver 를 통해 다이나믹 프로비저닝이 가능합니다. 공유 파일시스템을 위한 storage class 는 `csi/cephfs` 디렉토리에 있습니다.

[Shared Filesystem CRD](/ceph_storage/shared_filesstem_crd.html) 문서를 참고하세요.

### Object Storage

Ceph 은 HTTP(s)-type 로 get/put/post/delete 오퍼레이션을 수행할 수 있는 오브젝트 스토리지를 제공합니다. AWS S3 스토리지와 비슷합니다.

오브젝트 스토리지는 여러 pool 을 포함하며, 여러 시나리오에서 각각 아래 예제 파일들을 사용할 수 있습니다.
- `object.yaml`: 프로덕션을 위한 3개의 레플리케이션, 세개 이상의 노드 필요
- `object-openshift.yaml`: Openshift 환경을 위한 3개의 레플리케이션, 세개 이상의 노드 필요
- `object-ec.yaml`: 프로덕션을 위한 Erasure coding, 세개 이상의 노드 필요
- `object-test.yaml`: 테스트 환경을 위한 1개 레플리케이션, 한개 노드 필요

[Object Store CRD](/ceph_storage/object_store_crd.html) 문서를 참고하세요.

### Object Storage User
- `object-user.yaml`: 오브젝트 스토리지 유저를 생성하고, S3 API 를 위한 인증 정보를 생성합니다.

### Object Storage Bucket

또한 Ceph 오퍼레이터는 기 존재하는 버킷에 권한을 부여하거나 신규 버킷을 다이나믹하게 생성할 수 있는 object store bucket provisioner 를 실행합니다.
- `object-bucket-claim-retain.yaml`: Retain 상태로 StorageClass 를 참조하는 신규 버킷 요청을 생성합니다.
- `object-bucket-claim-delete.yaml`: Delete 상태로 StorageClass 를 참조하는 신규 버킷 요청을 생성합니다.
- `storageclass-bucket-retain.yaml`: Retain 상태로 Ceph 오브젝트 스토어, 리전을 정의하는 StorageClass 를 생성합니다.
- `storageclass-bucket-delete.yaml`: Delete 상태로 Ceph 오브젝트 스토어, 리전을 정의하는 StorageClass 를 생성합니다.
