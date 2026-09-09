# CLAUDE.md — cleanbrain-me-infra

이 파일은 이 repository와 `cleanbrain.me` 운영환경에 관련된 작업을 할 때
Claude Code 세션이 따라야 할 동작 규칙입니다.

기술적 세부사항(서버 스펙, Gateway 이름, RBAC, CI/CD 절차 등)의 canonical source는
[`README.md`](README.md)입니다. 이 파일은 그 내용을 중복하지 않고,
세션이 매번 다시 물어보지 않도록 "무엇을 먼저 확인해야 하는지"와
"어떤 제약을 지켜야 하는지"만 정리합니다.

## 이 repository의 역할

- `cleanbrain-me-infra`는 `cleanbrain.me` Hetzner K3s 클러스터에 배포되는
  모든 서비스의 production Kubernetes manifest와 운영 문서의 source of truth입니다.
- Application source code, Dockerfile, CI workflow는 각 application repository가 소유합니다
  (예: [`english-core-speaking`](https://github.com/cleanbrain-developer/english-core-speaking)).
- `cleanbrain.me`에 배포되는 신규 application repository에서 작업할 때도,
  production manifest는 이 repository에 작성되어야 하며 application repo에 중복해서 두지 않습니다.

## 새 세션에서의 context 복구 순서

`cleanbrain.me` 관련 작업 요청을 받으면 사용자에게 다시 묻기 전에 먼저:

1. 이 파일과 [`README.md`](README.md)를 읽는다 — server spec, Gateway 이름,
   namespace convention, cert-manager 정책, CI/CD 모델, RBAC 모델 등은 여기 이미 있다.
2. 대상 application repository의 `CLAUDE.md`/`README.md`를 읽는다.
3. 이 repo의 `kubernetes/` 아래 실제 manifest 현재 상태를 확인한다 (문서와 실제가 다를 수 있음).
4. 대상 application repo의 실제 source/Dockerfile/워크플로를 확인한다 — 문서만 보고
   runtime port, health endpoint, OAuth callback 등을 추측하지 않는다.
5. 위 정보와 현재 task 사이의 차이만 사용자에게 질문한다.

반복해서 묻지 말 것: Kubernetes/K3s 사용 여부, Gateway 이름, cert-manager 존재 여부,
Cloudflare proxy 설정, infra repo가 무엇인지 — 전부 README.md에 이미 있다.

## 신규 서비스 배포 시 지킬 것

- Shared Gateway(`cleanbrain-me-gateway`, namespace `cleanbrain-me-system`)와
  ClusterIssuer를 application마다 재생성하지 않는다. 새 서비스는 자기 namespace에
  `HTTPRoute`만 만들어 cross-namespace `parentRef`로 붙인다.
- Namespace는 `cleanbrain-me-<service-name>`, namespace 내부 리소스 이름은 짧게
  (`web`, `api`, `postgres` 등, README.md "Naming conventions" 참고).
- Public GHCR image라면 `imagePullSecrets`를 추가하지 않는다 — 과거 존재하지 않는
  Secret을 참조해서 `ImagePullBackOff`가 난 적이 있다 (README.md "GHCR image pull authentication").
- CI Kubernetes identity는 최소 권한으로 새로 설계한다. 기존 `english-core-speaking`의
  `ci-deployer` RBAC를 넓혀서 여러 서비스를 제어하게 만들지 않는다 — 서비스별로 분리한다.
- Stateful workload(PostgreSQL 등) 추가 전 `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB`/
  `DATABASE_URL`의 일관성을 반드시 검증한다 (과거 불일치로 Prisma 인증 실패 발생).
- 서버가 2 vCPU / 4 GB RAM / 40 GB disk라는 리소스 제약을 항상 고려한다.
  ArgoCD, Prometheus/Grafana/Loki, service mesh, multi-node, PostgreSQL HA는 현재 단계에서
  의도적으로 보류 중이며, 서비스 하나 추가한다는 이유로 임의로 도입하지 않는다.

## Bootstrap과 CI 권한 분리

- 최초 bootstrap(Namespace, Secret, StatefulSet, 최초 Deployment 등)은 cluster admin 권한으로
  수동 수행한다.
- 이후 지속적 배포는 GitHub Actions → SSH(`deploy` 사용자) → 최소권한 ServiceAccount만 사용한다.
- CI가 bootstrap을 할 수 있도록 RBAC를 넓히지 않는다. `/etc/rancher/k3s/k3s.yaml`,
  `system:masters`, `cluster-admin`은 CI에서 절대 사용하지 않는다.

## 절대 저장하지 않는 정보

Persistent context(이 파일, memory, 다른 문서)에는 다음 실제 값을 저장하지 않는다:
비밀번호, API key, OAuth client secret, ServiceAccount token, Kubernetes Secret 실제 값,
kubeconfig credential, SSH private key, GitHub token/PAT, session secret, DB 비밀번호.
구조와 리소스 이름만 기록한다 (예: "`HETZNER_SSH_PRIVATE_KEY`라는 GitHub Secret을 사용한다"는 OK,
실제 키 값은 금지).

## 실행 제한

명시적으로 요청받지 않은 상태에서 자동으로 수행하지 않는다:
`git commit`/`push`, production `kubectl apply`/`delete`, Secret 생성/수정,
Namespace/PVC 삭제, database destructive operation, SSH/firewall/서버 변경.
Repository 파일 수정은 요청에 따라 수행할 수 있으나 commit/push는 별도 확인을 받는다.

## 응답 언어

사용자 대상 설명은 한국어. Source code, command, YAML key, 파일 경로, 패키지명,
Kubernetes resource kind/name, 라이브러리/도구명, 기술 키워드는 영어 원문 유지.
Repository-facing 문서(README 등)는 기본적으로 영어로 작성한다.

## 운영환경 변경 시

실제 서버/클러스터 상태, 이 repo의 declarative manifest, 최근 확정된 architecture decision이
이 파일이나 README.md의 과거 내용과 다르면 과거 내용을 우선하지 않는다. 무엇이 바뀌었는지
설명하고, README.md와 이 파일을 canonical 최신 상태로 갱신한다 (append가 아니라 정리).
