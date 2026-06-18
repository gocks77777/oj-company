# AX 앱 MVP 완성 계획

> 목표: 기업상태표 폼 직접 입력이 메인 플로우인 데모 가능한 상태
> 현재 상태: ~/ax-consulting/ — 음성 업로드가 Step 1, 폼은 Step 2 (검토용)
> 필요 변경: 작음 — 폼 직접 입력을 Step 1으로 승격

관련 케이스: [case-doc-automation.md](case-doc-automation.md)

---

## 현재 아키텍처 (as-is)

```
Step 1: 녹음 파일 업로드 (필수)
    ↓ Whisper STT
Step 2: 기업상태표 검토/수정 (폼 UI 이미 존재)
    ↓ Claude API
Step 3: 사업계획서 + PPT 다운로드
```

**문제점:**
- 녹음 파일 없이는 진행 불가 → 데모/클라이언트 온보딩 장벽
- Claude CLI (`subprocess`) 사용 → 느리고 불안정
- fill_form.py가 Claude CLI에 의존

---

## 목표 아키텍처 (to-be)

```
[탭 선택]
  ├─ 직접 입력 (기본): 폼 바로 작성 → Step 2 문서 생성  ← 신규 메인 플로우
  └─ 녹음 업로드 (선택): 기존 플로우 유지
```

---

## 변경 사항 (최소 범위)

### 1. `static/index.html` — Step 바 + 탭 UI 수정
- Step 1 레이블: "녹음 업로드" → "기업상태표 입력"
- 상단에 탭 2개 추가: `[직접 입력]` / `[녹음 업로드]`
- "직접 입력" 탭 선택 시 → 즉시 기업상태표 폼 표시
- "녹음 업로드" 탭 선택 시 → 기존 음성 업로드 플로우

### 2. `main.py` — 직접 입력 엔드포인트 추가
```python
@app.post("/submit-form-direct")
async def submit_form_direct(form_data: dict):
    # STT 생략, 폼 데이터를 바로 generate_docs로 전달
    job_id = str(uuid.uuid4())
    asyncio.create_task(process_form_direct(job_id, form_data))
    return {"job_id": job_id}
```

### 3. `services/fill_form.py` → Claude SDK로 교체 (선택, 안정성)
```python
# 현재: subprocess.run(["claude", "-p", prompt])
# 개선: anthropic.Anthropic().messages.create(...)
```
> **Post 3 발행 전 권장** — subprocess는 PATH/세션 이슈로 데모 중 자주 깨짐. SDK 교체 시 안정성 대폭 향상.

---

## 데모 안정성 게이트 (Post 3 발행 전 필수)

Post 3은 "10분 안에 완성"이 핵심 후크. 아래 게이트 통과 전 발행 금지:

- [ ] 폼 입력 → PPT 다운로드 **3회 연속 성공** (오류 없이)
- [ ] 실제 소요 시간 측정 → **10분 미만** 확인 후 기록
- [ ] ngrok URL 24시간 라이브 유지 확인 (Post 3 본 사람이 즉시 검증 가능)

## 데모 완성 체크리스트

- [ ] 탭 UI 추가 (직접 입력 / 녹음 업로드)
- [ ] `/submit-form-direct` 엔드포인트 추가
- [ ] `fill_form.py` → Anthropic SDK로 교체 (subprocess 제거)
- [ ] 직접 입력 → 문서 생성 → 다운로드 E2E 테스트
- [ ] 데모 안정성 게이트 3개 통과
- [ ] ngrok으로 외부 접근 가능 URL 생성 (Railway 영구 배포는 이후)
- [ ] 데모 GIF 녹화 (Loom 또는 QuickTime)
  - 기업상태표 폼 작성 30초
  - "생성" 클릭
  - 사업계획서 미리보기 + PPT 다운로드 (10분 내)
- [ ] 콘텐츠 Post 3에 GIF 첨부

---

## 예상 소요 시간

| 작업 | 시간 |
|---|---|
| 탭 UI 추가 (HTML/JS) | 1~2시간 |
| 엔드포인트 추가 (Python) | 1시간 |
| E2E 테스트 | 30분 |
| 데모 GIF 녹화 | 30분 |
| **합계** | **3~4시간** |

> 2인 분업 시: 한 명이 UI/엔드포인트, 한 명이 SDK 교체 + E2E 테스트를 병렬로 진행해 반나절 내 완료 가능.

---

## 배포 옵션

| 옵션 | 장점 | 단점 |
|---|---|---|
| ngrok (임시) | 설정 5분, Claude API 키 로컬 | URL 매번 바뀜 |
| Railway/Render | 영구 URL, 무료 플랜 | env 설정 필요 |
| Vercel (Next.js 마이그레이션) | 소개 사이트와 통일 | 리팩토링 필요 |

> 추천: 데모용 ngrok → 게시물 올릴 때 스크린샷/GIF만 쓰고, 이후 Railway로 영구 배포
