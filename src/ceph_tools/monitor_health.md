# Monitor Health

분산 시스템에서 컴포넌트 다운은 언제든 발생할 수 있는 이슈입니다. Ceph 은 분산 시스템의 실패를 잘 다룰 수 있도록 설계되었습니다. 다음 레이어에서 Rook 은 관리자의 개입이 필요했던 Ceph 구성요소의 복구를 자동화하도록 처음부터 설계되었습니다. Monitor 의 상태는 Rook 이 모니터링하는 것 중 가장 중요한 부분입니다. Monitor 의 상태가 비정상적이라면 Rook 오퍼레이터는 클러스터를 정상화하고 재해로부터 보호하기 위한 조치를 취할 것입니다.

Ceph monitor (mon) 들은 분산 클러스터의 뇌입니다. mon 들은 데이터를 안전하게 보관하고 검색하는데 필요한 모든 메타데이터를 관리합니다. mon 이 정상적인 상태가 아니라면, 시스템의 모든 데이터를 잃을 수 있는 위험이 있다는 것입니다.

## Monitor Identity

Ceph 클러스터의 각각의 monitor 들은 구분자를 가지고 있습니다. 클러스터 내의 모든 컴포넌트들은 이 정보를 알고 있으며, 변경되지 않습니다. mon 의 구분자는 IP 주소입니다.

쿠버네티스 상에서 변경 불가능한 IP 주소를 가지기 위해서, Rook 은 각 monitor 의 서비스를 생성합니다. clusterIP 가 구분자로 사용됩니다.

monitor pod 이 시작될 때, podIP 가 바인딩되고, service IP 를 사용해 다른 컴포넌트들과 커뮤니케이션 합니다.

## Monitor Quorum

여러 mon 은 각각 메타데이터의 복제본을 가짐으로서 가용성을 제공합니다. Paxos 분산 알고리즘을 통해 클러스터의 상태를 합의를 통해 도출합니다. 클러스터에서 쿼럼을 설정하고 작업을 수행하려면 Paxos 에서 과반 이상의 mon 이 필요합니다. 과반 이상의 mon 이 동작 중이지 않으면, 클러스터에서 어떤 동작도 수행할 수 없습니다.

### How many mons?
대부분의 클러스터는 세 개의 mon 을 가지고 있습니다. 한 개의 mon 이 내려가더라도 과반 이상의 2/3 mon 이 정상적으로 동작 중임으로 클러스터가 정상 상태로 유지 될 수 있다는 뜻입니다.

