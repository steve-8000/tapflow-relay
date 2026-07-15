# tapflow-relay

[tapflow](https://github.com/jo-duchan/tapflow) 셀프호스팅 relay — `clab-cluster`에 ArgoCD GitOps로 배포되어 https://ios.clab.one 에서 서빙된다. iOS/Android 시뮬레이터 스트리밍 대시보드 + REST API + MCP 브릿지의 relay 역할만 담당한다.

## 구성

- `deploy/k8s/statefulset.yaml` — relay StatefulSet (replica=1, SQLite 단일 writer 제약 — 절대 스케일하지 않음). `/app/.tapflow-data`에 PVC(`local-path`, 5Gi) 마운트 — 세션 DB, 빌드 업로드, 녹화본(72h 자동 만료) 저장.
- `deploy/k8s/service.yaml` — ClusterIP :80 → :4000.
- `deploy/k8s/ingress.yaml` — `ios.clab.one`, traefik + cert-manager(letsencrypt-prod).
- 이미지: `ghcr.io/steve-8000/tapflow-relay:latest` — [jo-duchan/tapflow](https://github.com/jo-duchan/tapflow)의 공식 `Dockerfile`을 그대로 빌드(공식 배포 이미지가 없어 자체 빌드/게시), linux/amd64.

## 클러스터에 없는 것

relay는 대시보드/API/DB만 제공한다. 실제 iOS Simulator를 구동하는 **Mac agent는 macOS + Xcode 필수**이므로 clab-cluster(Linux 노드)에는 올릴 수 없다. Mac에서 `tapflow agent start --relay wss://ios.clab.one --token <agent-PAT>`로 별도 실행한다.

## 필요 시크릿 (git 미관리, kubectl로 직접 적용)

- `ghcr-pull-secret` — private GHCR pull (다른 steve-8000 앱과 동일).
- `tapflow-env` — `JWT_SECRET` (relay 세션 서명 키, 고정값).

## 배포

ArgoCD `clab-app-root`(app-of-apps)가 `clab-one/clab-app` 저장소의 `argocd/applications/tapflow-relay.yaml`을 통해 이 저장소를 자동 sync한다.
