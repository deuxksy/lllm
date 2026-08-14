# Repository Rules & Guidance

LiteLLM 기반 LLM 프록시 프로젝트의 공통 개발, 배포 및 운영 지침입니다.

## Project

LiteLLM 기반 LLM 프록시. Z.ai, ModelArk, 로컬 모델을 단일 OpenAI 호환 엔드포인트로 통합.
Tailscale K8s Operator로 `lllm.bun-bull.ts.net:4000` 노출.

## Commands

### 배포 (K8s — 운영)
```bash
sops -d k8s/secret.enc.yaml > k8s/secret.yaml && kubectl --context orbstack apply -k k8s/
```

### 배포 (Docker Compose — 로컬)
```bash
sops -d --input-type dotenv --output-type dotenv compose/.env.encrypted > compose/.env && docker compose -f compose/compose.yml up -d
```

### Secret 암호화 (변경 시)
```bash
sops -e --input-type dotenv --output-type dotenv --age age1qw643dna4spaup6sr5ap0jf039ncjd54e8ekvrfy6p6x96ys2y4qn5vcsy compose/.env > compose/.env.encrypted
sops -e --age age1qw643dna4spaup6sr5ap0jf039ncjd54e8ekvrfy6p6x96ys2y4qn5vcsy k8s/secret.yaml > k8s/secret.enc.yaml
```

### 확인
```bash
set -a
source compose/.env
set +a
curl http://lllm.bun-bull.ts.net:4000/v1/models -H "Authorization: Bearer ${LITELLM_MASTER_KEY}"
```

### MLX Gemma 서버 (로컬)
```bash
APC_ENABLED=1 \
APC_BLOCK_SIZE=16 \
APC_NUM_BLOCKS=4096 \
APC_EXACT_CACHE_ENTRIES=8 \
mlx_vlm.server \
  --model lmstudio-community/gemma-4-12B-it-MLX-4bit \
  --host 0.0.0.0 \
  --port 8080 \
  --max-kv-size 131072
```

