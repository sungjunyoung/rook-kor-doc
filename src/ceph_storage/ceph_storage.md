# Ceph Storage

Ceph 은 **블록 스토리지**, **오브젝트 스토리지**, 그리고 **공유 파일시스템** 을 지원하는 수년간 프로덕션 환경에서 사용된 확장성있는 분산 스토리지 솔루션입니다.

## Design

Rook 은 쿠버네티스의 요소들을 사용해서 Ceph 스토리지를 쿠버네티스 클러스터에서 동작하게 만들어 줍니다. 쿠버네티스 위에서 Ceph 이 동작함과 동시에, 쿠버네티스의 어플리케이션들은 Rook 에 의해 관리되는 블록 디바이스나 파일시스템을 마운트하여 사용할 수 있습니다. 또한, S3/Swift API 를 통해 오브젝트 스토리지를 사용할수도 있습니다. Rook 오퍼레이터는 스토리지가 정상적인 상태로 유지되도록 모니터링하고 스토리지 컴포넌트들을 설정합니다.

Rok 오퍼레이터는 스토리지 클러스터를 부트스트랩하고 모니터링 할 수 있는 모든 요소들을 갖춘 단순한 컨테이너입니다. 이 오퍼레이터는 [Ceph monitor pod](/ceph_tools/monitor_health.html)과 OSD 데몬, 다른 Ceph 데몬들을 시작하고 모니터링합니다. 또한 서비스를 실행시키는데 필요한 리소스들과 Pod 을 초기화하고 Pool, 오브젝트 스토어, 파일시스템을 위한 CRD 를 관리합니다.

오퍼레이터는 클러스터가 정상적인 상태가 되도록 모니터링 합니다. Ceph mon 들이 오퍼레이터에 의해 시작되거나, 필요에 따라 failover 되며, 클러스터가 커지거나 작아질 때마다 조정이 이루어집니다. 또한 커스텀 리소스의 변경 사항을 보고 변경 사항을 조정하기도 합니다.

Rook 은 여러분의 Pod 에 스토리지를 마운트하기 위해서 자동으로 Ceph-CSI driver 를 설정합니다.

![ceph_storage_01](ceph_storage_01.png)

`rook/ceph` 이미지는 클러스터를 관리하기 위한 모든 필요한 툴들이 포함되어 있습니다. placement group, crush map 같은 많은 Ceph 의 컨셉들은 숨겨져 있으며, 걱정할 필요가 없습니다. 대신에 Rook 은 물리적인 리소스, pool, volume, filesystem, bucket 에 대한 심플한 사용자 경험을 제공하빈다. 동시에, Ceph tool 을 통해 advanced 한 설정들을 적용할 수 도 있습니다.

Rook 은 golang 으로 개발되었고, Ceph 은 C++ 로 개발되었습니다. 우리는 이 조합이 최고를 제공한다고 믿습니다.