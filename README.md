# LLM Proxy (LiteLLM)

LiteLLM 기반 LLM 프록시. 여러 프로바이더를 단일 엔드포인트로 통합.

## 모델 목록

### Z.ai (Primary)
- glm-5.1, glm-5-turbo, glm-5, glm-4.7, glm-4.5-air

### ModelArk
- dola-seed-2.0-pro, dola-seed-2.0-lite, dola-seed-2.0-code
- bytedance-seed-code, kimi-k2.5, gpt-oss-120b
- glm-5.1, glm-4.7 (Z.ai fallback)

### Local
- qwen3-vl-4b (eve:58081), qwen3.5-4b (girl:58081)

## 실행

```bash
# .env 파일 확인 후
docker compose up -d
```

## API 사용

```bash
# 모델 목록
curl http://localhost:4000/v1/models -H "Authorization: Bearer sk-litellm-20260516"

# Chat Completions
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-20260516" \
  -H "Content-Type: application/json" \
  -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"hi"}]}'
```
