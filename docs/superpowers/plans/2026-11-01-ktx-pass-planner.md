# KTX 정기권 구매 플래너 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 달력에 휴가·주말 이동일을 찍으면 가장 싼 KTX 정기권·왕복표 조합과 결제일을 알려 주고, 결제일을 구글 캘린더에 넣는 모바일 우선 정적 웹앱.

**Architecture:** 순수 함수 도메인 계층(`src/domain/` — 날짜, 공휴일, 요금, 최적화)과 React UI 계층(`src/ui/`)을 나눈다. 상태는 localStorage 하나에 JSON으로 저장한다. 구글 캘린더는 브라우저에서 Google Identity Services 토큰으로 REST API를 직접 호출한다. 서버는 없다.

**Tech Stack:** Vite 6 + React 18 + TypeScript 5, Vitest + @testing-library/react + jsdom, lucide-react(아이콘), GitHub Pages(GitHub Actions 배포).

**Spec:** `docs/superpowers/specs/2026-11-01-ktx-pass-planner-design.md`

## Global Constraints

- **공개 레포 — 개인정보 금지.** 구간명·지명·실제 운임 기본값·휴가/구매 기록을 코드, 테스트 이름, 커밋 메시지, 문서에 넣지 않는다. 예외: 스펙 §3.2의 검증 숫자(6,400 / 7,500 / 128,000 등)는 "예시 구간" 테스트 픽스처로만 쓴다.
- 커밋 작성자는 레포 로컬 설정(`seong4687` / noreply)을 그대로 쓴다. `git config`를 바꾸지 않는다.
- 운임은 코드에 기본값이 없다. 첫 실행 설정 화면에서 입력받는다. 할인율 기본값만 둔다: 짧은 기간 0.45, 긴 기간 0.50.
- 가격 공식: `1회 = floor(기준운임 × (1 − 할인율) / 100) × 100`, `정기권 = 1회 × 2 × 사용가능일수`.
- 날짜는 전부 `'YYYY-MM-DD'` 문자열(`DateStr`)로 다룬다. 산술은 UTC 기준 `Date`로만 하고, "오늘"은 `Asia/Seoul` 기준으로 구한다.
- 모바일 우선: 기준 폭 360px, 터치 영역 최소 44×44px, 가로 스크롤 없음. UI 문구는 한국어.
- 결과 화면에 계산식을 보여 주지 않는다.
- OAuth: 공개 가능한 클라이언트 ID만 쓴다(빌드 변수 `VITE_GOOGLE_CLIENT_ID`). 시크릿은 쓰지 않는다. 범위는 `https://www.googleapis.com/auth/calendar.events`만 요청한다.
- 결제일 = 시작일 − 1일, 기본 시각 20:00 (Asia/Seoul).

## Review Focus

1. **말일 시작 1개월권** (1/31, 윤년 1/31): 다음 달 말일에 끝나야 한다. 3월로 넘어가면 안 된다. → Task 1 테스트
2. **평일형 정기권 기간 안의 주말 이동일**: 정기권으로 덮이지 않으니 왕복표 값이 붙어야 한다. 0원으로 계산되면 안 된다. → Task 4 테스트
3. **공휴일 직접 추가·삭제**: 그날이 평일 출근일에서 휴일로 바뀌어 가격과 이동 필요 여부가 함께 달라져야 한다. → Task 2·4 테스트
4. **저장소가 비었거나 깨진 JSON**: 앱이 죽지 않고 설정 화면으로 가야 한다. → Task 5 테스트
5. **구글 로그인 거부·토큰 만료·API 실패**: 오류 문구를 띄운다. 일부 일정만 만들어진 경우 성공한 ID는 저장한다. → Task 9 테스트

---

## File Structure

```
src/domain/dates.ts        날짜 문자열 유틸 (순수)
src/domain/holidays.ts     내장 공휴일 + 사용자 오버라이드 판정
src/domain/pricing.ts      상품 종류, 유효 기간, 정기권 가격
src/domain/optimizer.ts    DP 최적화, 전략별 비교안, 달력 범위 자르기
src/domain/planning.ts     계획 시작일·달력 범위·최적화 범위 계산
src/state/storage.ts       AppState 타입, localStorage 로드/저장
src/gcal/gcal.ts           GIS 토큰, 일정 생성/삭제
src/ui/App.tsx             화면 전환(설정/달력/결과/기록)
src/ui/SetupView.tsx       첫 실행·설정 (운임, 할인율, 알림 시각, 현재 정기권 종료일)
src/ui/CalendarView.tsx    달력 + 날짜 탭 + 계산하기
src/ui/ResultView.tsx      추천·비교·가격 수정·샀어요·일정 만들기
src/ui/PurchasesView.tsx   구매 기록 수정/삭제
src/ui/format.ts           금액·날짜 표시, 상품명
tests/...                  각 파일과 같은 이름의 *.test.ts(x)
```

