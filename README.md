# soloflow

> 한국 프리랜서·1인 사업자용 단일 운영 앱 — 대금/계약/일정/집중을 한 곳에서

## 목표

프리랜서가 가장 아픈 3축(대금·계약·일정)을 흩어진 도구 없이 빠르게 처리. SaaS 의존 없이 자체호스팅·zero-dep으로 통제 가능한 자기 운영 시스템.

## 대상

- 20대 후반~40대 한국 프리랜서/1인 사업자/부업러/크리에이터
- 본인 PC·휴대폰에서 자체호스팅 또는 단일 사용자 인스턴스 운용
- 클라우드 SaaS 의존 없이 로컬 통제를 원하는 사람

## 아님 (명시적 제외)

- ❌ 팀 협업 도구 (단일 사용자 single-tenant만)
- ❌ 회계 정산 SaaS (세무사 대체 X — 보조 데이터 제공만)
- ❌ 메신저·CRM 통합 허브 (외부 SaaS API 흡수 금지)
- ❌ 모바일 native 앱 v1 (순수 SSR 웹)
- ❌ 다국가 세무/통화 자동 변환 v1 (KRW 단일)

## 기술 스택

| 계층 | 선택 |
|------|------|
| **Language** | FreeLang v11 (.fl) — 코드 순수(다른 언어 0줄), 외부 npm 0개 |
| **Runtime** | Node.js 위의 freelang-v11 인터프리터 (bootstrap.js로 구동) |
| **Storage** | JSONL (append-only event log) — `data/*.jsonl` |
| **Render** | 순수 SSR (server-html/form-field/form-num) — JS 0줄 |
| **Server** | Node.js HTTP (port 41955) |
| **Database** | 없음 (JSONL event sourcing) |

## 모듈 구조

### M1: 송장 (Invoice)
- 송장 등록/조회/입금 기록
- 미수금 현황/월별 수입/aging
- 8개 라우트

### M2: 계약 (Contract)
- 계약서 작성/서명/만료 관리
- 템플릿 지원
- 7개 라우트 (v1.2-P0: cancel 추가)

### M3: 일정 (Schedule)
- 프로젝트 타임라인/마일스톤
- Daily focus 연동
- 8개 라우트

### M4: 집중 (Focus)
- Pomodoro·집중 시간 추적
- 습관 체크
- 11개 라우트

### v1.1-A: 발송 (Delivery)
- 채널별 발송 audit trail (이메일/SMS/카카오톡/수동)
- 5개 라우트

### v1.1-B: 템플릿 (Template)
- 계약서 템플릿 CRUD
- 버전 관리
- 5개 라우트

### v1.1-C: 연결 (Contract-Invoice Link)
- 계약 ↔ 송장 매핑 (1:N)
- Invariant: cancelled 계약은 송장 연결 금지
- 2개 라우트

## 설치 및 실행

### 필수 환경
```bash
Node.js (v18+)
/root/kim/freelang-v11/bootstrap.js (FreeLang runtime)
```

### 로컬 실행
```bash
node /root/kim/freelang-v11/bootstrap.js run src/main.fl
```

포트: `http://localhost:41955`

### 테스트
```bash
# M1 송장 테스트
SOLOFLOW_EVENTS_PATH=tests/data/projection-test.jsonl \
  node /root/kim/freelang-v11/bootstrap.js run tests/projection-test.fl

# M2 계약 테스트
SOLOFLOW_CONTRACT_EVENTS_PATH=tests/data/contract-test.jsonl \
  node /root/kim/freelang-v11/bootstrap.js run tests/contract-test.fl

# v1.2-P0 계약 취소
SOLOFLOW_CONTRACT_EVENTS_PATH=tests/data/contract-cancel-test.jsonl \
  node /root/kim/freelang-v11/bootstrap.js run tests/contract-cancel-test.fl
```

## 개발 상태

### ✅ 완료
- **v1.0** (M1~M4): 모든 기능 구현 + 테스트
- **v1.1-A** (Delivery): 채널별 발송 audit trail
- **v1.1-B** (Template): 템플릿 CRUD + 버전 관리
- **v1.1-C** (Contract-Invoice Link): 계약-송장 양방향 링크
- **v1.2-P0** (Contract Cancel): 계약 취소 기능 + I1 invariant 활성화
- **v1.2-P1A** (State one-pass): Invoice 읽기 최적화 (365 reads → 1 read)
- **v1.2-P1B** (Contract one-pass): Contract 읽기 최적화 (2 reads → 1 read)

