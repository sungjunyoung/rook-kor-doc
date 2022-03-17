# OSD Management

Ceph Object Storage Daemon (OSD) 는 Ceph 스토리지 플랫폼의 심장입니다. 각 OSD 는 로컬 디바이스를 관리하고 분산 스토리지를 제공합니다. Rook 은 CephCluster 커스텀 리소스에서 OSD 의 복잡성을 숨기고 안정적으로 OSD 를 관리합니다. 이 가이드에서는 OSD 를 설정하는 몇가지 시나리오를 살펴봅니다.

## OSD Health
[rook-ceph-tools pod](toolbox.html) 는 Ceph 툴을 실행할 수 있는 간단한 환경을 제공합니다. 이 문서에서 언급되는 `ceph` 커맨드는 toolbox 내에서 실행되어야 합니다.

toolbox 가 만들어지면, OSD 와 PG 들을 조회해 클러스터의 상태를 분석하기 위해 `ceph` 커맨드를 pod 안에서 실행합니다. OSD 를 분석하기 위한 몇가지 커맨드들입니다.
```bash
ceph status
ceph osd tree
ceph osd status
ceph osd df
ceph osd utilization
```
```bash
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```

## Add an OSD
[QuickStart 가이드](/quickstart.html) 는 클러스터를 생성하고 몇 개의 OSD 와 함께 시작하는 기본적인 절차를 제공합니다. OSD 를 설정하기 위한 자세한 내용은 [Cluster CRD](/ceph_storage/cluster_crd.html) 문서를 참고하세요. OSD 가 정상적으로 생성되지 않았다면, [Ceph 트러블슈팅 가이드](/ceph_tools/common_issues.html) 를 참고합니다.

더 많은 OSD 를 추가하기 위해서, Rook 은 클러스터에 추가되는 노드와 디바이스를 watch 합니다. 클러스터 커스텀 리소스의 `storage` 섹션에 설정된 필터나 다른 설정과 일치하는 노드 혹은 디바이스가 추가될 경우, 오펄에티너는 자동으로 신규 OSD 를 생성합니다.

## Add an OSD on a PVC
raw 블록 스토리지 프로바이더와 함께 스토리지가 동적으로 프로비저닝 될 수 있는 환경에서는, OSD 가 PVC 를 백엔드로 사용할 수 있습니다. [Cluster CRD](/ceph_storage/cluster_crd.html) 토픽에서 `storageClassDeviceSets` 문서를 참고하세요.

더 많은 OSD 를 추가하기 위해서, 클러스터 CR 에서 `count` 를 증가시킴으로서 더 많은 디바이스 셋을 추가할 수 있습니다. 오퍼레이터는 자동으로 업데이트된 클러스터 CR 을 통해 OSD 를 생성하게 됩니다.

## Remove an OSD
디스크 fault 나 재설정을 위해 OSD 를 제거하기 위해서, 데이터의 안정성을 유지하기 위해 아래의 절차를 따릅니다.
- OSD 를 삭제해도 클러스터 내에 충분한 공간이 있는지 확인합니다.
- 데이터 리밸런싱을 위해 남아있는 OSD 와 PG 의 상태가 정상적인지 확인합니다.
- 한번에 너무 많은 OSD 를 제거하지 않습니다.
- 여러 OSD 를 제거할 때, 리밸런싱을 기다립니다.

만약 모든 PG 가 `active+clean` 상태이고, 공간 부족에 대한 경고가 없다면 데이터가 완전히 복제되어 계속 진행해도 안전함을 의미합니다. 만약 한 OSD 에 장애가 발생하면 PG 가 완전히 clean 한 생태가 아니더라도 계속 진행해야 합니다.

### Host-based cluster
CephCluster 커스텀리소스를 업데이트합니다. CR 의 설정에 따라, 설정에서 디바이스 리스트에서 삭제하거나 디바이스 필터를 업데이트 해야 할 수 있습니다. 만약 여러분이 `useAllDevices: true` 설정을 사용하고 있다면, CR 을 변경할 필요가 없습니다.

**중요: host-based 클러스터에서는 디스크가 삭제되기 전에 Rook 이 이전 OSD 를 감지하고 다시 생성하지 못하도록 OSD 제거 단계를 수행하는 동안 Rook 오퍼레이터를 중지해야 할 수 도 있습니다.**

Rook 오퍼레이터를 중지하려면, `kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=0` 을 수행합니다.

(1) OSD 를 삭제하고 (2.a) 데이터를 삭제하고, (2.b) Rook 오퍼레이터를 다시 시작하기 전 디스크를 교체 하는 절차로 진행해야 합니다.

완료하고 나면, `kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=1` 커맨드로 오퍼레이터를 다시 시작할 수 있습니다.

### PVC-based cluster
클러스터에서 스토리지를 축소시키거나 장애가 발생한 PVC 로 생성된 OSD를 제거하려면,
1. CephCluster CR 에서 `storageClassDeviceSets` 의 OSD 숫자를 줄입니다. 여러 디바이스 셋을 가지고 있다면, 아래에서 index 0 을 변경해야 할 수 있습니다.
    - `kubectl -n rook-ceph patch CephCluster rook-ceph --type=json -p '[{"op": "replace", "path": "/spec/storage/storageClassDeviceSets/0/count", "value":<desired number>}]'`
    - 원하는 숫자로 OSD `count` 를 줄입니다. Rook 는 OSD 제거에 대해서는 자동으로 액션을 수행하지 않습니다.