---

### Task 1: 프로젝트 골격 + 날짜 유틸

**Files:**
- Create: `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/main.tsx`, `.gitignore`
- Create: `src/domain/dates.ts`
- Test: `tests/domain/dates.test.ts`

**Interfaces:**
- Produces:
  - `type DateStr = string` ('YYYY-MM-DD')
  - `addDays(d: DateStr, n: number): DateStr`
  - `daysInclusive(start: DateStr, end: DateStr): number`
  - `eachDay(start: DateStr, end: DateStr): DateStr[]`
  - `dayOfWeek(d: DateStr): number` (0=일 … 6=토)
  - `oneMonthEnd(start: DateStr): DateStr` — 시작일 + 1개월 − 1일, 다음 달에 같은 날짜가 없으면 다음 달 말일
  - `lastDayOfNextMonth(d: DateStr): DateStr`
  - `todayInSeoul(now?: Date): DateStr`

- [ ] **Step 1: Vite React-TS 골격 생성**
  - `npm create vite@latest . -- --template react-ts` 후 `npm i lucide-react`, `npm i -D vitest @testing-library/react @testing-library/user-event jsdom`
  - `vite.config.ts`: `base: '/ktx-pass-planner/'`, `test: { environment: 'jsdom' }`
  - `package.json` 스크립트에 `"test": "vitest run"` 추가
  - 템플릿 데모 파일(App.css, 로고 등)은 지운다.
- [ ] **Step 2: 실패하는 테스트 작성** (`tests/domain/dates.test.ts`)

```ts
expect(oneMonthEnd('2026-11-01')).toBe('2026-11-30')
expect(oneMonthEnd('2026-10-01')).toBe('2026-10-31')
expect(oneMonthEnd('2026-11-15')).toBe('2026-12-14')
expect(oneMonthEnd('2027-01-31')).toBe('2027-02-28')
expect(oneMonthEnd('2028-01-31')).toBe('2028-02-29')   // 윤년
expect(oneMonthEnd('2026-12-20')).toBe('2027-01-19')   // 연도 넘김
expect(daysInclusive('2026-11-01', '2026-11-21')).toBe(21)
expect(addDays('2026-12-31', 1)).toBe('2027-01-01')
expect(dayOfWeek('2026-11-01')).toBe(0)                  // 일요일
expect(lastDayOfNextMonth('2026-11-17')).toBe('2026-12-31')
expect(todayInSeoul(new Date('2026-10-31T16:30:00Z'))).toBe('2026-11-01')
expect(eachDay('2026-11-29', '2026-12-01')).toEqual(['2026-11-29', '2026-11-30', '2026-12-01'])
```

- [ ] **Step 3: 실행해 실패 확인** — `npx vitest run tests/domain/dates.test.ts` → FAIL (모듈 없음)
- [ ] **Step 4: `src/domain/dates.ts` 구현** — `Date.UTC`로만 계산한다. `oneMonthEnd`: 다음 달에 같은 날짜가 있으면 그 날짜 − 1일, 없으면 다음 달 말일. `todayInSeoul`: `Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Seoul' })`.
- [ ] **Step 5: 실행해 통과 확인** — 같은 명령 → PASS. `npm run build`도 성공해야 한다.
- [ ] **Step 6: 커밋** — `git add -A && git commit -m "feat: 프로젝트 골격과 날짜 유틸"`

### Task 2: 공휴일

**Files:**
- Create: `src/domain/holidays.ts`
- Test: `tests/domain/holidays.test.ts`

**Interfaces:**
- Consumes: `DateStr`, `dayOfWeek`
- Produces:
  - `type HolidayOverrides = { added: DateStr[]; removed: DateStr[] }`
  - `BUILTIN_HOLIDAYS: ReadonlySet<DateStr>`
  - `HOLIDAY_DATA_LAST_YEAR = 2027`
  - `isHoliday(d: DateStr, o: HolidayOverrides): boolean` — 내장 목록 ∪ added − removed
  - `isWorkday(d: DateStr, o: HolidayOverrides): boolean` — 월~금이면서 휴일이 아님

- [ ] **Step 1: 실패하는 테스트 작성**