### 🔄 진행 중
- **v1.2-P1C**: Schedule 읽기 최적화 (5 reads → 1 read)
- **v1.2-P1D**: Focus 읽기 최적화 (3+H reads → 1 read)

### 📋 백로그 (이후 버전)
- v1.2-P2: Linked invoice 역방향 표시 (UX 완결)
- v2: 모바일 native 앱
- v2: 다중 통화·환율 지원
- v2: 이메일/SMS 자동 발송

## 아키텍처 원칙

### dec-002: Event Sourcing
- 모든 상태 변화 = 이벤트 기록
- Row update 금지 (append-only JSONL)
- Projection = 이벤트 재생

### dec-007: Zero-Dependency
- 외부 npm 0개 (표준 Node.js만)
- 캐시 레이어 없음 (메모리 임시 그룹화만)
- Snapshot 파일 없음
- Index 파일 없음

### dec-008: Bounded Context
- 각 모듈 = 독립 이벤트 파일
  - `data/invoice-events.jsonl` (M1)
  - `data/contract-events.jsonl` (M2)
  - `data/schedule-events.jsonl` (M3)
  - `data/focus-events.jsonl` (M4)
  - `data/delivery-events.jsonl` (v1.1-A)
  - `data/template-events.jsonl` (v1.1-B)
  - `data/contract-invoice-link-events.jsonl` (v1.1-C)

## Invariant (도메인 규칙)

| ID | 규칙 | 위반 시 |
|----|----|--------|
| I1 | Cancelled contract는 송장과 연결 금지 | HTTP 400 |
| I2 | 이미 연결된 계약-송장은 재연결 금지 | HTTP 409 |
| I3 | 이미 취소된 계약 재취소 금지 | HTTP 400 |
| I4 | Invoice는 최대 1개 contract과만 연결 | HTTP 409 |
| I5 | Idempotent unlink (재unlink → 200 OK) | (no-op) |
| I6 | Delivery pending 중복 발송 금지 | HTTP 409 |

## 성능 최적화 (v1.2)

### One-Pass Scan Pattern
불필요한 read 반복을 제거하여 단일 요청 내 file-read 횟수를 감소.

```lisp
; Before (v1.1)
(let [invs (list-invoices)]           ; read-events 1회 + N×project 각 1회
      unpaid (unpaid-invoices)]       ; read-events 1회 (중복) + N×project 각 1회
  ...)

; After (v1.2)
(let [evs (read-events)]              ; read-events 1회만!
      invs (list-invoices-from-events evs)
      unpaid (unpaid-invoices-from-invoices invs)]
  ...)
```

### Read Count 감소
- `GET /` (대시보드): **365 → 1 read** ✅ (v1.2-P1A)
- `GET /contracts/:id`: **2 → 1 read** ✅ (v1.2-P1B)
- `GET /schedule/project/:id`: **5 → 1 read** (v1.2-P1C 진행 중)
- `GET /focus`: **3+H → 1 read** (v1.2-P1D 진행 중)

## 파일 구조

