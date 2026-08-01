# MLX Gemma 4 26B QAT Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** MLX 기반 `google/gemma-4-26b-a4b-qat` (`mlx-community/gemma-4-26b-a4b-it-4bit`) 모델을 Mac 호스트에서 구동하고, OrbStack K8s의 LiteLLM (`lllm`) 및 Tailscale Aperture (`sylph`)를 거쳐 Hermes Agent (`247365`)에서 활용하도록 4단계 검증 프로세스를 거쳐 통합 구현한다.

**Architecture:** Mac 호스트에서 `mlx_lm.server --host 0.0.0.0`으로 실행하고, OrbStack K8s의 LiteLLM 프록시가 `http://host.orbstack.internal:8080/v1`으로 접근하며, Tailscale Operator가 `lllm.bun-bull.ts.net:4000` 주소로 Aperture 및 Hermes Agent에 OpenAI `chat_completions` 프로토콜 기반 모델을 노출한다.

**Tech Stack:** macOS MLX-LM (`mlx_lm.server`), OrbStack K8s (LiteLLM, Tailscale Operator), Tailscale Aperture (`sylph`), Hermes Agent (`247365`).

## Global Constraints

- MLX 서버는 K8s 파드 통신을 위해 반드시 `--host 0.0.0.0 --port 8080`으로 바인딩한다.
- LiteLLM 퍼블릭 모델 식별자는 `google/gemma-4-26b-a4b-qat`로 고정하고, MLX 체크포인트는 `mlx-community/gemma-4-26b-a4b-it-4bit`를 사용한다.
- Hermes Agent와 Aperture 간 통신 프로토콜은 OpenAI `chat_completions` (`api_mode: chat_completions`, `base_url: http://ai/v1`)를 준수한다.
- 각 단계(Phase 1~4)는 지정된 Verification Gate의 실제 추론(Inference) 테스트를 통과해야 다음 단계로 진입한다.

---

### Task 1: Phase 1 - MLX Local Server Setup & Inference Verification

**Files:**
- Host execution: Mac Host Terminal (`mlx_lm.server`)

**Interfaces:**
- Produces: OpenAI-compatible REST API at `http://localhost:8080/v1/chat/completions` and `http://0.0.0.0:8080/v1`

- [ ] **Step 1: MLX 패키지 및 서버 실행 환경 확인**

```bash
python3 -m pip install -U mlx-lm
```

- [ ] **Step 2: MLX Local Server 0.0.0.0 바인딩 실행**

```bash
mlx_lm.server --model mlx-community/gemma-4-26b-a4b-it-4bit --host 0.0.0.0 --port 8080
```
Expected: `Starting server on 0.0.0.0:8080` 로그 출력 및 모델 로딩 완료

- [ ] **Step 3: GET /v1/models 조회를 통한 엔드포인트 수신 확인**

```bash
curl http://localhost:8080/v1/models
```
Expected: `mlx-community/gemma-4-26b-a4b-it-4bit` 모델이 나열된 JSON 응답

- [ ] **Step 4: POST /v1/chat/completions 실제 추론 테스트 (Verification Gate 1)**

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mlx-community/gemma-4-26b-a4b-it-4bit",
    "messages": [{"role": "user", "content": "Hello! Reply with OK."}]
  }'
```
Expected: HTTP 200 OK 및 `"content": "OK"` 또는 응답 텍스트 포함 JSON 반환

---

### Task 2: Phase 2 - LiteLLM Proxy Integration (lllm Repository & K8s)

**Files:**
- Modify: `/Users/crong/git/lllm/compose/litellm/config.yaml`
- Modify: `/Users/crong/git/lllm/k8s/configmap.yaml`

**Interfaces:**
- Consumes: MLX Server at `http://host.orbstack.internal:8080/v1`
- Produces: LiteLLM endpoint at `http://lllm.bun-bull.ts.net:4000/v1/chat/completions`

- [ ] **Step 1: compose/litellm/config.yaml 에 google/gemma-4-26b-a4b-qat 모델 추가**

`/Users/crong/git/lllm/compose/litellm/config.yaml`의 `model_list` 섹션 끝에 추가:
```yaml
  - model_name: google/gemma-4-26b-a4b-qat
    litellm_params:
      model: openai/mlx-community/gemma-4-26b-a4b-it-4bit
      api_base: http://host.orbstack.internal:8080/v1
      api_key: sk-dummy
```

