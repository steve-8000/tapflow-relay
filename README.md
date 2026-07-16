# tapflow-relay

[tapflow](https://github.com/jo-duchan/tapflow) 셀프호스팅 relay — `clab-cluster`에 ArgoCD GitOps로 배포되어 https://ios.clab.one 에서 서빙된다. iOS/Android 시뮬레이터 스트리밍 대시보드 + REST API + MCP 브릿지의 relay 역할만 담당한다.

## 구성

- `deploy/k8s/statefulset.yaml` — relay StatefulSet (replica=1, SQLite 단일 writer 제약 — 절대 스케일하지 않음). `/app/.tapflow-data`에 PVC(`local-path`, 5Gi) 마운트 — 세션 DB, 빌드 업로드, 녹화본(72h 자동 만료) 저장.
- `deploy/k8s/service.yaml` — ClusterIP :80 → :4000. 브라우저(대시보드)는 이걸 통해 `ios.clab.one` ingress로 접근.
- `deploy/k8s/service-agent-nodeport.yaml` — NodePort `30400` → :4000. **Mac agent 전용 직결 경로.** tapflow 공식 가이드가 "agent→relay는 LAN `ws://` 직결" 권장 — agent를 대시보드와 같은 public ingress(`wss://ios.clab.one`, traefik+TLS)로 물렸더니 30fps 비디오 스트림 시작 직후 1초 간격 재연결 루프가 발생했음(리버스프록시가 지속 바이너리 스트림과 안 맞음). NodePort로 traefik/TLS를 완전히 우회해서 해결.
- `deploy/k8s/ingress.yaml` — `ios.clab.one`, traefik + cert-manager(letsencrypt-prod). **브라우저 전용 — agent는 여기로 연결하지 않는다.**
- 이미지: `ghcr.io/steve-8000/tapflow-relay:latest` — [jo-duchan/tapflow](https://github.com/jo-duchan/tapflow)의 공식 `Dockerfile`을 그대로 빌드(공식 배포 이미지가 없어 자체 빌드/게시), linux/amd64.

## 클러스터에 없는 것

relay는 대시보드/API/DB만 제공한다. 실제 iOS Simulator를 구동하는 **Mac agent는 macOS + Xcode 필수**이므로 clab-cluster(Linux 노드)에는 올릴 수 없다. Mac에서 아래처럼 별도 실행한다:

```sh
tapflow agent start --relay ws://219.255.103.189:30400 --token <agent-PAT>
```

`wss://ios.clab.one`(ingress)이 아니라 **NodePort 직결**(`ws://<node-ip>:30400`)을 반드시 써야 한다 — 위 서비스 구성 이유 참고.

## 시뮬레이터 구성 & 기기 배정 컨벤션

연결된 Mac에는 시뮬레이터 2대만 유지한다(그 외 posy 프로젝트 전용 `Posy Reference SE 3 QA` 제외):

| 기기 | UUID | 용도 |
|---|---|---|
| iPhone 17 Pro (iOS 26.5) | `8614C140-C044-40C2-9747-351582A99A4D` | **MCP 자동화 전용** |
| iPhone 17 (iOS 26.5) | `D043EE5F-3CD9-477F-AC0A-A98D48EBDA10` | **대시보드 수동 리뷰 전용** |

relay는 대시보드 로그인 쿠키와 MCP의 PAT(agent 스코프가 아닌 PAT)를 똑같이 `role: browser`로 분류한다(`classifyConnection`, `lib/connectionAuth.js`) — 즉 기기 세션 락(`busy`)은 계정이 아니라 **기기당 browser-role 연결 1개** 단위다. 같은 기기를 대시보드와 MCP가 동시에 잡으면 서로 "in use"로 충돌한다. 위 배정을 지키면 겹치지 않는다.

`@tapflowio/mcp-server`는 기기를 코드로 강제 고정하는 기능이 없다(모든 툴이 매 호출마다 `sessionId`를 인자로 받음) — 컨벤션은 MCP를 호출하는 쪽(에이전트)이 항상 iPhone 17 Pro의 sessionId를 골라 `connect_device`하는 방식으로 지킨다.

세션이 정상 종료 없이 끊겨 `busy: true`로 고정되는 경우, relay 재시작(`kubectl rollout restart statefulset/tapflow-relay -n tapflow`)으로 정리한다(세션 상태는 SQLite가 아니라 relay 메모리에만 있음).

## 필요 시크릿 (git 미관리, kubectl로 직접 적용)

- `ghcr-pull-secret` — private GHCR pull (다른 steve-8000 앱과 동일).
- `tapflow-env` — `JWT_SECRET` (relay 세션 서명 키, 고정값).

## 배포

ArgoCD `clab-app-root`(app-of-apps)가 `clab-one/clab-app` 저장소의 `argocd/applications/tapflow-relay.yaml`을 통해 이 저장소를 자동 sync한다.
