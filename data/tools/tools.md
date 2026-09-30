# 툴 정의와 픽스처 데이터

function calling·에이전트 루프에서 모델에게 준 가짜 툴 5종의 스키마 설명과, 툴이 돌려준 데이터다. 툴은 실제 외부 서비스에 붙지 않고 아래 표의 고정 데이터로만 답한다. 그래서 정답을 같은 데이터에서 계산할 수 있고, 채점에 LLM 판정이 들어가지 않는다.

일정·고객 표의 인명은 모두 테스트용으로 지어낸 이름이다.

## 툴 5종

모든 툴은 Responses API의 function 툴 형식이고, `additionalProperties: false` 에 필수 인자를 `required` 로 지정했다.

| 툴 | 설명(모델에게 준 문구) | 인자 | 필수 |
|---|---|---|---|
| `get_weather` | 도시의 현재 날씨(기온·상태·습도)를 조회한다. | `city`: string (영문 권장, 예: Seoul) · `unit`: enum `celsius`/`fahrenheit` (기본 celsius) | city |
| `convert_currency` | 금액을 한 통화에서 다른 통화로 환산한다 (ISO 4217 코드). | `amount`: number · `from_currency`: string (예: USD) · `to_currency`: string (예: KRW) | 셋 다 |
| `get_schedule` | 특정 날짜의 사내 일정 목록을 조회한다. person 을 주면 그 사람 일정만. | `date`: string (YYYY-MM-DD) · `person`: string (선택, 참석자 이름) | date |
| `query_db` | 사내 주문 DB에 SQL(SELECT)을 실행하고 행 목록을 돌려준다. 설명에 테이블 스키마를 같이 적었다. | `sql`: string | sql |
| `calculator` | 사칙연산·거듭제곱·괄호 수식을 정확히 계산한다. 예: (1234*5678)/7 | `expression`: string | expression |

툴 쪽 동작:

- `get_weather`: 도시명은 한글·영문·약칭(예: 서울, seoul, nyc)을 표준 영문명으로 맞춘 뒤 조회한다. 목록에 없는 도시는 오류를 돌려준다. 화씨 요청이면 섭씨 값을 변환해 준다.
- `convert_currency`: 아래 고정 환율로 USD를 거쳐 환산하고 소수 둘째 자리에서 반올림한다. 지원하지 않는 통화면 오류.
- `get_schedule`: 날짜가 정확히 일치하는 일정만 돌려준다. `person` 은 부분 일치.
- `query_db`: 인메모리 SQL 데이터베이스에서 실행한다. SELECT/WITH 로 시작하지 않으면 거부하고, 최대 50행을 돌려준다.
- `calculator`: 사칙연산·거듭제곱·나머지·괄호만 계산한다. 함수 호출 같은 다른 식은 오류(`unsupported expression`)를 돌려준다. 에이전트 과제에서 집계한 "툴 오류"는 전부 이 경우였다.

## 픽스처 데이터

기준일: 2026-09-30 (수요일). 모델에게 주는 지시문(instructions)에 이 날짜를 넣었다.

### 날씨

| 도시 | 기온(°C) | 상태 | 습도(%) |
|---|---|---|---|
| Seoul | 18 | Clear | 45 |
| Busan | 21 | Cloudy | 60 |
| Tokyo | 22 | Rain | 80 |
| New York | 12 | Windy | 50 |
| London | 11 | Rain | 85 |
| Paris | 14 | Cloudy | 70 |
| Berlin | 10 | Clear | 55 |
| Osaka | 23 | Clear | 65 |

### 환율 (1단위 = USD)

| 통화 | USD 환산 |
|---|---|
| USD | 1.0 |
| KRW | 1/1350 |
| EUR | 1.08 |
| JPY | 1/150 |
| GBP | 1.25 |

### 일정

| 날짜 | 시각 | 제목 | 참석자 | 장소 |
|---|---|---|---|---|
| 2026-09-25 | 10:00 | 주간회의 | 김민수 | Seoul |
| 2026-09-30 | 14:00 | 코드리뷰 | 김민수 | Seoul |
| 2026-09-30 | 09:00 | 고객미팅 | Emma Brown | London |
| 2026-10-01 | 10:00 | 출장 | 박지은 | Tokyo |
| 2026-10-01 | 15:00 | 1on1 | 김민수 | Seoul |
| 2026-10-02 | 11:00 | 워크숍 | 박지은 | Paris |
| 2026-10-05 | 09:30 | 분기회의 | 김민수 | Seoul |
| 2026-10-05 | 13:00 | 파트너미팅 | Tanaka Yuki | Tokyo |
| 2026-10-05 | 16:00 | 리뷰 | John Smith | New York |

### DB: customers

`customers(id INTEGER, name TEXT, city TEXT, currency TEXT)`

| id | name | city | currency |
|---|---|---|---|
| 1 | 김민수 | Seoul | KRW |
| 2 | 박지은 | Busan | KRW |
| 3 | John Smith | New York | USD |
| 4 | Tanaka Yuki | Tokyo | JPY |
| 5 | Emma Brown | London | GBP |
| 6 | Sato Ken | Tokyo | JPY |
| 7 | Lucas Martin | Paris | EUR |

### DB: orders

`orders(id INTEGER, customer_id INTEGER, amount_usd REAL, created_at TEXT 'YYYY-MM-DD')`

| id | customer_id | amount_usd | created_at |
|---|---|---|---|
| 101 | 1 | 120.0 | 2026-08-03 |
| 102 | 1 | 80.5 | 2026-09-02 |
| 103 | 2 | 300.0 | 2026-08-15 |
| 104 | 3 | 950.0 | 2026-09-10 |
| 105 | 3 | 40.0 | 2026-09-12 |
| 106 | 4 | 610.0 | 2026-09-05 |
| 107 | 4 | 75.25 | 2026-08-21 |
| 108 | 5 | 220.0 | 2026-09-18 |
| 109 | 5 | 180.0 | 2026-09-27 |
| 110 | 6 | 1200.0 | 2026-09-20 |
| 111 | 7 | 55.0 | 2026-08-30 |
| 112 | 1 | 310.0 | 2026-09-25 |
| 113 | 1 | 45.0 | 2026-09-28 |
| 114 | 2 | 99.9 | 2026-09-29 |
| 115 | 7 | 410.0 | 2026-09-14 |
| 116 | 1 | 60.0 | 2026-08-11 |

## 지시문

- function calling: "너는 사내 비서다. 오늘은 2026-09-30 (수요일). 필요한 경우에만 도구를 호출하고, 도구가 필요 없는 질문에는 도구 없이 바로 답한다. 독립적인 조회가 여러 개면 한 번에 병렬로 호출해도 된다."
- 에이전트 루프: "너는 도구를 쓰는 업무 에이전트다. 오늘은 2026-09-30 (수요일). 사실 정보(날씨·환율·일정·DB)는 추측하지 말고 반드시 도구로 조회한다. 필요하면 여러 단계에 걸쳐 도구를 호출한다. 작업이 끝나면 답변의 마지막 줄을 정확히 '최종 답: <값>' 형식으로 쓴다. 값이 여러 개면 ' | ' 로 구분하고, 숫자에는 단위나 통화기호를 붙이지 않는다."