```
soloflow/
├── src/
│   ├── main.fl                           # 라우트 등록 + 서버 시작 (46개 route)
│   ├── render.fl                         # HTML 헬퍼 (money, fmt-date, esc 등)
│   ├── state.fl                          # Invoice projection (v1.2-P1A 최적화)
│   ├── invoice.fl                        # Invoice 라우트 핸들러 (8개)
│   ├── contract-state.fl                 # Contract projection (v1.2-P1B 최적화)
│   ├── contract-invoice-link-state.fl    # Link projection (bounded-context)
│   ├── contract-invoice-link.fl          # Link 라우트 (2개: link/unlink)
│   ├── contract.fl                       # Contract 라우트 (7개, v1.2-P0 cancel)
│   ├── contract-template.fl              # Template 활용 (select-contract-template)
│   ├── template-state.fl                 # Template projection (seed 3개)
│   ├── template.fl                       # Template 라우트 (5개)
│   ├── schedule-state.fl                 # Schedule projection (5개 함수)
│   ├── schedule.fl                       # Schedule 라우트 (8개)
│   ├── focus-state.fl                    # Focus projection (session/habit)
│   ├── focus.fl                          # Focus 라우트 (11개: pomodoro + habit)
│   ├── delivery-state.fl                 # Delivery projection (pending/sent/failed)
│   ├── delivery.fl                       # Delivery 라우트 (5개)
│   └── cron.fl                           # Cron job (만료 알림 등)
├── tests/
│   ├── projection-test.fl                # M1 송장 검증 (26/26 PASS)
│   ├── contract-test.fl                  # M2 계약 검증 (30/30 PASS)
│   ├── contract-cancel-test.fl           # v1.2-P0 검증 (11/11 PASS)
│   ├── contract-invoice-link-test.fl     # v1.1-C 검증 (29/29 PASS)
│   ├── template-test.fl                  # v1.1-B 검증 (37/37 PASS)
│   ├── focus-test.fl                     # M4 집중 검증 (21/21 PASS)
│   ├── delivery-test.fl                  # v1.1-A 검증 (26/26 PASS)
│   └── read-count-test.fl                # v1.2-P1A 계측 검증 (14/14 PASS)
├── data/
│   ├── invoice-events.jsonl              # 송장 이벤트 (append-only)
│   ├── contract-events.jsonl             # 계약 이벤트
│   ├── schedule-events.jsonl             # 일정 이벤트
│   ├── focus-events.jsonl                # 집중 이벤트
│   ├── delivery-events.jsonl             # 발송 이벤트
│   ├── template-events.jsonl             # 템플릿 이벤트
│   └── contract-invoice-link-events.jsonl # 링크 이벤트
├── SPEC.airc                             # 프로젝트 정의서 (hot=v1.1-C)
└── README.md                             # 이 파일
```

## 테스트 커버리지

| 테스트 | 대상 | 결과 | 마지막 업데이트 |
|--------|------|------|-----------------|
| projection-test.fl | M1 + aging bucket + compute-status | 26/26 ✅ | 2026-05-24 |
| contract-test.fl | M2 all contracts + expiring-soon | 30/30 ✅ | 2026-05-24 |
| contract-cancel-test.fl | v1.2-P0 cancel + I1/I3 | 11/11 ✅ | 2026-05-24 |
| contract-invoice-link-test.fl | v1.1-C link/unlink + I2/I4/I5 | 29/29 ✅ | 2026-05-24 |
| template-test.fl | v1.1-B CRUD + version | 37/37 ✅ | 2026-05-24 |
| focus-test.fl | M4 session + habit | 21/21 ✅ | 2026-05-24 |
| delivery-test.fl | v1.1-A status machine | 26/26 ✅ | 2026-05-24 |
| read-count-test.fl | v1.2-P1A one-pass semantics | 14/14 ✅ | 2026-05-24 |
| **합계** | **8개 테스트** | **194/194** ✅ | **2026-05-24** |

## 자주 묻는 질문

### Q: 다중 사용자 지원?
**A:** 아니오. Single-tenant only. 다중 사용자는 v2+ 검토 대상.

### Q: 클라우드 배포?
**A:** 지원하지 않음. 자체호스팅(온프레미스) 또는 로컬 운용만.

### Q: 모바일 앱?
**A:** v1은 웹(순수 SSR)만. Native 앱은 v2+ 백로그.

### Q: API 문서?
**A:** 각 라우트는 `src/*.fl`의 주석 참고. OpenAPI 문서 미제공.

### Q: 데이터 백업?
**A:** JSONL 파일을 직접 백업 (git, tar, rsync 등 표준 도구 사용).

### Q: 외부 SaaS 연동?
**A:** v1은 연동 금지 (zero-dep). v2에서 선택적 지원 검토.

---

**Last Updated**: 2026-05-24  
**Status**: v1.2-P1B (one-pass scan 진행 중)  
**Total Tests**: 194/194 PASS  
**Repository**: gogs.dclub.kr/kim/soloflow (private)  
**License**: Internal Project (수정/재배포 자유, 상업화 협의)