2. 장애가 발생한 OSD 가 사용하고 있는 PVC 를 아래 커맨드로 확인합니다.
    - `kubectl -n rook-ceph get pvc -l ceph.rook.io/DeviceSet=<deviceSet>`
3. 삭제할 OSD 를 확인합니다.
    - PVC 에 할당된 OSD 는 PVC 의 라벨로 확인할 수 있습니다.
    - `kubectl -n rook-ceph get pod -l ceph.rook.io/pvc=<orphaned-pvc> -o yaml | grep ceph-osd-id`
    - 예시로, 위 커맨드는 `ceph-osd-id: "0"` 을 리턴합니다.
    - OSD 를 제거하기 위해 OSD ID 를 기억합니다.

나중에 디바이스 셋의 count 를 증가시키면, 오퍼레이터는 현재 존재하는 OSD PVC 보다 높은 index 로 PVC 를 생성합니다.

### Confirm the OSD is down
장애가 발생한 OSD 를 제거하길 원한다면, OSD Pod 은 `CrashLoopBackoff` 상태이거나, toolbox 에서의 `ceph` 컴맨드에서 OSD 가 `down` 상태일 것입니다. 정상적인 OSD 를 제거하려면, toolbox 에서 `kubectl -n rook-ceph scale deployment rook-ceph-osd-<ID> --replicas=0` 명령을 수행하고, `ceph osd down osd.<ID>` 를 수행합니다.

### Purge the OSD from the Ceph cluster
OSD 제거는 [rook-ceph-purge-osd job](https://github.com/rook/rook/blob/release-1.8/deploy/examples/osd-purge.yaml) 으로 자동 수행할 수 있습니다. osd-purge.yaml 에서, `<OSD-IDs>` 를 삭제하기 원하는 ID 로 변경합니다.
- Job 을 수행합니다: `kubectl create -f osd-purge.yaml`
- Job 이 완료되면, 성공했는지 확인합니다: `kubectl -n rook-ceph logs -l app=rook-ceph-purge-osd`
- 끝나고 난 뒤, Job 을 삭제합니다: `kubectl delete -f osd-purge.yaml`

수동으로 OSD 를 제거하려면, 아래 섹션을 참고합니다. 하지만, 운영 에러 방지를 위해 위의 job 가이드를 따르는 것을 추천합니다.

### Purge the OSD manually
만약 OSD purge job 이 실패하거나, 전체 제거 작업을 매뉴얼하게 진행하고 싶다면, toolbox 로부터 아래의 커맨드들을 실행합니다.

1. Rook 으로부터 OSD PVC 를 분리합니다.
    - `kubectl -n rook-ceph label pvc <orphaned-pvc> ceph.rook.io/DeviceSetPVCId-`
2. OSD 를 `out` 으로 지정합니다. 이 작업은 Ceph 에게 해당 OSD 의 데이터들을 다른 OSD 로 옮기도록 트리거합니다. (backfilling)
    - `ceph osd out osd.<ID>`
3. 다른 OSD 로 데이터들이 backfilling 완료되길 기다립니다.
    - `ceph status` 명령으로 전체 PG 가 `active+clean` 상태임을 확인합니다. 그렇다면, 이제 디스크를 제거해도 안전합니다.
4. Ceph 클러스터에서 OSD 를 제거합니다.
    - `ceph osd purge <ID> --yes-i-really-mean-it`
5. CRUSH map 에서 OSD 가 노드로부터 제거되었는지 확인합니다.
    - `ceph osd tree`

오퍼레이터는 Ceph 으로부터 OSD 가 제거해도 안전하다고 판단되면 OSD Deployment 를 자동으로 삭제할 수 있습니다. 위 절차 이후에, 데이터들이 다른 OSD 로 모두 옮겨져 제거해도 안전하다고 판단됩니다. CephCluster CR 에 아래 설정이 활성화 되어 있어야 합니다.
```bash
removeOSDsIfOutAndSafeToRemove: true
```

혹은, deployment 를 수동으로 삭제해도 됩니다.
- `kubectl delete deployment -n rook-ceph rook-ceph-osd-<ID>`

PVC-based 클러스터에서는, OSD 에 연결되어 있지 않은 PVC 들도 삭제해줍니다.

### Delete the underlying data
OSD 가 실행되었던 디바이스를 클린업하고 싶다면, [클러스터 클린업](/ceph_storage/cleanup.html) 토픽을 참고합니다.

## Replace an OSD
장애가 발생한 디스크를 교체하려면,
1. [OSD 삭제](#remove-an-osd) 섹션의 절차를 수행합니다.
2. 물리 디바이스를 교체하고, 디바이스가 정상적으로 attach 되었는지 확인합니다.
3. cluster CR 이 신규 디바이스를 찾는지 확인합니다. `useAllDevices: true` 설정을 사용하고 있다면 스킵해도 됩니다. 만약 개별 디바이스들을 지정해 리스팅하고 있거나, 디바이스 필터를 사용한다면 CR 을 업데이트합니다.
4. 오퍼레이터는 새로운 디바이스를 추가하거나 CR 을 업데이트 한 후 수 분 내로 신규 OSD 를 생성해야 합니다. 새로운 OSD 가 자동으로 생성되지 않는다면, OSD 생성을 트리거하기 위해 오퍼레이터를 재시작합니다. 
5. toolbox 에서 `ceph osd tree` 커맨드를 입력해 OSD 가 생성되었는지 확인합니다.

삭제한 OSD 와 신규 OSD 의 ID 는 달라집니다.