- [ ] **Step 2: k8s/configmap.yaml 에 동기화 적용**

`/Users/crong/git/lllm/k8s/configmap.yaml`의 `config.yaml` 내 `model_list`에도 동일한 YAML 항목 추가.

- [ ] **Step 3: K8s ConfigMap 적용 및 litellm 파드 재시작**

```bash
kubectl --context orbstack apply -k /Users/crong/git/lllm/k8s/ && kubectl --context orbstack rollout restart deployment litellm -n lllm
```
Expected: `deployment.apps/litellm restarted`, `kubectl --context orbstack rollout status deployment litellm -n lllm` 성공

- [ ] **Step 4: LiteLLM 엔드포인트 모델 목록 및 추론 테스트 (Verification Gate 2)**

```bash
curl http://lllm.bun-bull.ts.net:4000/v1/models -H "Authorization: Bearer sk-litellm-20260516"
```
Expected: JSON 목록에 `google/gemma-4-26b-a4b-qat` 존재 확인.

```bash
curl http://lllm.bun-bull.ts.net:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-20260516" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-26b-a4b-qat",
    "messages": [{"role": "user", "content": "K8s LiteLLM Test"}]
  }'
```
Expected: HTTP 200 OK 및 추론 결과 반환

- [ ] **Step 5: Git 커밋**

```bash
git add compose/litellm/config.yaml k8s/configmap.yaml
git commit -m "feat: add google/gemma-4-26b-a4b-qat model route to LiteLLM"
```

---

### Task 3: Phase 3 - Tailscale Aperture Integration (sylph Repository)

**Files:**
- Modify: `/Users/crong/git/sylph/aperture/config.json`

**Interfaces:**
- Consumes: LiteLLM Proxy at `http://lllm:4000`
- Produces: Tailscale Aperture AI Gateway route for `google/gemma-4-26b-a4b-qat`

- [ ] **Step 1: sylph 레포지토리 aperture/config.json 모델 등록**

`/Users/crong/git/sylph/aperture/config.json`의 `providers` 내 `ZAI-openai` 및 `ModelArk-openai` (또는 지정된 provider) `models` 배열에 `"google/gemma-4-26b-a4b-qat"` 추가.

- [ ] **Step 2: Aperture 구성 변경 검증 (Verification Gate 3)**

Aperture 게이트웨이 호출 테스트:
```bash
curl http://ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-26b-a4b-qat",
    "messages": [{"role": "user", "content": "Aperture Test"}]
  }'
```
Expected: Tailscale Aperture를 경유한 HTTP 200 OK 응답 수신

- [ ] **Step 3: Git 커밋 (sylph 레포지토리)**

```bash
cd /Users/crong/git/sylph && git add aperture/config.json && git commit -m "feat: register google/gemma-4-26b-a4b-qat model in aperture config"
```

---

### Task 4: Phase 4 - Hermes Agent Integration (247365 Repository)

**Files:**
- Modify: `/Users/crong/git/247365/ansible/roles/hermes/templates/config.yaml.j2` (또는 agent 실행 환경 설정 `.env`)

**Interfaces:**
- Consumes: Tailscale Aperture `http://ai/v1` with `chat_completions` protocol
- Produces: Functional Hermes Agent using `google/gemma-4-26b-a4b-qat`

- [ ] **Step 1: Hermes Agent 설정 업데이트**

`/Users/crong/git/247365` 설정 파일에서 모델 및 provider 프로토콜 구성 지정:
```yaml
model:
  default: google/gemma-4-26b-a4b-qat
  provider: custom:aperture-mlx
custom_providers:
  - name: aperture-mlx
    base_url: http://ai/v1
    api_mode: chat_completions
```

- [ ] **Step 2: Hermes Agent 대화 및 툴 실행 검증 (Verification Gate 4)**

Hermes Agent 구동 후 `google/gemma-4-26b-a4b-qat` 모델을 사용한 대화 및 툴 호출(Tool Call) 무결성 확인.
Expected: 멀티턴 대화 및 툴 호출 결과가 정상 처리되고 에이전트 응답 수신 완료

- [ ] **Step 3: Git 커밋 (247365 레포지토리)**

```bash
cd /Users/crong/git/247365 && git add ansible/roles/hermes/templates/config.yaml.j2 && git commit -m "feat: configure Hermes agent to use google/gemma-4-26b-a4b-qat via Aperture"
```
