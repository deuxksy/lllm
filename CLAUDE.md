# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

LiteLLM 기반 LLM 프록시. Z.ai, ModelArk, 로컬 모델을 단일 OpenAI 호환 엔드포인트로 통합.
Tailscale K8s Operator로 `lllm.bun-bull.ts.net:4000` 노출.

## Commands

### 배포 (K8s — 운영)
```bash
sops -d k8s/secret.enc.yaml > k8s/secret.yaml && kubectl apply -k k8s/
```

### 배포 (Docker Compose — 로컬)
```bash
sops -d --input-type dotenv --output-type dotenv compose/.env.encrypted > compose/.env && docker compose -f compose/compose.yml up -d
```

### Secret 암호화 (변경 시)
```bash
sops -e --input-type dotenv --output-type dotenv --age age1qw643dna4spaup6sr5ap0jf039ncjd54e8ekvrfy6p6x96ys2y4qn5vcsy compose/.env > compose/.env.encrypted
sops -e --age age1qw643dna4spaup6sr5ap0jf039ncjd54e8ekvrfy6p6p6x96ys2y4qn5vcsy k8s/secret.yaml > k8s/secret.enc.yaml
```

### 확인
```bash
curl http://lllm.bun-bull.ts.net:4000/v1/models -H "Authorization: Bearer sk-litellm-20260516"
```

## Architecture

### 모델 라우팅
- **Z.ai** (priority 1): glm-5.1, glm-5-turbo, glm-5, glm-4.7, glm-4.5-air
- **ModelArk** (priority 2 fallback): glm-5.1, glm-4.7 + 전용 6개 (dola-seed-2.0-pro/lite/code, bytedance-seed-code, kimi-k2.5, gpt-oss-120b)
- **Local**: qwen3-vl-4b (eve), qwen3.5-4b (girl)
- 동일 model_name의 priority로 Z.ai → ModelArk 자동 폴백

### 두 배포 경로
- `compose/`: Docker Compose — 로컬 개발/테스트용. 포트 32400→4000
- `k8s/`: Kubernetes (OrbStack) — 운영. Tailscale Operator로 노출

### Secret 관리
- SOPS + age로 암호화. age 키는 `~/.config/sops/age/keys.txt`
- 평문 `.env`, `k8s/secret.yaml`은 `.gitignore` 제외
- `.sops.yaml`에 파일별 creation_rules 정의

### K8s 리소스
- Service에 `tailscale.com/expose: "true"`, `tailscale.com/hostname: "lllm"` annotation으로 Tailscale 노출
- 헬스체크는 TCP 프로브 사용 (LiteLLM `/health`가 master key 인증 필요)
- 메모리 limit 1Gi (512Mi에서 OOMKilled 이력)

### Tailscale ACL 요구사항
- `tag:k8s-operator`가 `tag:k8s`의 owner여야 함 (`tagOwners`에 `"tag:k8s": ["tag:k8s-operator"]`)
- OAuth client는 Trust credentials 페이지에서 `Devices Core`, `Auth Keys`, `Services` Write + `tag:k8s-operator`로 생성
- ACL 정책은 `~/git/sylph/policy.hujson`에서 관리 (CI/CD 자동 동기화)