```ts
const none = { added: [], removed: [] }
for (const d of ['2026-09-24','2026-09-25','2026-10-05','2026-10-09','2026-12-25'])
  expect(isHoliday(d, none)).toBe(true)
expect(isWorkday('2026-10-05', none)).toBe(false)
expect(isWorkday('2026-11-02', none)).toBe(true)
expect(isWorkday('2026-11-01', none)).toBe(false)                                  // 일요일
expect(isWorkday('2026-11-02', { added: ['2026-11-02'], removed: [] })).toBe(false)
expect(isWorkday('2026-10-05', { added: [], removed: ['2026-10-05'] })).toBe(true)
```

- [ ] **Step 2: 실행해 실패 확인** — `npx vitest run tests/domain/holidays.test.ts` → FAIL
- [ ] **Step 3: `src/domain/holidays.ts` 구현** — 2026·2027 법정공휴일(대체공휴일 포함)을 리터럴 배열로 넣는다.
  - 기준 출처: 우주항공청(한국천문연구원) "월력요항" 2026·2027. 아래 목록은 초안이므로 커밋 전에 월력요항과 한 줄씩 대조하고, 다르면 월력요항을 따른다.
  - **2026 초안**: 01-01, 02-16, 02-17, 02-18, 03-02(삼일절 대체), 05-05, 05-25(부처님오신날 대체), 06-03(지방선거), 08-17(광복절 대체), 09-24, 09-25, 10-05(개천절 대체), 10-09, 12-25. 실제 휴일 당일이 토·일인 날(03-01, 05-24, 06-06, 08-15, 09-26, 10-03)도 목록에 넣어 둔다. 제헌절(07-17) 공휴일 재지정 여부는 월력요항으로 확인한다.
  - **2027**: 월력요항 2027을 기준으로 작성한다.
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 내장 공휴일과 사용자 오버라이드`

### Task 3: 요금 계산

**Files:**
- Create: `src/domain/pricing.ts`
- Test: `tests/domain/pricing.test.ts`

**Interfaces:**
- Consumes: `DateStr`, `daysInclusive`, `eachDay`, `addDays`, `oneMonthEnd`, `isWorkday`, `HolidayOverrides`
- Produces:
  - `type PassKind = 'regular10' | 'regular1m' | 'flexWeekday' | 'flexAll'`
  - `type Fares = { baseFare: number; ticketOneWay: number; rateShort: number; rateLong: number }`
  - `passEnds(kind: PassKind, start: DateStr): DateStr[]` — 가능한 종료일 목록
    - regular10: `[start+9]`
    - regular1m: `[oneMonthEnd(start)]`
    - flex*: start+9 … oneMonthEnd(start)
  - `coversDay(kind: PassKind, d: DateStr, o: HolidayOverrides): boolean` — flexAll은 항상 true, 나머지는 `isWorkday`
  - `passPrice(kind: PassKind, start: DateStr, end: DateStr, fares: Fares, o: HolidayOverrides): number`
  - `passRate(kind: PassKind, start: DateStr, end: DateStr, fares: Fares): number`
    - regular10 → rateShort, regular1m → rateLong
    - flex* → 달력 일수 ≥ 21이면 rateLong, 아니면 rateShort
  - `roundTripTicket(fares: Fares): number` = ticketOneWay × 2

- [ ] **Step 1: 실패하는 테스트 작성** — 스펙 §3.2 다섯 건과 버림·주말 포함을 고정한다.

```ts
const f = { baseFare: 6400, ticketOneWay: 7500, rateShort: 0.45, rateLong: 0.5 }
const o = { added: [], removed: [] }
expect(passPrice('regular1m', '2026-09-01', '2026-09-30', f, o)).toBe(128000)
expect(passPrice('regular1m', '2026-10-01', '2026-10-31', f, o)).toBe(128000)
expect(passPrice('regular1m', '2026-11-01', '2026-11-30', f, o)).toBe(134400)
expect(passPrice('flexWeekday', '2026-11-01', '2026-11-21', f, o)).toBe(96000)
expect(passPrice('regular10', '2026-11-01', '2026-11-10', f, o)).toBe(49000)       // 3,520 → 3,500 버림
expect(passPrice('flexWeekday', '2026-11-02', '2026-11-21', f, o)).toBe(105000)   // 20일 → 45%: 3,500×2×15
expect(passPrice('flexAll', '2026-11-01', '2026-11-21', f, o)).toBe(134400)       // 21일 전체 × 3,200 × 2
expect(passEnds('regular10', '2026-11-01')).toEqual(['2026-11-10'])
expect(passEnds('flexAll', '2026-11-01')).toHaveLength(21)                         // 11-10 … 11-30
expect(roundTripTicket(f)).toBe(15000)
```

- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: `src/domain/pricing.ts` 구현** — 사용가능일수는 `eachDay(start,end).filter(d => coversDay(kind,d,o)).length`.
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 정기권 가격 계산 (100원 버림, 기간자유형 할인 구간)`

