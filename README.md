# LLM Proxy (LiteLLM)

LiteLLM 기반 LLM 프록시. 여러 프로바이더를 단일 엔드포인트로 통합.

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
sops -d --input-type dotenv --output-type dotenv compose/.env.encrypted > compose/.env && docker compose -f compose/compose.yml up -d
```

### Kubernetes

```bash
sops -d k8s/secret.enc.yaml > k8s/secret.yaml && kubectl apply -k k8s/
```

## API 사용

Tailscale 네트워크에서 `lllm.bun-bull.ts.net:4000`으로 접속.

```bash
# 모델 목록
curl http://lllm.bun-bull.ts.net:4000/v1/models -H "Authorization: Bearer sk-litellm-20260516"

# Chat Completions
curl http://lllm.bun-bull.ts.net:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-20260516" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"hi"}]}'
```

## Aperture 설정

```
Base URL: http://lllm.bun-bull.ts.net:4000/v1
API Key: sk-litellm-20260516
Protocol: openai chat
```
