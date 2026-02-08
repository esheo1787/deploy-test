# 🚀 SI 배포 알림 자동화 파이프라인

> GitHub Actions + n8n Webhook을 활용한 CI/CD 배포 결과 자동 알림 시스템

## 개요

SI 프로젝트에서 개발자가 코드를 push하면, 빌드 성공/실패 여부를 **자동으로 판단**하여 팀에 알림을 보내고 실패 시 에러 로그를 기록하는 파이프라인입니다.

**기존 문제:** 배포 결과를 사람이 직접 확인 → Slack에 수동 알림 → 실패 시 엑셀에 수동 기록  
**해결:** 코드 push만 하면 알림 + 로그가 **End-to-End 자동화**

## 아키텍처

```
개발자 코드 push (GitHub)
       │
       ▼
GitHub Actions (자동 빌드/테스트)
       │
       ▼ Webhook (빌드 결과 JSON 전송)
       │
   n8n 워크플로우
       │
       ├── Branch Check (main 브랜치만 필터)
       │
       ├── Build Status Check
       │       │
       │   성공 ──→ ✅ 성공 알림 (Slack/Webhook)
       │       │
       │   실패 ──→ 🚨 실패 알림 (Slack/Webhook)
       │            └── 📋 Google Sheets 에러 로그 자동 기록
```

## 기술 스택

| 구분 | 기술 |
|------|------|
| CI/CD | GitHub Actions |
| 워크플로우 자동화 | n8n (Webhook 트리거) |
| 알림 | Webhook (Slack 연동 가능) |
| 에러 로그 | Google Sheets (자동 Append) |
| 테스트 도구 | Hoppscotch (API 테스트) |

## 핵심 구현

### 1. GitHub Actions — 빌드 결과를 Webhook으로 자동 전송

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]    # main 브랜치 push 시 자동 실행

jobs:
  build:
    steps:
      - name: Run build/test
        run: |
          echo "빌드 실행 중..."
          # 실제 환경: npm install && npm test
          # 테스트용 실패: exit 1

      - name: Notify Success    # 빌드 성공 시
        if: success()
        run: curl -X POST {n8n_webhook_url} -d '{"build_status": "success"}'

      - name: Notify Failure    # 빌드 실패 시
        if: failure()
        run: curl -X POST {n8n_webhook_url} -d '{"build_status": "failed"}'
```

### 2. n8n 워크플로우 — 조건 분기 + 알림 + 로그

- **Webhook 노드**: GitHub Actions에서 보낸 POST 요청 수신
- **Branch Check (IF)**: `branch === "main"` 필터링
- **Build Status Check (IF)**: `build_status === "success"` 분기
- **성공 알림**: 성공 메시지 전송
- **실패 알림 + Google Sheets**: 실패 알림 전송 + 에러 로그 자동 기록 (병렬 실행)

### 3. Google Sheets — 에러 로그 자동 축적

| timestamp | project | branch | status | error_message |
|-----------|---------|--------|--------|---------------|
| 2026-02-07 15:00 | 대한항공IBE | main | failed | NullPointerException at PaymentService.java:142 |
| 2026-02-08 13:21 | esheo1787/deploy-test | main | failed | Build failed at step: build |

## 실행 결과

### ✅ 성공 케이스
- GitHub Actions: 초록색 체크 ✅
- Webhook 알림: "배포 성공!" 메시지 수신

### ❌ 실패 케이스
- GitHub Actions: 빨간색 ❌ (`exit 1`로 빌드 강제 실패)
- Webhook 알림: "배포 실패! 즉시 확인 필요" 메시지 수신
- Google Sheets: 에러 로그 자동 기록

### n8n 워크플로우
![workflow](https://github.com/user-attachments/assets/3672cf67-976b-43a4-9c23-cb9496489eb5)

### GitHub Actions 실행 결과
![actions](https://github.com/user-attachments/assets/4930d8d1-ae30-4aeb-850c-050c26cc3a7b)

### Google Sheets 에러 로그
![sheets](https://github.com/user-attachments/assets/ef2117fb-7c25-43ab-8b4e-127b1afb8358)

### Webhook 알림 수신
![webhook](https://github.com/user-attachments/assets/ddc9c717-461b-4f22-be81-516270b6039b)

## 개발 과정

1. **Hoppscotch로 Webhook 개념 학습** — 수동으로 JSON을 보내서 n8n 워크플로우 테스트
2. **n8n 워크플로우 설계** — Webhook → 조건 분기 → 알림/로그 파이프라인 구축
3. **GitHub Actions 연동** — 실제 CI/CD 환경에서 자동 트리거되는 End-to-End 파이프라인 완성

## 배운 점

- **Webhook의 본질**: Google Apps Script의 `onEdit` 트리거와 같은 이벤트 기반 구조
- **n8n의 강점**: 코드 없이 시각적으로 워크플로우를 설계할 수 있어 비개발 부서에도 확산 가능
- **확장 가능성**: Docker + Claude Code를 n8n SSH 노드로 연결하면 AI 코드 리뷰 자동화, MCP로 Notion/Swagger 등 외부 시스템 직접 연동 가능