### Task 4: 최적화

**Files:**
- Create: `src/domain/optimizer.ts`
- Test: `tests/domain/optimizer.test.ts`

**Interfaces:**
- Consumes: Task 1~3 전부
- Produces:
  - `type DayMark = 'vacation' | 'travel'` — 기본값이 아닌 날만 저장
  - `type PriceOverrides = Record<string, number>` — 키는 `${kind}|${start}|${end}`
  - `overrideKey(kind: PassKind, start: DateStr, end: DateStr): string`
  - `type PassItem = { kind: PassKind; start: DateStr; end: DateStr; price: number; overridden: boolean; payDate: DateStr }`
  - `type PlanResult = { passes: PassItem[]; ticketDates: DateStr[]; ticketCost: number; total: number }`
  - `type OptimizeInput = { planStart: DateStr; calendarEnd: DateStr; horizonEnd: DateStr; marks: Record<DateStr, DayMark>; fares: Fares; holidays: HolidayOverrides; overrides: PriceOverrides; allowed: PassKind[] }`
  - `needsTravel(d: DateStr, marks, holidays): boolean`
    - 평일 출근일: mark ≠ 'vacation'이면 이동
    - 휴일·주말: mark === 'travel'이면 이동
    - `calendarEnd` 이후는 marks를 무시하고 기본 패턴(평일 이동, 휴일 안 감)
  - `optimize(input: OptimizeInput): PlanResult` — 결과는 달력 범위로 자른 값
  - `type StrategyId = 'best' | 'asIs' | 'monthly' | 'tenDay' | 'flexWeekday' | 'flexAll' | 'ticketsOnly'`
  - `compareStrategies(input: Omit<OptimizeInput, 'allowed'>): { id: StrategyId; plan: PlanResult }[]` — 'best' 먼저, 나머지는 스펙 §5.3 순서

- [ ] **Step 1: 실패하는 테스트 작성** — 값은 손으로 검산할 수 있는 시나리오로 정한다.

```ts
const base = { fares: f, holidays: o, overrides: {}, marks: {} }
// 1) 휴가 없음 11/1~11/30: 1개월권 하나가 최선
const r1 = optimize({ ...base, planStart: '2026-11-01', calendarEnd: '2026-11-30', horizonEnd: '2026-11-30', allowed: ALL })
expect(r1.passes).toEqual([expect.objectContaining({ kind: 'regular1m', start: '2026-11-01', end: '2026-11-30', price: 134400, payDate: '2026-10-31' })])
expect(r1.total).toBe(134400)

// 2) 같은 값이면 일반형 우선 (flexWeekday 11/1~11/30도 134,400)
expect(r1.passes[0].kind).toBe('regular1m')

// 3) 평일형 정기권 기간 안의 주말 이동일은 왕복표 (Review Focus 2)
const r3 = optimize({ ...base, marks: { '2026-11-07': 'travel' }, planStart: '2026-11-01', calendarEnd: '2026-11-30', horizonEnd: '2026-11-30', allowed: ['regular1m'] })
expect(r3.ticketDates).toEqual(['2026-11-07'])
expect(r3.total).toBe(134400 + 15000)

// 4) 11/16~11/30 휴가: 출근일은 11/2~11/13 평일 10일뿐
//    regular10 11/2~11/11(평일 8일, 56,000) + 표 11/12·13(30,000) = 86,000
//    flexWeekday 11/2~11/13(12일·45%, 평일 10일 × 7,000) = 70,000  ← 최선
//    flex 21일 이상(50%)은 어떤 21일 창이든 평일 15일 → 96,000
const vac = Object.fromEntries(eachDay('2026-11-16', '2026-11-30').map(d => [d, 'vacation']))
const r4 = optimize({ ...base, marks: vac, planStart: '2026-11-01', calendarEnd: '2026-11-30', horizonEnd: '2026-11-30', allowed: ALL })
expect(r4.total).toBe(70000)
expect(r4.passes).toEqual([expect.objectContaining({ kind: 'flexWeekday', start: '2026-11-02', end: '2026-11-13', price: 70000, payDate: '2026-11-01' })])
expect(r4.ticketDates).toEqual([])

// 5) 가격 오버라이드가 쓰인다
const r5 = optimize({ ...base, overrides: { [overrideKey('regular1m','2026-11-01','2026-11-30')]: 130000 }, planStart: '2026-11-01', calendarEnd: '2026-11-30', horizonEnd: '2026-11-30', allowed: ALL })
expect(r5.passes[0]).toMatchObject({ price: 130000, overridden: true })

// 6) 달력 범위 밖에서 시작하는 정기권은 결과에서 빠진다
const r6 = optimize({ ...base, planStart: '2026-11-01', calendarEnd: '2026-11-20', horizonEnd: '2026-12-31', allowed: ALL })
expect(r6.passes.every(p => p.start <= '2026-11-20')).toBe(true)

// 7) 공휴일 추가가 반영된다 (Review Focus 3)
const r7 = optimize({ ...base, holidays: { added: ['2026-11-02'], removed: [] }, planStart: '2026-11-01', calendarEnd: '2026-11-30', horizonEnd: '2026-11-30', allowed: ['regular1m'] })
expect(r7.total).toBe(128000)                                    // 평일 20일
```

  - 비교 테스트: `compareStrategies`가 7개 id를 위 순서로 돌려준다. `asIs`는 planStart부터 1개월권을 연달아 산 값이다. `ticketsOnly`의 passes는 비어 있다.
- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: `optimize` 구현** — 날짜 인덱스 DP(뒤에서 앞으로).

```
days = eachDay(planStart, horizonEnd); N = days.length
best[N] = 0
for i = N-1 .. 0:
  cand = []
  cand.push( (needsTravel(days[i]) ? roundTrip : 0) + best[i+1] )          // 정기권 없이 하루 넘기기
  for kind in allowed (순서: regular1m, regular10, flexWeekday, flexAll):
    for end in passEnds(kind, days[i]):
      price = overrides[key] ?? passPrice(...)
      j = index(end) + 1 (N보다 크면 N)
      extra = 기간 안 needsTravel인데 coversDay가 false인 날 수 × roundTrip
      cand.push(price + extra + best[j])
  best[i] = cand 중 최소. 비교 키는 (금액, 기간자유형 정기권 개수) 튜플이고, 그래도 같으면 먼저 넣은 후보
```

  - **왜 튜플인가**: 금액만 비교하면 시나리오 1에서 "11/1(일) 넘기기 + 11/2~11/30 기간자유형"(134,400)이 "11/1 시작 1개월권"(134,400)보다 먼저 들어와 이긴다. 그러면 스펙 §3.3 "값이 같으면 일반형"을 어긴다.
  - 기간자유형 개수를 두 번째 키로 두면, 하루를 넘겨서 생기는 동점에서도 일반형이 이긴다.

  - 선택을 따라가며 정기권과 표 날짜를 모은다.
  - 마지막에 `start > calendarEnd`인 정기권과 `> calendarEnd`인 표를 버리고 total을 다시 합한다.
  - 가정 구간까지 이어지는 정기권은 전체 가격으로 남는다.
- [ ] **Step 4: `compareStrategies` 구현**
  - `allowed` 조합: best=전부, monthly=[regular1m], tenDay=[regular10], flexWeekday=[flexWeekday], flexAll=[flexAll], ticketsOnly=[]
  - `asIs`는 DP 없이 계산한다. planStart부터 `oneMonthEnd`로 regular1m을 연달아 calendarEnd를 덮고, 기간 안 주말 이동일만 표로 산다.
- [ ] **Step 5: 실행해 통과 확인** → PASS
- [ ] **Step 6: 커밋** — `feat: 정기권 조합 최적화와 전략별 비교`

### Task 5: 상태 저장과 계획 범위

**Files:**
- Create: `src/state/storage.ts`, `src/domain/planning.ts`
- Test: `tests/state/storage.test.ts`, `tests/domain/planning.test.ts`

**Interfaces:**
- Consumes: `Fares`, `HolidayOverrides`, `DayMark`, `PriceOverrides`, `PassKind`
- Produces:
  - `type Purchase = { id: string; kind: PassKind; start: DateStr; end: DateStr; price: number }`
  - `type CreatedEvent = { eventId: string; payDate: DateStr }`
  - `type AppState = { version: 1; fares: Fares | null; notifyTime: string /* 'HH:MM', 기본 '20:00' */; initialPassEnd: DateStr | null; holidays: HolidayOverrides; marks: Record<DateStr, DayMark>; overrides: PriceOverrides; purchases: Purchase[]; createdEvents: CreatedEvent[] }`
  - `emptyState(): AppState` — fares는 null (운임 기본값 없음)
  - `loadState(storage?: Storage): AppState` — 키 `'ktx-pass-planner:v1'`. 없거나, 파싱 실패거나, version이 다르면 `emptyState()`. 접근 자체가 throw해도 `emptyState()`.
  - `saveState(s: AppState, storage?: Storage): void` — throw를 삼킨다.
  - `planStart(s: AppState, today: DateStr): DateStr` — max(마지막 구매 end+1 또는 initialPassEnd+1, today)
  - `calendarRange(start: DateStr): { calendarEnd: DateStr; horizonEnd: DateStr }` — calendarEnd = lastDayOfNextMonth(start), horizonEnd = calendarEnd + 31