최고의 가용성을 위해서, 홀수 개의 mon 이 필요합니다. 쿼럼을 유지하기 위해서는 50% 의 mon 은 충분하지 않습니다. 두 개의 mon 이 있다고 가정했을 때, 한 개의 mon 이 내려간다면 이것은 과반수 이상이 아니며, 그렇기 때문에 mon 두 개가 모두 정상화 될때까지 클러스터는 대기하게 됩니다. Rook 은 가용성을 위해 짝수 개의 mon 을 허용합니다. 한 개의 mon 이 남았을 때 여기서 쿼럼을 복구하기 위해서는 [재해 복구 가이드](disaster_recovery.html#restoring-mon-quorum) 를 참고하세요.

클러스터에 생성할 mon 의 갯수는 노드 손실 허용 갯수에 따라 달라집니다. mon 이 한 개라면, 쿼럼을 유지하기 위해서 어떤 노드도 죽으면 안됩니다. 세 개의 mon 이라면 한 개, 다섯 개라면 두 개 까지 허용됩니다. Rook 오퍼레이터는 하나의 mon 이 죽었을 때 자동으로 새로운 mon 을 배포하기 때문에, 일반적으로는 세개의 mon 을 띄웁니다. mon 이 많아질수록 클러스터 변경에 대한 오버헤드가 많아지며, 거대한 클러스터에서는 퍼포먼스 이슈가 있을 수 있습니다.

## Mitigating Monitor Failure
> Mon 실패 완화하기

mon 이 실패하는 어떤 이유든 (파워 Off, 소프트웨어 에러, hang, 등), mon 의 복구를 돕기 위한 여러 레이어에서의 완화 장치가 있습니다. 새로운 mon 을 배포하는 것 보다 이미 존재하는 mon 을 복구하는 것이 낫습니다. 

Rook 오퍼레이터는 mon 이 죽을 때 항상 재시작 시키기 위해서 mon 을 Deployment 로 배포합니다. mon 이 어떤 이유로 죽든 간에, 쿠버네티스는 Pod 을 항상 다시 시작할 것입니다.

mon 이 pod/node 재시작에 잘 대응하도록 하기 위해서, mon 의 메타데이터는 디스크에 영속적으로 저장됩니다. (CephCluster CR 의 `dataDirHostPath`) 혹은 `volumeClaimTemplate` 에 명시된 볼륨에 저장될 수 도 있습니다. 이는 mon 이 다시 시작될 때 이전에 존재했던 메타데이터로부터 시작할 수 있도록 합니다. 이 볼륨이 없이는 mon 은 재시작될 수 없습니다.

## Failing over a Monitor

만약 mon 상태가 정상적이지 않고, pod 이 재시작되거나 liveness probe 가 정상적인 백업을 가져오기에 충분하지 않은 경우, 오퍼레이터는 이런 mon 배포를 종료시키고 새로운 ID 를 가진 신규 mon 을 기동합니다. 이 오퍼레이션은 mon 쿼럼이 다른 mon 들에 의해 클러스터에서 유지되고 있어야 합니다.

오퍼레이터는 mon 의 health 를 45초마다 체크합니다. monitor 가 다운되면, 오퍼레이터는 다운된 mon 이 failover 하기까지 10분을 기다립니다. 이 두 인터벌은 CephCluster 커스텀 리소스 파라미터로 조정할 수 있습니다. 인터벌이 너무 짧으면, mon 이 지나치게 많이 failover 될 수 있습니다. 인터벌이 너무 길면, 다른 mon 이 브링업될 동안 쿼럼을 잃어버리게 될 수 있습니다.

```bash
healthCheck:
  daemonHealth:
    mon:
      disabled: false
      interval: 45s
      timeout: 10m
```

mon failover 를 테스트나 다른 목적으로 해보려면, mon deployment 의 replica 를 0으로 스케일다운 시키고 타임아웃동안 대기합니다. 오퍼레이터가 다시 시작되거나 CephCluster CR 이 업데이트 되는 경우와 같이 전체 reconcile 이 트리거 되면, 오퍼레이터가 자동으로 mon 을 다시 스케일업 합니다.

만약 mon pod 이 pending 상태이고, 노드 drain 등으로 노드에 할당되지 못한다면, 오퍼레이터는 mon failover 전에 타임아웃을 다시 기다립니다. 이럴 경우 mon failover 대기 시간이 두배로 증가합니다.

mon 자동 failover 를 비활성화하려면, `timeout` 을 `0` 으로 설정합니다. 이렇게 하면, mon 이 쿼럼을 벗어나더라도 Rook 이 다른 노드로 failover 시키지 않습니다. 유지 보수에 특히 유용합니다.

### Example Failover
Rook 은 mon-a, mon-b, mon-c 와 같은 네이밍으로 mon 을 생성합니다. mon-b 에 이슈가 생겨 pod 이 다운되었다고 가정해 봅시다.
```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mon
```
```bash
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-mon-a-74dc96545-ch5ns    1/1     Running   0          9m
rook-ceph-mon-b-6b9d895c4c-bcl2h   1/1     Error     2          9m
rook-ceph-mon-c-7d6df6d65c-5cjwl   1/1     Running   0          8m
```

failover 이후에, 새로운 mon 인 mon-d 가 생성된 것을 확인할 수 있습니다. 완전히 정상적인 mon 쿼럼이 형성되어 동작합니다.
```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mon
```
```bash
NAME                             READY     STATUS    RESTARTS   AGE
rook-ceph-mon-a-74dc96545-ch5ns    1/1     Running   0          19m
rook-ceph-mon-c-7d6df6d65c-5cjwl   1/1     Running   0          18m
rook-ceph-mon-d-9e7ea7e76d-4bhxm   1/1     Running   0          20s
```

toolbox 에서 mon 쿼럼 상태와 클러스터 상태를 확인할 수 있습니다.
```bash
ceph -s
```
```bash
 cluster:
   id:     35179270-8a39-4e08-a352-a10c52bb04ff
   health: HEALTH_OK

 services:
   mon: 3 daemons, quorum a,b,d (age 2m)
   mgr: a(active, since 12m)
   osd: 3 osds: 3 up (since 10m), 3 in (since 10m)
 ...
```