# Admission Controller

어드미션 컨트롤러는 쿠버네티스 리소스가 실제 반영되기 전 요청을 가로채어 인증 및 인가를 확인합니다.

Rook 이 커스텀 리소스 세팅과 함께 올바르게 설정되었는지 확인하기 위해 Rook 어드미션 컨트롤러를 활성화가 권장됩니다.

## Quick Start

Rook 어드미션 컨트롤러를 배포하기 위해서, 설정을 자동화 하기 위한 헬퍼 스크립트를 제공합니다.

이 스크립트는 다음과 같은 작업을 하도록 돕습니다.
- cert-manager 를 사용해 certificate 를 생성
- 클러스터로부터 적절한 CA 번들로 ValidatingWebhookConfig 를 채워 생성

아래 커맨드를 실행합니다.
```sh
kubectl create -f deploy/examples/crds.yaml -f deploy/examples/common.yaml
tests/scripts/deploy_admission_controller.sh
```

Secret 이 배포되고 나면, operator 를 배포합니다.
```sh
kubectl create -f deploy/examples/operator.yaml
```

완료되면, 오퍼레이터는 어드미션 컨트롤러 Deployment 를 자동으로 싱행하고, Rook 리소스에 대한 요청을 Webhook 이 인터셉트 하기 시작합니다.