- [ ] **Step 1: 실패하는 테스트 작성**

```ts
// 저장소 (Review Focus 4)
const mem = new Map<string,string>(); const st = { getItem: (k)=>mem.get(k) ?? null, setItem: (k,v)=>mem.set(k,v) } as Storage
expect(loadState(st)).toEqual(emptyState())
mem.set('ktx-pass-planner:v1', '{not json'); expect(loadState(st)).toEqual(emptyState())
const throwing = { getItem: () => { throw new Error('denied') }, setItem: () => { throw new Error('denied') } } as unknown as Storage
expect(loadState(throwing)).toEqual(emptyState()); expect(() => saveState(emptyState(), throwing)).not.toThrow()
// 왕복 저장
const s = { ...emptyState(), initialPassEnd: '2026-11-30' }; saveState(s, st); expect(loadState(st)).toEqual(s)

// 계획 범위
expect(planStart({ ...emptyState(), initialPassEnd: '2026-11-30' }, '2026-11-25')).toBe('2026-12-01')
expect(planStart({ ...emptyState(), initialPassEnd: '2026-11-30' }, '2026-12-10')).toBe('2026-12-10')   // 오늘이 더 늦음
expect(planStart({ ...emptyState(), initialPassEnd: '2026-11-30', purchases: [{ id:'a', kind:'regular10', start:'2026-12-01', end:'2026-12-10', price:1 }] }, '2026-11-25')).toBe('2026-12-11')
expect(calendarRange('2026-12-11')).toEqual({ calendarEnd: '2027-01-31', horizonEnd: '2027-03-03' })
```

- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: 두 파일 구현**
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 상태 저장과 롤링 계획 범위`

### Task 6: 설정 화면과 앱 뼈대

**Files:**
- Create: `src/ui/App.tsx`, `src/ui/SetupView.tsx`, `src/ui/format.ts`, `src/ui/styles.css`
- Modify: `src/main.tsx`
- Test: `tests/ui/SetupView.test.tsx`, `tests/ui/format.test.ts`

**Interfaces:**
- Consumes: `AppState`, `loadState`, `saveState`, `emptyState`
- Produces:
  - `App` — `fares`가 null이거나 `initialPassEnd`와 구매 기록이 모두 없으면 SetupView, 아니면 하단 탭(달력 / 기록 / 설정)
  - `SetupView({ state, onSave }: { state: AppState; onSave: (s: AppState) => void })`
  - `format.ts`
    - `won(n: number): string` → `'134,400원'`
    - `md(d: DateStr): string` → `'11/3'`
    - `PASS_LABEL: Record<PassKind,string>` = `{ regular10: '10일 정기권', regular1m: '1개월 정기권', flexWeekday: '기간자유형(평일)', flexAll: '기간자유형(주말 포함)' }`

- [ ] **Step 1: 실패하는 테스트 작성**
  - SetupView에 정기권 기준 운임·일반실 편도 입력칸이 **비어 있다** (플레이스홀더만 있고 값 없음).
  - 두 값을 넣고 "현재 정기권 종료일"을 고른 뒤 저장하면 `onSave`가 `{ fares: { baseFare, ticketOneWay, rateShort: 0.45, rateLong: 0.5 }, initialPassEnd, notifyTime: '20:00' }`가 반영된 state로 호출된다.
  - 운임이 0 이하이거나 비어 있으면 저장 버튼이 비활성이다.
  - 할인율·알림 시각 수정칸이 있다.
  - 공휴일 추가/삭제 입력이 있어 `holidays.added`/`removed`를 바꾼다.
  - format 테스트: `won(134400) === '134,400원'`, `md('2026-11-03') === '11/3'`.
- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: 구현** — 스타일은 `styles.css` 하나에 CSS 변수로 둔다. 360px 기준, 입력·버튼 높이 최소 44px.
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 첫 실행 설정 화면과 앱 뼈대`

### Task 7: 달력 화면

**Files:**
- Create: `src/ui/CalendarView.tsx`
- Modify: `src/ui/App.tsx`, `src/ui/styles.css`
- Test: `tests/ui/CalendarView.test.tsx`

**Interfaces:**
- Consumes: `planStart`, `calendarRange`, `isWorkday`, `isHoliday`, `DayMark`, `Purchase`
- Produces: `CalendarView({ state, today, onChange, onCalculate }: { state: AppState; today: DateStr; onChange: (s: AppState) => void; onCalculate: () => void })`

