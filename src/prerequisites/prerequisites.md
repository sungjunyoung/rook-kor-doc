# Prerequisite

Rook 는 최소 버전 조건을 충족하고, 필요한 권한만 부여되어 있다면 어떤 쿠버네티스 클러스터에서도 설치가 가능합니다. (자세한 정보는 아래를 참고하세요.)

## Minimum Version

Ceph operator 를 위해 Kubernetes v1.16 버전 이상이 필요합니다.

## Ceph Prerequistes

Ceph 스토리지 클러스터를 설정하기 위해, 아래 로컬 스토리지 옵션 중 하나를 충족해야 합니다:
- Raw 디바이스 (파티션이 없고, 포맷되어있지 않아야함)
- Raw 파티션 (포맷되어 있지 않아야 함)
- StorageClass 로 PersistentVolume 이 `block` 모드로 생성 가능해야 함

아래 커맨드로 파티션이나 디바이스가 파일시스템으로 포맷되었는지 확인할 수 있습니다.

```sh
lsblk -f
```
```sh
NAME                  FSTYPE      LABEL UUID                                   MOUNTPOINT
vda
└─vda1                LVM2_member       >eSO50t-GkUV-YKTH-WsGq-hNJY-eKNf-3i07IB
 ├─ubuntu--vg-root   ext4              c2366f76-6e21-4f10-a8f3-6776212e2fe4   /
 └─ubuntu--vg-swap_1 swap              9492a3dc-ad75-47cd-9596-678e8cf17ff9   [SWAP]
vdb
```

`FSTYPE` 필드가 비어있지 않다면, 해당 디바이스에 파일시스템이 존재하는 것입니다. 위 예제에서는 `vdb` 를 사용할 수 있지만, `vda` 는 사용할 수 없습니다.

## LVM package

Ceph OSD 는 아래 시나리오에서 LVM 에 디펜던시를 가지고 있습니다.
- OSD 가 raw 디바이스나 파티션에 생성될 때
- 암호화 옵션이 켜져 있을 때 (cluster 커스텀 리소스의 `encryptedDevice: true`)
- `metadata` 디바이스가 지정되어 있을 때

다음과 같은 상황에서는 LVM 이 필요하지 않습니다.
- `storageClassDeviceSets` 를 사용해 PVC 위에 OSD 를 생성할 떄

여러분의 시나리오에서 LVM 이 필요하다면, OSD 가 실행될 호스트에서 LVM 이 설치되어 있어야 합니다. 
몇몇 리눅스 배포판에서는 `lvm2` 패키지가 설치되어 있지 않습니다. LVM 패키지 없이는, 노드가 재시작 될 때 OSD Pod 이 정상적으로 시작되지 않습니다. 
리눅스 배포판의 패키지 매니저로 LVM 패키지 설치가 권장됩니다.

CentOS:
```sh
sudo yum install -y lvm2
```
Ubuntu:
```sh
sudo apt-get install -y lvm2
```
RancherOS:
- [1.5.0](https://github.com/rancher/os/issues/2551) 부터 LVM 이 지원됩니다.
- 논리적인 볼륨은 부팅 프로세스에서는 [활성화되지 않습니다.](https://github.com/rook/rook/issues/5027). 가능하게 하기 위해서는 [runcmd command](https://rancher.com/docs/os/v1.x/en/installation/configuration/running-commands/) 를 추가해야 합니다.
```yaml
runcmd:
- [ vgchange, -ay ]
```

## Kernel

### RBD
Ceph 은 RBD 모듈이 포함된 리눅스 커널이 필요합니다. 대부분의 리눅스 배포판에는 이 모듈이 포함되어 있지만, 아닌 배포판도 존재합니다. (GKE Container-Optimised OS)

쿠버네티스 노드에서 `modeprobe rbd` 커맨드로 확인이 가능합니다. 만약 'not found' 라고 나온다면, 다른 리눅스 배포판으로 커널을 다시 빌드해야 합니다.

### Cephfs
Ceph shared filesystem (Cephfs) 볼륨을 만들고자 한다면, 4.17 이상의 커널 버전이 권장됩니다. 커널 버전이 4.17 미만이라면, 요청된 PVC 사이즈 쿼타가 정상적으로 제한되지 않습니다.