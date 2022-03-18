# Advanced Configuration

이 문서는 Rook 스토리지 클러스터에서 고급 설정을 수행하는 방법을 보여줍니다.
- [Prerequisites](#prerequisites)
- [Using alternate namespaces](#using-alternate-namespaces)
- [Deploying a second cluster](#deploying-a-second-cluster)
- [Log Collection](#log-collection)
- [OSD Information](#osd-information)
- [Separate Storage Groups](#separate-storage-groups)
- [Configuring Pools](#configuring-pools)
- [Custom ceph.conf Settings](#custom-cephconf-settings)
- [OSD CRUSH Settings](#osd-crush-settings)
- [OSD Dedicated Network](#osd-dedicated-network)
- [Phantom OSD Removal](#phantom-osd-removal)
- [Auto Expansion of OSDs](#auto-expansion-of-osds)

## Prerequisites
대부분의 예제는 `ceph` 클라이언트 커맨드를 사용합니다. [Rook Toolbox 컨테이너](/ceph_tools/toolbox.html) 를 사용해 Ceph 클라이언트에 접근할 수 있습니다.

쿠버네티스 기반 예제들은 Rook OSD 팟들이 모두 `rook-ceph` 네임스페이스에 존재한다고 가정하고 있습니다. 만약 다른 네임스페이스에 배포했다면, `kubectl -n rook-ceph [...]` 커맨드를 상황에 맞게 변경하세요.

## Using alternate namespaces
기본값인 `rook-ceph` 네임스페이스 대신 다른 네임스페이스에 Rook 오퍼레이터와 Ceph 클러스터를 배포하길 바란다면, `sed` 커맨드를 통해 마니페스트를 변경해야 합니다. 

```bash
cd deploy/examples

export ROOK_OPERATOR_NAMESPACE="rook-ceph"
export ROOK_CLUSTER_NAMESPACE="rook-ceph"

sed -i.bak \
    -e "s/\(.*\):.*# namespace:operator/\1: $ROOK_OPERATOR_NAMESPACE # namespace:operator/g" \
    -e "s/\(.*\):.*# namespace:cluster/\1: $ROOK_CLUSTER_NAMESPACE # namespace:cluster/g" \
    -e "s/\(.*serviceaccount\):.*:\(.*\) # serviceaccount:namespace:operator/\1:$ROOK_OPERATOR_NAMESPACE:\2 # serviceaccount:namespace:operator/g" \
    -e "s/\(.*serviceaccount\):.*:\(.*\) # serviceaccount:namespace:cluster/\1:$ROOK_CLUSTER_NAMESPACE:\2 # serviceaccount:namespace:cluster/g" \
    -e "s/\(.*\): [-_A-Za-z0-9]*\.\(.*\) # driver:namespace:operator/\1: $ROOK_OPERATOR_NAMESPACE.\2 # driver:namespace:operator/g" \
    -e "s/\(.*\): [-_A-Za-z0-9]*\.\(.*\) # driver:namespace:cluster/\1: $ROOK_CLUSTER_NAMESPACE.\2 # driver:namespace:cluster/g" \
  common.yaml operator.yaml cluster.yaml # add other files or change these as desired for your config

# You need to use `apply` for all Ceph clusters after the first if you have only one Operator
kubectl apply -f common.yaml -f operator.yaml -f cluster.yaml # add other files as desired for yourconfig
```

## Deploying a second cluster
새로운 CephCluster 를 `rook-ceph`이 아닌 다른 네임스페이스에 만들고 단일 오퍼레이터가 두 클러스터를 관리하게 하고자 한다면, 아래 커맨드를 입력합니다:
```bash
cd deploy/examples

NAMESPACE=rook-ceph-secondary envsubst < common-second-cluster.yaml | kubectl create -f -
```

이 커맨드는 새로운 네임스페이스에 필요한 RBAC 들을 생성합니다. 해당 스크립트는 `common.yaml` 이 이미 배포되었다고 가정합니다. 두번쨰 CephCluster CR 을 생성할 때, 두번째 클러스터를 설정한 네임스페이스를 사용합니다.

## Log Collection
Rook 로그는 아래 커맨드로 쿠버네티스 환경에서 수집할 수 있습니다. 
```bash
for p in $(kubectl -n rook-ceph get pods -o jsonpath='{.items[*].metadata.name}')
do
    for c in $(kubectl -n rook-ceph get pod ${p} -o jsonpath='{.spec.containers[*].name}')
    do
        echo "BEGIN logs from pod: ${p} ${c}"
        kubectl -n rook-ceph logs -c ${c} ${p}
        echo "END logs from pod: ${p} ${c}"
    done
done
```

이 커맨드는 Rook pod 의 모든 컨테이너에서 로그를 수집합니다. 

## OSD Information
OSD 와 OSD 가 사용하는 스토리지 디바이스를 추적하는 것은 어렵습니다. 아래 스크립트는 이런 조회를 손쉽게 해줍니다.
### Kubernetes
```bash
# OSD Pod 을 조회합니다.
# "rook" 이름의 기본 설정 클러스터 이름을 사용합니다.
OSD_PODS=$(kubectl get pods --all-namespaces -l \
  app=rook-ceph-osd,rook_cluster=rook-ceph -o jsonpath='{.items[*].metadata.name}')

# OSD pod 과 연관된 노드와 드라이브를 조회합니다.
for pod in $(echo ${OSD_PODS})
do
 echo "Pod:  ${pod}"
 echo "Node: $(kubectl -n rook-ceph get pod ${pod} -o jsonpath='{.spec.nodeName}')"
 kubectl -n rook-ceph exec ${pod} -- sh -c '\
  for i in /var/lib/ceph/osd/ceph-*; do
    [ -f ${i}/ready ] || continue
    echo -ne "-$(basename ${i}) "
    echo $(lsblk -n -o NAME,SIZE ${i}/block 2> /dev/null || \
    findmnt -n -v -o SOURCE,SIZE -T ${i}) $(cat ${i}/type)
  done | sort -V
  echo'
done
```

결과는 아래와 같습니다.
```bash
Pod:  osd-m2fz2
Node: node1.zbrbdl
-osd0  sda3  557.3G  bluestore
-osd1  sdf3  110.2G  bluestore
-osd2  sdd3  277.8G  bluestore
-osd3  sdb3  557.3G  bluestore
-osd4  sde3  464.2G  bluestore
-osd5  sdc3  557.3G  bluestore

Pod:  osd-nxxnq
Node: node3.zbrbdl
-osd6   sda3  110.7G  bluestore
-osd17  sdd3  1.8T    bluestore
-osd18  sdb3  231.8G  bluestore
-osd19  sdc3  231.8G  bluestore

Pod:  osd-tww1h
Node: node2.zbrbdl
-osd7   sdc3  464.2G  bluestore
-osd8   sdj3  557.3G  bluestore
-osd9   sdf3  66.7G   bluestore
-osd10  sdd3  464.2G  bluestore
-osd11  sdb3  147.4G  bluestore
-osd12  sdi3  557.3G  bluestore
-osd13  sdk3  557.3G  bluestore
-osd14  sde3  66.7G   bluestore
-osd15  sda3  110.2G  bluestore
-osd16  sdh3  135.1G  bluestore
```

## Separate Storage Groups
> **DEPRECATED**: 수동으로 설정하는 것 대신, `CephBlockPool`, `CephFilesystem`, `CephObjectStore` CRD 의  `deviceClass` 프로퍼티를 사용할 수 있습니다.

## Configuring Pools
### Placement Group Sizing
> **NOTE**: Ceph Nautilus (v14.x) 부터, PG 를 오토 스케일링하도록 Ceph MGR 의 `pg_autoscaler` 모듈을 사용할 수 있습니다. 이 피처를 활성화 하려면, [Default PG and PGP counts](/ceph_storage/configuration.html#default-pg-and-pgp-counts) 를 참고합니다.

얼마나 많은 PG 들이 Pool 에 있어야 하는지에 대한 기본적인 룰은 아래와 같습니다:
- 5개 이하의 OSD - pg_num: 128
- 5-10개의 OSD - pg_num: 512
- 10-50개의 OSD - pg_num: 1024

50개 이상의 OSD 를 가지고 있다면, 트레이드오프를 이해하고 있어야 하며, 어떻게 pg_num 을 계산할지 스스로 알고 있어야 합니다. pg_num 을 계산하는 방법은 [pgcalc tool](https://old.ceph.com/pgcalc/) 을 참고할 수 있습니다. 이미 pool 을 사용하고 있다면 [PG count 를 증가](#setting-pg-count) 시키는 게 안전합니다. PG count 를 사용 중에 줄이는 것은 권장되지 않습니다. PG count 를 줄이는 안전한 방법은 데이터를 백업하고, Pool 을 삭제한 후 다시 생성하는 것입니다. 백업을 이용하면 사용중인 pool 에 대해 안전하지 않을 수 있는 몇 가지 트릭을 [시도](http://cephnotes.ksperis.com/blog/2015/04/15/ceph-pool-migration) 할 수 있습니다.

### Setting PG Count
PG 수를 변경하기 전에 [placement group sizing](#placement-group-sizing) 섹션을 참고해야 합니다.
```bash
# rbd pool 의 pg 수를 512 로 변경하기
ceph osd pool set rbd pg_num 512
```

## Custom ceph.conf Settings
> **경고**: 아래 방법은 더 유연한 설정을 제공하기 위해 수동으로 Ceph CLI 나 Ceph 대시보드를 사용합니다. 반드시 필요할 때만 사용되는 것이 권고되며, 설정이 필요없게 되었을 떄는 문자열로 리셋하는 것을 강하게 추천합니다. config 파일 구성은 Ceph 클러스터를 CLI 및 대시보드에서 구성의 유연성을 떨어뜨리고 향후 튜닝 또는 디버깅을 어렵게 만들 수 있습니다.

Ceph CLI 를 사용하여 설정하려면 적어도 1개의 mon 을 사용하고 있어야 합니다. 대시보드를 사용하여 설정을 하려면 1개 이상의 mgr 이 있어야 합니다. 또한 Ceph 은 CLI 나 대시보드를 통해 쉽게 설정할 수 없는 매우 고급 설정이 있습니다. monitor 가 활성화 되기 전에 설정을 조정하기 위해서 `rook-config-override` ConfigMap 이 존재하며, config 프로퍼티를 사용하여 `ceph.conf` 파일의 내용을 변경할 수 있습니다. 이 configmap 은 모든 mon, mgr, osd, mds, 그리고 rgw 에 전파되어 `/etc/ceph/ceph.conf` 파일로 저장됩니다.

> **경고**: Rook 은 설정에 대해 정합성을 체크하지 않으며, 설정의 변경은 유저에게 책임이 있습니다.

만약 `rook-config-override` configmap 이 클러스터가 시작되기 전에 생성되었다면, Ceph 데몬은 자동으로 설정을 가져옵니다. 클러스터가 초기화 된 후에 configmap 이 변경된다면 설정을 적용하기 위해 데몬별로 재시작이 필요합니다.
- mons: 각 mon pod 을 재시작 하면서 모든 세 개의 mon 이 온라인 상태임을 확인합니다.
- mgrs: mgr pod 은 stateless 하고 필요할 때 재시작 할 수 있습니다. 다만, 재시작 시에 Ceph 대시보드가 사용 불가능합니다.
- OSDs: 한번에 하나씩의 OSD 만 재시작해야 하고, 재시작 할 떄마다 `ceph -s` 명령을 통해 클러스터가 "active/clean" 상태임을 확인합니다.
- RGW: stateless 하며, 필요할 떄 재시작할 수 있습니다.
- MDS: stateless 하며, 필요할 떄 재시작할 수 있습니다.

Pod 이 재시작 되고 나면, 신규 설정이 적용됩니다. 클러스터가 생성되기 전에 ConfigMap 이 만들어져 있으면, 데몬들은 첫 시작에 해당 설정을 사용합니다.

Ceph 데몬 pod 들의 재시작을 자동화 하려면, pod 스펙에 업데이트를 트리거해야 합니다. 가장 간단한 방법은 자동 재시작을 원하는 데몬에 대해 CephCluster CR 에 [annotation 혹은 labels](/ceph_storage/cluster_crd.html#annotations-and-labels) 을 추가하는 것입니다. 이렇게 하면 오퍼레이터는 롤링 업데이트를 수행합니다.

### Example
이 예제에는 기본 Pool 크기를 2로 설정하고 OSD 데몬에게 시작 시 OSD 의 weight 을 변경하지 않도록 지정합니다. 

> **경고**: Ceph 설정을 조심히 수정하세요. Rook 이 테스트한 샌드박스를 벗어난 설정을 변경하는 것은 데몬 장애를 일어키거나 잘못 사용하면 데이터가 손실될 수 있습니다.

Rook 오퍼레이터가 클러스터를 생성하면 Ceph 설정을 덮어쓸 수 있는 ConfigMap 이 생성됩니다. 데몬 Pod 이 시작될 때, Rook 이 생성한 기본 설정 위에 해당 ConfigMap 이 합쳐집니다.

오버라이드 설정의 기본값은 빈 값입니다. 관련 없는 속성을 없애면, 클러스터 생성 후 다음과 같은 기본값이 표시됩니다.

```bash
kubectl -n rook-ceph get ConfigMap rook-config-override -o yaml
```
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: ""
```

원하는 설정을 적용하기 위해서, ConfigMap 업데이트가 필요합니다. pod 이 다음으로 재시작 될 때 이 설정이 적용됩니다.
```bash
kubectl -n rook-ceph edit configmap rook-config-override
```

설정을 수정하고 저장합니다. 아래와 같이 `config` 프로퍼티에서 인덴트를 가지고 있어야 합니다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    osd crush update on start = false
    osd pool default size = 2
```

## OSD CRUSH Settings
[CRUSH Map](https://docs.ceph.com/en/latest/rados/operations/crush-map/) 을 확인하려면 아래 커맨드를 사용합니다.
```bash
ceph osd tree
```
### OSD Weight
CRUSH weight 은 각 OSD 에 분산되어야 할 데이터의 비율을 조정합니다. 즉, OSD 의 디스크 I/O 처리량이 각각 높거나 낮을 수 있음을 의미합니다.

기본적으로 OSD 는 저장 용량에 비례하여 weight 을 조정합니다. 드라이브 크기가 다양하더라도 모든 드라이브를 동일한 비율로 채움으로서 전체 클러스터 용량을 최대화 합니다. 대부분의 케이스에서 효과가 있지만, 다음과 같은 상황에서는 weight 을 변화시켜야 할 수 있습니다.
- 클러스터에 비교적 느린 OSD 또는 노드가 존재할 때, weight 을 줄이면 병목을 줄일 수 있습니다.
- Rook 0.3.1 이전 버전으로 프로비저닝 된 블루스토어 드라이브를 사용하고 있지 않을 경우, OSD weight 이 스토리지 용량에 비례하여 설정되지 않은 것을 알 수 있습니다. weight 을 변경하면 이 문제를 해결하고 클러스터 용량을 극대화 할 수 있습니다.

아래 예제는 600GiB 인 osd.0 의 weight 을 수정합니다.
```bash
ceph osd crush reweight osd.0 .600
```

### OSD Primary Affinity
사이즈가 1보다 큰 Pool 을 설정하면, 노드와 OSD 간에 데이터가 복제됩니다. 모든 데이터 청크에 대해 클라이언트가 데이터를 읽기 위해서는 Primary OSD 가 선택됩니다. Primary Affinity 설정을 사용하여 OSD 가 Primary 가 될 가능성을 제어할 수 있습니다. 이것은 OSD weight 설정과 비슷하지만, 스토리지 디바이스 읽기에만 영향을 미치고 용량이나 쓰기에는 영향을 미치지 않습니다.

아래 예제에서는 복제본 데이터를 보유하고 있는 다른 모든 OSD 를 사용할 수 없는 경우에 `osd.0` 이 Primary 로 선택되도록 합니다.
```bash
ceph osd primary-affinity osd.0 0
```

## OSD Dedicated Network
OSD 가 통신하기 위한 전용 네트워크를 이용하도록 Ceph 을 설정할 수 있습니다. [CEPH Network](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref/#ceph-networks) 문서를 참고하면 이에 대한 유용한 정보를 얻을 수 있습니다. 클러스터 전용 네트워크를 살정하게 된다면, OSD 는 하트비트, 복제, 리커버리 트래픽을 클러스터 전용 네트워크를 사용합니다. 단일 네트워크를 사용하는 것보다 나은 퍼포먼스를 이룰 수 있습니다.

이 기능을 활성화 하기 위해서 두가지 변경이 필요합니다.

### Use hostNetwork in the rook ceph cluster configuration
아래와 같이 [Ceph Cluster CRD 설정](/ceph_storage/cluster_crd.html#samples) 에서 `hostNetwork` 설정을 활성화합니다.
```yaml
  network:
    provider: host
```
> 중요: 이 설정은 동작 중인 Rook 클러스터에서는 반영되지 않습니다. 클러스터가 처음 생성될 때만 설정이 가능합니다.

### Define the subnets to use for public and private OSD networks
커스텀 네트워크 설정을 위해 `rook-config-override` 설정을 수정합니다:
```bash
kubectl -n rook-ceph edit configmap rook-config-override
```

어떤 서브넷이 퍼블릭 네트워크이고 어떤 서브넷이 프라이빗 네트워크인지 설정하는 커스텀 설정을 에디터에서 추가합니다.
```yaml
apiVersion: v1
data:
  config: |
    [global]
    public network =  10.0.7.0/24
    cluster network = 10.0.10.0/24
    public addr = ""
    cluster addr = ""
```

rook-config-override configmap 업데이트를 적용하고 나면, 설정을 적용시키기 위해 OSD pod 을 재시작 해야 합니다. OSD 는 하나씩 재시작 하며, 재시작 할 떄마다 `ceph -s` 명령을 사용하여 클러스터가 "active/clean" 상태임을 확인합니다.

## Phantom OSD Removal
디스크가 없는 OSD 가 있다면, 지금부터 설명할 방법으로 이런 "Phantom OSD" 를 제거할 수 있습니다. 아래와 같이 확인합니다.
```bash
ceph osd tree
```

예제의 아웃풋은 다음과 같습니다.
```bash
ID  CLASS WEIGHT  TYPE NAME STATUS REWEIGHT PRI-AFF
-1       57.38062 root default
-13        7.17258     host node1.example.com
2   hdd  3.61859         osd.2                up  1.00000 1.00000
-7              0     host node2.example.com   down    0    1.00000
```

결과의 `node2.example.com` 노드는 디스크가 없으며, "Phantom OSD" 입니다.

첫번째 컬럼의 ID 를 사용하여 지웁니다. 위의 예제에서 ID 는 `-7` 입니다. 커맨드는 아래와 같습니다.
```bash
$ ceph osd out <ID>
$ ceph osd crush remove osd.<ID>
$ ceph auth del osd.<ID>
$ ceph osd rm <ID>
```

Phantom OSD 가 삭제되었는지 확인하려면, 아래 커맨드를 재실행하여 해당하는 ID 의 OSD 가 사라졌는지 확인합니다.
```bash
ceph osd tree
```

## Auto Expansion of OSDs
### Prerequisites
1) `storageClassDeviceSet` 설정과 함께 다이나믹 프로비저닝 환경에서 [PVC based 클러스터](/ceph_storage/cluster_crd.html#pvc-based-cluster) 가 배포되어 있어야 함
2) Rook [Toolbox](/ceph_tools/toolbox.html) 를 생성

> Note: `auto-grow-storage` 스크립트를 사용하기 위해서는 [Prometheus 오퍼레이터](/ceph_storage/prometheus_monitoring.html#prometheus-operator) 와 [Prometheus 인스턴스](/ceph_storage/prometheus_monitoring.html#prometheus-instances)가 있어야 합니다.

### To scale OSDs Vertically
OSD 가 near-full 임계점에 도달했을 때, PVC based 클러스터에서 OSD 의 사이즈를 자동으로 키우기 위해서 아래 스크립트를 실행합니다.
```bash
tests/scripts/auto-grow-storage.sh size  --max maxSize --growth-rate percent
```

> growth-rate 퍼센트는 OSD 용량의증가율을 나타내고, maxSize 는 최대 디스크 크기를 나타냅니다.

예를 들어, OSD 사이즈를 30% 늘리고 디스크 사이즈를 1Ti 로 설정하려면 아래와 같이 실행합니다.
```bash
./auto-grow-storage.sh size  --max 1Ti --growth-rate 30
```

### To scale OSDs Horizontally
OSD 가 near-full 임계점에 도달했을 때, PVC based 클러스터에서 OSD 의 갯수를 자동으로 늘리기 위해서 아래 스크립트를 실행합니다.
```bash
tests/scripts/auto-grow-storage.sh count --max maxCount --count rate
```

> count 는 추가할 OSD 수를 나타내고, maxCount 는 클러스터에서 지원할 디스크 수를 나타냅니다.

예를 들어, OSD 수를 3으로 늘리고 maxCount 가 10개라면 아래와 같이 실행합니다.
```bash
./auto-grow-storage.sh count --max 10 --count 3
```