- [ ] **Step 1: 실패하는 테스트 작성**
  - planStart가 속한 달부터 calendarEnd 달까지 월별 그리드가 이어서 렌더된다 (월요일 시작 7열).
  - **평일 칸 탭**: marks에 'vacation'이 생기고 휴가 아이콘(`aria-label="휴가"`)이 보인다. 다시 탭하면 키가 사라진다.
  - **토·일·공휴일 칸 탭**: 'travel'이 생기고 기차 아이콘(`aria-label="이동"`)이 보인다.
  - planStart 이전 날짜는 `disabled`이고, 탭해도 marks가 바뀌지 않는다.
  - 구매 기록 기간에 든 날짜는 `data-owned="true"`다.
  - 공휴일 칸에는 `data-holiday="true"`다.
  - 하단 고정 "계산하기" 버튼을 누르면 `onCalculate`가 호출된다.
- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: 구현** — 아이콘은 lucide-react `Palmtree`(휴가)와 `TrainFront`(이동). 칸 높이 최소 44px, 그리드는 `grid-template-columns: repeat(7, 1fr)`로 가로 스크롤을 없앤다.
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 휴가·이동일 달력`

### Task 8: 결과 화면 — 비교, 가격 수정, 샀어요, 구매 기록

**Files:**
- Create: `src/ui/ResultView.tsx`, `src/ui/PurchasesView.tsx`
- Modify: `src/ui/App.tsx`
- Test: `tests/ui/ResultView.test.tsx`, `tests/ui/PurchasesView.test.tsx`

**Interfaces:**
- Consumes: `compareStrategies`, `overrideKey`, `PassItem`, `PlanResult`, `Purchase`, `format.ts`
- Produces:
  - `ResultView({ state, today, onChange, onCreateEvents }: { …; onCreateEvents: (plan: PlanResult) => Promise<void> })`
  - `PurchasesView({ state, onChange })`
  - `STRATEGY_LABEL: Record<StrategyId,string>` = best '추천', asIs '지금처럼', monthly '1개월 정기권 위주', tenDay '10일 정기권 위주', flexWeekday '기간자유형 평일', flexAll '기간자유형 주말 포함', ticketsOnly '정기권 없이 표만'

- [ ] **Step 1: 실패하는 테스트 작성** (state는 Task 4 시나리오 1·3 재사용)
  - **추천 카드**: 총액 `won(total)`이 보인다. 각 정기권 줄에 `PASS_LABEL`, `md(start)~md(end)`, 가격, `"{md(payDate)} 저녁 결제"`가 보인다. 왕복표 줄은 `"왕복표 N회 · {won}"`이다.
  - 달력 범위를 넘어 끝나는 정기권 줄에는 `"{md(end)}까지"`가 표시된다.
  - **비교 목록**: 6개 전략이 보이고, 각 줄에 `+{won(차액)}`이 있다. 1위와 같으면 `"추천과 같음"`이다.
  - 화면 어디에도 `×`, `=`, `%` 문자가 없다 (계산식 숨김).
  - **가격 수정**: 정기권 가격을 탭해 130000을 입력하면 `overrides[overrideKey(...)] = 130000`으로 onChange되고 다시 계산된다. 그 줄에 `"직접 입력"` 배지가 보인다.
  - **샀어요**: purchases에 `{ kind, start, end, price }`가 추가된다. 같은 기간을 다시 누르면 중복으로 추가되지 않는다.
  - "이 계획으로 일정 만들기"를 누르면 `onCreateEvents(bestPlan)`이 호출된다.
  - **PurchasesView**: 목록 표시, 가격·기간 수정, 삭제(확인 후)가 state에 반영된다.
- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: 구현**
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 결과 비교·가격 수정·구매 기록`

### Task 9: 구글 캘린더 연동

**Files:**
- Create: `src/gcal/gcal.ts`
- Modify: `src/ui/App.tsx`, `index.html` (GIS 스크립트 `https://accounts.google.com/gsi/client` async 로드)
- Test: `tests/gcal/gcal.test.ts`