- `MAX_KV_SIZE`는 `prompt_tokens + max_tokens` 전체 한도다.
- Hermes는 `context_length: 131072`, `max_tokens: 65536`으로 맞춘다.
- APC는 기본 비활성화다. 현재 Gemma는 exact-cache path를 사용하며 최근 prompt snapshot을 최대 8개 유지한다.
- `APC_NUM_BLOCKS`는 block-cache 호환 모델의 최대 65,536-token pool 설정이다.
- APC는 process 재시작 시 초기화되므로 첫 요청은 cold request다.
- 반복 요청은 `/v1/cache/stats`의 `exact_hits` 증가로 확인한다. Hermes usage에는 `cached_tokens`가 전달되지 않을 수 있다.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/cache/stats
```

### 모델 추가/수정
1. `compose/litellm/config.yaml` 수정
2. `k8s/configmap.yaml`에도 동일 반영 (두 파일은 수동 동기화)
3. Compose: `docker compose -f compose/compose.yml restart`
4. K8s: `kubectl --context orbstack apply -k k8s/` (ConfigMap 변경 시 파드 재시작 필요 — `kubectl --context orbstack rollout restart deployment litellm -n lllm`)

### K8s 디버깅
```bash
kubectl --context orbstack get pods -n lllm -o wide
kubectl --context orbstack logs -n lllm -l app=litellm --tail=50
kubectl --context orbstack get pods -n tailscale  # operator + proxy 파드
```

## Architecture

### 모델 라우팅
- **Z.ai** (priority 1): glm-5.2, glm-5.2[1m], glm-5.1, glm-5-turbo, glm-5, glm-4.7, glm-4.5-air
- **ModelArk** (priority 2 fallback): glm-5.1, glm-4.7 + 전용 6개 (dola-seed-2.0-pro/lite/code, bytedance-seed-code, kimi-k2.5, gpt-oss-120b)
- **Local**: qwen3-vl-4b (eve), qwen3.5-4b (girl), google/gemma-4-12b-it (Mac MLX)
- 동일 model_name의 priority로 Z.ai → ModelArk 자동 폴백

### 두 배포 경로
- `compose/`: Docker Compose — 로컬 개발/테스트용. 포트 32400→4000
- `k8s/`: Kubernetes (OrbStack) — 운영. Tailscale Operator로 노출

### Secret 관리
- SOPS + age로 암호화. age 키는 `~/.config/sops/age/keys.txt`
- 평문 `.env`, `k8s/secret.yaml`은 `.gitignore` 제외
- `.sops.yaml`에 파일별 creation_rules 정의
- `sops exec-env`는 dotenv 포맷 미인식 — 반드시 2단계로 복호화 (`sops -d > .env && docker compose up`)

### K8s 리소스
- Service에 `tailscale.com/expose: "true"`, `tailscale.com/hostname: "lllm"` annotation으로 Tailscale 노출
- 헬스체크는 TCP 프로브 사용 (LiteLLM `/health`가 master key 인증 필요)
- 메모리 limit 1Gi (512Mi에서 OOMKilled 이력)
- OrbStack NodePort는 localhost(127.0.0.1)에만 바인딩 — Tailscale 외부 접근 불가

### Tailscale ACL 요구사항
- `tag:k8s-operator`가 `tag:k8s`의 owner여야 함 (`tagOwners`에 `"tag:k8s": ["tag:k8s-operator"]`)
- OAuth client는 Trust credentials 페이지에서 `Devices Core`, `Auth Keys`, `Services` Write + Tags: `tag:k8s-operator`, `tag:k8s` 할당 필수
  - 태그 누락: 400 "requested tags are invalid or not permitted"
  - Scope 누락: 403 "calling actor does not have enough permissions"
- ACL 정책은 `~/git/sylph/policy.hujson`에서 관리 (CI/CD 자동 동기화)

### Tailscale Operator 관리
```bash
helm --kube-context orbstack install tailscale-operator tailscale/tailscale-operator -n tailscale --set oauth.clientId=ID --set oauth.clientSecret=SECRET
```

### 완전 삭제 후 재설치

> namespace와 Helm release를 삭제하는 작업이다. 대상 확인과 사용자 승인 후에만 실행한다.

```bash
# 모든 kubectl/helm 명령에 orbstack context 필수 (기본 context는 ecoai.dxai.kr)
kubectl --context orbstack delete namespace lllm tailscale --force --grace-period=0
# finalizer stuck 시: kubectl --context orbstack replace --raw /api/v1/namespaces/NAME/finalize -f <(kubectl --context orbstack get namespace NAME -o json | jq '.spec.finalizers = []')
helm --kube-context orbstack uninstall tailscale-operator -n tailscale  # namespace 삭제 후에도 release 잔존
helm --kube-context orbstack install tailscale-operator tailscale/tailscale-operator -n tailscale --create-namespace --set oauth.clientId=ID --set oauth.clientSecret=SECRET
sops -d k8s/secret.enc.yaml > k8s/secret.yaml && kubectl --context orbstack apply -k k8s/
```

- 서비스 5개+ 시 ProxyGroup 전환으로 리소스 절감 가능 (현재는 per-service expose 방식)

## Gotchas

- `/v1/responses` API 미지원 — Z.ai/ModelArk 모두 responses 프로토콜 미지원. Aperture에서 `openai chat` 프로토콜만 사용
- `k8s/secret.yaml`은 `.gitignore`에 있어 첫 배포 전 `sops -d`로 생성 필요
- KUBECONFIG가 `~/.kube/config:~/.kube/ecoai.config`로 다중 context 설정 — 기본 context는 `ecoai.dxai.kr`(외부 운영). OrbStack 배포 시 반드시 `kubectl --context orbstack` 명시 (실수로 ecoai에 배포 방지)
- K8s namespace 삭제 시 Tailscale finalizer가 stuck될 수 있음 — `kubectl --context orbstack replace --raw .../finalize`로 강제 제거
- Tailscale Admin에서 이전 오프라인 머신 삭제 후 프록시 StatefulSet + identity secret(`ts-litellm-*-0`) 삭제해야 hostname이 깔끔하게 재등록됨
- Helm release는 namespace 삭제 후에도 잔존 — 반드시 `helm --kube-context orbstack uninstall` 별도 실행
