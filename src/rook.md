# Rook

Rook 은 플랫폼, 프레임워크를 제공하고, cloud-native 환경과 통합된 다양한 스토리지를 지원하는 오픈소스 **cloud native 스토리지 오케스트레이터** 입니다.

Rook 은 배포, 부트스트래핑, 설정, 프로비저닝, 스케일링, 업그레이드, 마이그레이션, 재난 복구, 모니터링, 리소스 관리를 자동화하여 스토리지 소프트웨어를 self-managing, self-scailing, self-healing 스토리지 서비스로 바꿔줍니다. Rook 은 cloud-native 컨테이너 관리, 스케줄링, 오케스트레이션 플랫폼 아래에서 그 역할을 수행합니다.

Rook 은 (Kubernetes 의) 확장 기능을 사용하여 cloud native 환경에 통합됩니다. 그리고, 스케줄링, 라이프사이클 관리, 리소스 관리, 보안, 모니터링, 그리고 유저 경험에 있어 seamless 한 경험을 제공합니.

Ceph operator 는 프로덕션 스토리지 플랫폼을 몇년간 제공하며 2018년 12월에 Rook v0.9 가 릴리즈되며 안정화되었습니다.

## Quick Start Guide

몇 가지의 `kubectl` 커맨드를 사용하여 Ceph 을 클러스터에 구성할 수 있습니다. [QuickStart](./quickstart.html) 가이드를 참고하여 Ceph operator 를 시작하세요!

## Designs

[Ceph](https://docs.ceph.com/en/latest/) 은 블록 스토리지, 오브젝트 스토리지, 공유 파일시스템에 프로덕션 레벨로 수년간 안정화된 확장성있는 분산 스토리지 솔루션입니다. [Ceph overview](https://rook.io/docs/rook/v1.8/ceph-storage.html) 를 참고하세요.

자세한 디자인 문서는, [design docs](https://github.com/rook/rook/tree/master/design) 를 참고하세요.

## Need help? Be sure to join the Rook Slack

질문이 있다면, 주저하지 말고 [Slack channel](https://rook-io.slack.com/) 에 문의하세요. 