**Interfaces:**
- Consumes: `PlanResult`, `CreatedEvent`, `PASS_LABEL`, `won`, `md`, `todayInSeoul`
- Produces:
  - `eventTitle(p: PassItem): string` → `'🚄 기간자유형(평일) 결제 — 11/3~11/23 · 96,000원'`
  - `eventBody(p: PassItem, notifyTime: string)` → `{ summary, start: { dateTime: `${payDate}T${notifyTime}:00`, timeZone: 'Asia/Seoul' }, end: 30분 뒤, reminders: { useDefault: true } }`
  - `syncEvents(plan: PlanResult, prev: CreatedEvent[], notifyTime: string, deps: { getToken: () => Promise<string>; fetch: typeof fetch; today: DateStr }): Promise<{ created: CreatedEvent[]; kept: CreatedEvent[]; error?: string }>`
    - `prev` 중 `payDate >= today`인 것을 DELETE한다 (404는 성공으로 본다).
    - plan.passes마다 `POST https://www.googleapis.com/calendar/v3/calendars/primary/events`
    - `kept` = 지난 일정(payDate < today)
  - `getToken()` — `google.accounts.oauth2.initTokenClient({ client_id: import.meta.env.VITE_GOOGLE_CLIENT_ID, scope: 'https://www.googleapis.com/auth/calendar.events' })`. 거부나 오류면 reject.

- [ ] **Step 1: 실패하는 테스트 작성** (fetch·getToken은 가짜로 주입)
  - `eventTitle`이 위 예시 문자열과 정확히 같다 (`flexWeekday`, 2026-11-03~2026-11-23, 96000).
  - `eventBody(..., '20:00').start.dateTime === '2026-11-02T20:00:00'`, `end.dateTime === '2026-11-02T20:30:00'`
  - **교체**: 미래 이벤트 2개는 DELETE되고, 지난 이벤트 1개는 그대로 kept, plan의 정기권 2개는 POST되어 created에 들어간다.
  - **getToken reject** (Review Focus 5): fetch가 한 번도 호출되지 않고, `error`가 `'구글 로그인이 취소되었거나 실패했어요'`이며, prev는 그대로다.
  - **두 번째 POST가 500**: 첫 번째로 만든 일정은 created에 남고, `error`는 `'일부 일정만 만들었어요'`다.
- [ ] **Step 2: 실행해 실패 확인** → FAIL
- [ ] **Step 3: 구현** — App의 `onCreateEvents`가 `syncEvents` 결과로 `createdEvents = [...kept, ...created]`를 저장하고, `error`면 화면 상단에 토스트를 띄운다. `VITE_GOOGLE_CLIENT_ID`가 비어 있으면 버튼을 비활성화하고 "구글 연동 설정 전"이라고 표시한다.
- [ ] **Step 4: 실행해 통과 확인** → PASS
- [ ] **Step 5: 커밋** — `feat: 구글 캘린더 결제일 일정`

### Task 10: 배포와 홈 화면 추가

**Files:**
- Create: `.github/workflows/deploy.yml`, `public/manifest.webmanifest`, `public/icon-192.png`, `public/icon-512.png`, `docs/SETUP.md`
- Modify: `index.html` (manifest 링크, `apple-mobile-web-app-capable`, theme-color, viewport)

- [ ] **Step 1: GitHub Pages 배포 워크플로 작성**
  - main 푸시 시 `npm ci && npm test && npm run build`를 실행한다.
  - `actions/upload-pages-artifact`로 `dist/`를 올리고 `actions/deploy-pages`로 배포한다.
  - 빌드 env `VITE_GOOGLE_CLIENT_ID: ${{ vars.GOOGLE_CLIENT_ID }}` (레포 변수. 공개 가능한 ID라 Secret이 아니다).
- [ ] **Step 2: manifest와 아이콘**
  - `name: 'KTX 정기권 플래너'`, `display: 'standalone'`, `start_url: '/ktx-pass-planner/'`
  - 아이콘은 단색 기차 모양을 단순하게 그린다.
- [ ] **Step 3: `docs/SETUP.md` 작성** — 사용자가 직접 할 일을 순서대로 적는다.
  1. 레포 Settings → Pages → Source를 "GitHub Actions"로 바꾼다.
  2. Google Cloud Console에서 새 프로젝트를 만들고 Google Calendar API를 사용 설정한다.
  3. OAuth 동의 화면을 외부·테스트 상태로 만들고, 본인 gmail을 테스트 사용자로 등록한다.
  4. OAuth 클라이언트 ID(웹)를 만든다. 승인된 JavaScript 출처는 `https://seong4687.github.io`이다.
  5. 레포 Settings → Variables에 `GOOGLE_CLIENT_ID`를 등록한다.
  - 운임·구간 같은 개인 값은 문서에 쓰지 않는다.
- [ ] **Step 4: 검증**
  - `npm test && npm run build` 성공
  - `npx vite preview` 후 360px 폭에서 가로 스크롤이 없는지 확인한다.
  - push 후 Actions가 초록이고, `https://seong4687.github.io/ktx-pass-planner/`가 열리는지 확인한다.
- [ ] **Step 5: 커밋·푸시** — `chore: GitHub Pages 배포와 홈 화면 추가 지원`
