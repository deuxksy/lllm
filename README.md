# LLM Proxy (LiteLLM)

LiteLLM 기반 LLM 프록시. 여러 프로바이더를 단일 엔드포인트로 통합.

## 프로젝트 구조

```text
lllm/
├── compose.yml              # Docker Compose 설정
├── litellm/
│   └── config.yaml          # LiteLLM 모델 설정
├── k8s/                     # Kubernetes 매니페스트
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── secret.enc.yaml      # SOPS 암호화된 Secret
│   └── kustomization.yaml
├── .env.encrypted           # SOPS 암호화된 환경변수
└── .sops.yaml               # SOPS 설정 (age 키)
```

## 모델 목록

### Z.ai (Primary)
- glm-5.1, glm-5-turbo, glm-5, glm-4.7, glm-4.5-air

### ModelArk
- dola-seed-2.0-pro, dola-seed-2.0-lite, dola-seed-2.0-code
- bytedance-seed-code, kimi-k2.5, gpt-oss-120b
- glm-5.1, glm-4.7 (Z.ai fallback, priority 2)

### Local
- qwen3-vl-4b (eve:58081), qwen3.5-4b (girl:58081)

## 배포

Secret은 SOPS + age로 암호화하여 git에 관리. 평문 `.env`, `k8s/secret.yaml`은 `.gitignore`에 제외.

### Docker Compose

```bash
sops -d --input-type dotenv --output-type dotenv .env.encrypted > .env && docker compose up -d
```

### Kubernetes

```bash
# Secret 복호화 후 전체 적용
sops -d k8s/secret.enc.yaml > k8s/secret.yaml && kubectl apply -k k8s/
```

## API 사용

```bash
# Docker Compose
curl http://localhost:4000/v1/models -H "Authorization: Bearer sk-litellm-20260516"

# Kubernetes (Tailscale)
curl http://axiom:32400/v1/models -H "Authorization: Bearer sk-litellm-20260516"

# Chat Completions
curl http://axiom:32400/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-20260516" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"hi"}]}'
```
