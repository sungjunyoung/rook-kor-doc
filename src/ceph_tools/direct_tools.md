# Direct Tools

Rook 은 처음부터 쿠버네티스 디자인 원칙을 따라 디자인되었습니다. 이 문서에서는 쿠버네티스 스토리지의 범위를 벗어나 쿠버네티스의 마법 없이 Pod 에서 직접 블록 및 파일 스토리지를 사용하는 방법을 보여줍니다. 이 주제의 목적은 새로운 구성을 신속하게 테스트 하는 것이며, 이 설정은 프로덕션 환경에서 사용하는 것은 아닙니다. failover, detach, attach 등 쿠버네티스 스토리지의 이점은 사용이 불가능하며, pod 이 죽으면 mount 도 함께 없어집니다.

## Start the Direct Mount Pod
Ceph 볼륨 마운트를 테스트하려면, 필수 마운트 옵션과 함께 pod 을 기동합니다. 예제 디렉토리에서 yaml 파일을 확인할 수 있습니다. 
```bash
kubectl create -f deploy/examples/direct-mount.yaml
```

Pod 이 생성되면, 아래와 같이 pod 에 연결합니다.
```bash
kubectl -n rook-ceph get pod -l app=rook-direct-mount
$ kubectl -n rook-ceph exec -it <pod> bash
```

## Block Storage Tools
[Block Storage](/ceph_storage/block_storage.html) 토픽에 따라 pool 을 생성하고 난 뒤, 블록 이미지를 생성하고 pod 에 직접 마운트 시킬 수 있습니다. 아래 예제는 어떻게 pod 에 rbd 볼륨을 마운트 할 수 있는지 보여줍니다.

[Direct Mount Pod](#Start-the-Direct-Mount-Pod) 을 생성합니다.

볼륨 이미지(10MB) 를 생성합니다.
```bash
rbd create replicapool/test --size 10
rbd info replicapool/test

# 커널 모듈에 없는 rbd 피처들을 비활성화합니다.
rbd feature disable replicapool/test fast-diff deep-flatten object-map
```

블록 볼륨을 map 하고, 포맷합니다.
```bash
# rbd 디바이스를 Map 합니다. pod 이 "hostNetwork: false" 로 실행될 경우, 이 명령은 hang 되며, Ctrl+C 로 중지해야 합니다. 다만 커맨드는 성공합니다. (https://github.com/rook/rook/issues/2021)
rbd map replicapool/test

# 디바이스 이름을 찾습니다. (ex. /dev/rbd0)
lsblk | grep rbd

# 볼륨을 포맷합니다. (처음 1회만 실행하며, 중복 실행할경우 데이터는 삭제됩니다.)
mkfs.ext4 -m0 /dev/rbd0

# 블록 디바이스를 마운트합니다.
mkdir /tmp/rook-volume
mount /dev/rbd0 /tmp/rook-volume
```

파일을 읽고 씁니다:
```bash
echo "Hello Rook" > /tmp/rook-volume/hello
cat /tmp/rook-volume/hello
```

### Unmount the Block device
볼륨을 Unmount 하고 커널 디바이스를 unmap 합니다.
```bash
umount /tmp/rook-volume
rbd unmap /dev/rbd0
```

## Shared Filesystem Tools
[Shared Filesystem](/ceph_storage/shared_filesystem.html) 토픽에서 파일시스템을 생성한 후, 여러 pod 에서 파일시스템을 마운트해 사용할 수 있습니다. Direct Mount pod 에 같은 파일시스템을 마운트 해 볼 것입니다. 이 예제는 Ceph 파일시스템을 테스트 해보기 위한 목적이며, 프로덕션 쿠버네티스 pod 에서는 권장되지 않습니다.

[Direct Mount Pod](#Start-the-Direct-Mount-Pod) 에 따라 pod 을 만들고, pod 에 연결한 후 아래 커맨드들을 실행합니다.
```bash
# 디렉토리 생성
mkdir /tmp/registry

# 연결을 위한 mon 엔드포인트와 유저 시크릿 식별
mon_endpoints=$(grep mon_host /etc/ceph/ceph.conf | awk '{print $3}')
my_secret=$(grep key /etc/ceph/keyring | awk '{print $3}')

# 파일시스템 마운트
mount -t ceph -o mds_namespace=myfs,name=admin,secret=$my_secret $mon_endpoints:/ /tmp/registry

# 마운트된 파일시스템 확인
df -h
```

이제 마운트된 파일시스템을 확인할 수 있습니다. 레지스트리에 이미지를 푸시했다면 `docker` 라는 디렉토리를 확인할 수 있을것입니다.
```bash
ls /tmp/registry
```

공유 파일시스템에 파일을 읽고 써봅니다.
```bash
echo "Hello Rook" > /tmp/registry/hello
cat /tmp/registry/hello

# 확인했다면 파일을 삭제합니다.
rm -f /tmp/registry/hello
```

### Unmount the Filesystem
Direct Mount Pod 에서 공유 파일시스템을 Unmount 합니다:
```bash
umount /tmp/registry
rmdir /tmp/registry
```

파일시스템을 unmount 해도 데이터는 삭제되지 않습니다.
