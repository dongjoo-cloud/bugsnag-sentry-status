# Bug QA Monitoring — Sentry 이전 리뷰 (2026-09-28)

- **작성:** Slack 비서 (Dongjoo 요청)
- **범위:** Linear 프로젝트 [Bug QA Monitoring](https://linear.app/storika/project/bug-qa-monitoring-9c0f05dd1415) 티켓 10개 + 블로커 STORIKA-12108 + Sentry 공식 문서 기준 best practice
- **현황판:** https://dongjoo-cloud.github.io/bugsnag-sentry-status/
- **목적:** 궁극 목표 정의 + 현재 티켓 순서/내용 개선안 (second pair of eyes 요청용)

---

## 1. 궁극의 목표 (North star)

한 줄: **에러 감시의 정본을 Sentry 하나로 두고, Bugsnag은 증거 남기고 깨끗이 끈다.** 그 상태에서 “앱이 깨지면 누가 어디서 어떻게 아는지”가 예전보다 같거나 나아져야 한다.

끝모습:

1. **서비스별 Sentry 프로젝트 분리** — `storika-api` / `storika-web` / `storika-python` (필요하면 crawler 분리). 언어·배포가 다른데 한 통에 넣으면 crawler 소음이 API 이슈를 덮는다.
2. **환경·릴리스가 항상 찍힘** — `production` / `staging` / `development` 이름 통일, 배포마다 `서비스@커밋` 같은 release. JS는 CI에서 소스맵/Debug ID 업로드.
3. **PII 바닥선** — SDK `beforeSend` + org scrubbing. DPA·9507은 이미 했으니 코드/설정으로 고정.
4. **알림은 고신호만** — 새 high / 재발 → Slack. 마이그레이션 게이트만 `#product`. 소유권은 monorepo path 기준.
5. **스파이크·쿼터 방어** — Spike Protection on, 필요하면 필터·DSN 제한. 에러는 샘플 1.0 유지(병행 비교용), 트레이스는 나중에·낮게.
6. **Bugsnag 완전 퇴장** — SDK·시크릿·계정 취소 증빙 + “Sentry가 정본” 메모리/런북.

지금 프로젝트 방향(증거 먼저 → 병행 → 제거 → 해지 → 검수)은 이 목표와 **대체로 맞다**. 손볼 곳은 “끝모습 정의”, “게이트 규칙 충돌”, “빠진 티켓”이다.

---

## 2. 프로젝트·티켓 스냅샷 (2026-09-28 기준)

| 티켓 | 한 줄 | Linear | 현황판 | due |
|------|--------|--------|--------|------|
| [10440](https://linear.app/storika/issue/STORIKA-10440) | Bugsnag 은퇴 증거 백업 | Done | Done | 9/25 |
| [9507](https://linear.app/storika/issue/STORIKA-9507) | 외부 전송 정책 + DPA | Done | Done | 9/26 |
| [10442](https://linear.app/storika/issue/STORIKA-10442) | Sentry ops 베이스라인 | Done | Done | 9/29 |
| [10446](https://linear.app/storika/issue/STORIKA-10446) | API Sentry 동등성 증명 | Done | Done | 10/3 |
| [10443](https://linear.app/storika/issue/STORIKA-10443) | Python → Sentry, Bugsnag pin 제거 | Todo | Brice 대기 | 9/30 (late) |
| [10444](https://linear.app/storika/issue/STORIKA-10444) | web에 Sentry 병행 + 소스맵 | Todo | 작업 중 | 10/1 (late) |
| [10447](https://linear.app/storika/issue/STORIKA-10447) | API에서 Bugsnag 제거 | Todo | 작업 중 | 10/10 |
| [10445](https://linear.app/storika/issue/STORIKA-10445) | web에서 Bugsnag 제거 | Todo | 대기 | 10/8 |
| [9506](https://linear.app/storika/issue/STORIKA-9506) | 시크릿 삭제·계정 해지 | Todo | Brice 실행 | 10/13 |
| [10441](https://linear.app/storika/issue/STORIKA-10441) | Sentry 단일 모니터 최종 검수 | Todo | 대기 | 10/14 |

**프로젝트 밖 블로커:** [STORIKA-12108](https://linear.app/storika/issue/STORIKA-12108) — crawler GHA 배포 (assignee Brice, due 10/2). 10443을 막음.

**선행 작업:** [STORIKA-9392](https://linear.app/storika/issue/STORIKA-9392) (Done) — API에 Sentry SDK 이중 전송 기반.

---

## 3. 현재 처리 순서

### Linear / 현황판이 암시하는 흐름

```
9507 ──┬──► 10442 ──► 10443 ──┬──► 10444 ──► [7d/증거] ──► 10445 ──┐
       │                       │                                      │
10440 ─┼───────────────────────┼──────────────────────────────────────┤
       │                       │                                      ▼
       │      10446 ──► [7d] ──► 10447 ─────────────────────────────► 9506 ──► 10441
       │                       ▲
       │                    12108 (crawler deploy)
```

현황판 한 줄: `10440 → 9507 → 10442 → 10446 → 10443 → 10444 → 7일 병행 → 10447 → 10445 → 9506 → 10441`

### 잘한 점
- 정책·DPA → ops 베이스 → 런타임 붙이기 → 제거 → 해지 → 검수 큰 흐름 OK.
- API는 9392로 이미 dual → **10446 동등성 먼저** 후 10447 제거는 best practice와 맞음.
- 10440 인벤토리를 해지 전에 둔 것도 맞음.

### 어긋나거나 위험한 점
1. **「7일 병행」 vs 「증거만」** — 티켓 Decision은 “경과 시간이 아니라 증거”, Brice·현황판은 “최소 7일”. 10445/10447 머지 타이밍을 흔든다. **하나로 못 박아야** 한다.
2. **10443이 10444를 block** — 현황판은 나란히. Python 때문에 web 추가를 굳이 막을 필요는 거의 없다. **병렬**이 맞다.
3. **Nest `HttpException` 함정** — 기본 필터가 HTTP 예외를 안 잡을 수 있음. 10446이 의도적 503 미실행 + “예상 밖 실패 25경로가 4xx로 나가며 Sentry 없음”을 기록. 병행을 ‘공평’이라 부르기 전에 갭을 닫거나 의도적 제외로 문서화해야 한다.
4. **알림·릴리스·맵** — best practice는 “알림 먼저 / 맵·릴리스 day one”. 10442 Done이지만 Team 플랜이라 키당 60/min 불가, GitHub 연동 없어 suspect-commit 불가. 후속 없이 defer만 됨.
5. **소유권 표기** — 10장 assignee DJ인데 해지·시크릿·Vercel·12108은 Brice. Linear에 실행자/게이트 오너를 명시하지 않으면 헷갈린다.

### 현재 병목 (현황판 2026-09-28 21:05 KST)
1. Brice: Vercel Sentry **development** env (10444)
2. Brice: 10443 **prod 승인** (brand-extraction: dev 머지 ≈ prod)
3. Brice: **12108** crawler 배포 경로
4. 7일 dual-run vs 증거-only 정책 충돌
5. Bugsnag 다음 결제 **2026-10-05 ($23)**

---

## 4. Sentry best practice (우리 상황에 맞게)

출처는 주로 Sentry 공식 문서. 상세 메모: 박스 `/workspace/research/sentry-storika-best-practices.md`

| 주제 | 요지 | Storika에 맞는 이유 |
|------|------|---------------------|
| 프로젝트 분리 | 서비스/언어별로 나눔. env ≠ project | API / web / Python 스택이 다름. crawler 소음 분리 |
| env·release·맵 | 이름 통일, release day one, JS 맵 CI | Bugsnag 병행 비교·스택 가독성 |
| Dual-write | ≥1–2 릴리스 사이클 병행, **알림 먼저**, 조용한 주만으로 증명 금지 | 이미 ≥7일 + #product 게이트 방향과 맞음 |
| PII | SDK scrub + org Advanced Scrubbing | DPA/9507 이후 코드로 고정 |
| 알림 | Issue alert = 새/재발. 고신호만 Slack | 작은 팀 노이즈 방지 |
| Spike/쿼터 | SP → quota → DSN limit → 필터 → SDK sample | Team 플랜 limit 한계 → SDK/`beforeSend`로 보완 |
| Nest 함정 | `HttpException` 기본 미포착 가능 | 10446 갭과 직결 |
| 컷오버 함정 | dual init 순서, 맵 누락, 알림 나중, 에러 샘플링, SP가 버스티 Python 삼킴 | 10443/10444/10447 AC에 반영할 것 |

---

## 5. 티켓 수정·개선 제안

### A. 기존 티켓

| 티켓 | 제안 |
|------|------|
| 프로젝트 / 10445·10447·9506 | Decision 한 줄로 통일. 예: 「병행 ≥7일 **그리고** 핵심 경로 합성/자연 에러 증거 + #product 승인. 둘 다 만족해야 cut.」 |
| 10443 ↔ 10444 | Linear에서 10443 blocks 10444 **해제** |
| 10443 | AC: Brice prod 승인 + 12108 + brand-extraction prod 리스크. Linear Todo→In Progress |
| 10444 | AC에 Vercel **development** 커스텀 env `SENTRY_*` 명시. creator-web은 out-of-scope + 후속 후보 한 줄 |
| 10446 후속 | #10225의 Sentry 미포착 25경로 분류(의도 vs 버그). Nest 필터 재검증을 cut 전 필수에 |
| 10447 | “가장 빠른 머지 10/5”와 Bugsnag 결제 10/5 겹침 → 게이트 미충족 시 한 달 더 내는 선택을 #product에 |
| 10442 잔여 | 후속 티켓: SDK 볼륨 캡 / (원하면) Business·키당 limit / GitHub↔Sentry |
| 9506 | DJ 준비 / **실행 Brice(Master)** description에 명시 |
| 10441 | prod에 API cut 반영 전엔 api-dev만으로 Done 금지 |

### B. 새로 두면 좋은 티켓
1. **릴리스·소스맵 CI 계약** — Cloud Run / Vercel / Python
2. **Python Spike Protection 결정** — crawler/cron 버스트 시 SP 정책
3. **컷오버 후 온콜/런북** — #api-error / #web-error 소유·에스컬레이션
4. **creator-web / 휴면 agents·pipeline 키** — 백로그 한 줄
5. **해지 후 비용 확인** — 10/5 $23 미청구 체크

### C. 추천 실행 순서 (현실 버전)

```
이미 끝: 10440, 9507, 10442, 10446

지금 병렬:
  · 10443 — Brice(prod 승인 + 12108)
  · 10444 — Brice(Vercel development env) 후 PR/머지
  · (선택) Nest 미포착 경로 분류

병행 윈도우 (게이트 문서화 후):
  · API dual → 10447 (증거+7일+결제 판단)
  · web 10444 머지 후 7일 → 10445

그다음: 9506 (Brice 실행) → 10441
```

---

## 6. Second pair of eyes에 묻고 싶은 것

리뷰어에게 특히 봐 달라고 요청하는 포인트:

1. 「7일 + 증거」 게이트 문구가 과도한가 / 부족한가?
2. 10443 → 10444 block 해제가 맞는지 (의존성 놓친 게 없는지)
3. Nest 미포착 25경로를 **새 티켓**으로 뺄지, **10447 AC**에 넣을지
4. Python을 프로젝트 하나(`storika-python`)로 둘지, crawler 분리할지
5. Bugsnag 10/5 결제 전에 cut을 무리하게 당길 가치가 있는지
6. 빠진 티켓(릴리스 CI, SP, 온콜) 우선순위

---

## 7. 한 줄 요약

목표는 **“Sentry 3프로젝트 + 릴리스/맵/스크럽/고신호 알림이 정본, Bugsnag은 증거 후 소멸”** 이고, 지금 티켓 뼈대는 그 길을 잘 간다. 고칠 핵심은 **게이트 규칙 통일, 10443↔10444 직렬 끊기, Nest 포착 갭·CI 릴리스/맵·Python SP·온콜을 티켓으로 명시, Brice 실행 항목을 Linear에 드러내기**.

---

## 참고 링크
- 프로젝트: https://linear.app/storika/project/bug-qa-monitoring-9c0f05dd1415
- Slack 인계 스레드: https://storika-ai.slack.com/archives/C07EB8FM44D/p1790177773055409
- 현황판: https://dongjoo-cloud.github.io/bugsnag-sentry-status/
- Sentry org getting started: https://docs.sentry.io/organization/getting-started/
- NestJS: https://docs.sentry.io/platforms/javascript/guides/nestjs/
- Next.js manual: https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/
- Spike protection: https://docs.sentry.io/pricing/quotas/spike-protection/

---

## 8. Second pair of eyes — Researcher (2026-09-28)

### 총평
North star·큰 흐름·API dual 후 10447 순서 **동의**. 고칠 핵심 네 줄도 맞음.

### 추천별
| 항목 | 판정 |
|------|------|
| 게이트 ≥7일 AND 증거 AND #product | **Agree** — OR로 완화 금지. 조용한 7일만으로 증명 금지 |
| 10443 blocks 10444 해제 | **Agree** — 병렬 OK. 공유 쿼터/SP는 병행 기간 모니터링(AC 한 줄) |
| 10443 AC (Brice+12108+prod 리스크) | **Agree** |
| 10444 Vercel development + creator-web OOS | **Agree** |
| Nest 25경로 cut 전 | **Agree** — 배치는 아래 §6-3 |
| 10447 vs 10/5 #product | **Agree** |
| 10442 잔여 (볼륨/GitHub) | **Agree, P2** — cut 블로커 아님 |
| 9506 실행 Brice | **Agree** |
| 10441 api-dev만 Done 금지 | **Agree** (강하게) |

### 새 티켓
- 릴리스·소스맵 CI — **Agree, 우선 높음** (특히 web)
- Python SP — **Agree**
- 온콜/런북 — **Agree**, Bugsnag 끄기 **전**
- creator-web/휴면 키 — 독립 티켓 대신 10444/10441 백로그 한 줄
- $23 미청구 — **standalone cut** → **9506 AC**에 「10/5 이후 미청구 확인」

### 추가
- Decision에 **증거 체크리스트** (서비스별: 합성 1 + 자연 high/med ≥1, 알림→Slack)
- dual 기간 **쿼터/SP 모니터링** 담당
- 컷오버 직전 **알림 라우팅 리허설** (Sentry→Slack only)

### §6 답
1. **게이트** — 적정. 증거를 체크리스트로 고정. 트래픽 적은 web은 7일 + **최소 이벤트 수** 병기 가능. 7일 OR 증거로 바꾸지 말 것.
2. **10443→10444** — 해제 맞음. 공유 자원은 쿼터/SP뿐.
3. **Nest 25** — **새 티켓** + **10447은 그 티켓의 ‘의도적 제외 문서화’에 block**. 분류≠제거 스킬; 진짜 수정은 제외 목록 #product 합의 후 cut 뒤로 OK.
4. **Python 프로젝트** — 지금은 **`storika-python` 하나** + 태그. 볼륨 보이면 분리. SP 티켓에 탈출구 문구.
5. **10/5 cut** — **거의 가치 없음.** $23 ≪ 잘못된 cut. 한 달 더 내고 증거. 결제 날짜로 AC 완화 금지.
6. **우선순위** — P0 Nest triage → P0 릴리스/맵 CI → P1 온콜 → P1 Python SP → P2 나머지.

### Researcher 한 줄
뼈대 유지. **게이트 7일∧증거∧#product**, **10443∥10444**, Nest는 **별도 티켓으로 10447 앞**, **$23 때문에 cut 당기지 말 것**, 새 티켓은 릴리스/맵·SP·온콜만 무게.
