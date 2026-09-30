# speed — 속도·토큰 사용량·Fast 모드

## 무엇을 테스트했나

두 가지를 쟀습니다.

1. **effort별 속도와 토큰 사용량.** 같은 질문을 reasoning effort 5단계로 던졌을 때 첫 글자까지 걸리는 시간(TTFT), 전체 응답 시간, 추론토큰, 스트리밍 속도가 어떻게 바뀌는지.
2. **Fast 모드.** `service_tier: "priority"`(Codex에서 "Fast"로 표시)를 켜면 실제로 얼마나 빨라지는지, 그리고 `ultrafast` 값이 효과가 있는지.

측정 환경은 ChatGPT Pro 구독의 Codex 환경이고, 측정일은 2026-09-30입니다.

## 방법

### 1) effort 스케일링 (225회 호출, 실패 0)

| 항목 | 내용 |
|---|---|
| 모델 | gpt-6.1-sol, gpt-6-sol, gpt-6-astra, gpt-6-luna, gpt-5.6-sol |
| effort | low, medium, high, xhigh, max |
| 프롬프트 | `short_fact`: "What is the capital city of Australia? Answer with the city name only, nothing else." |
| | `medium_explain`: HTTPS(TLS) 연결 수립 과정을 약 300단어 산문으로 설명 |
| | `reasoning`: 1≤a,b≤60 정수 순서쌍 중 a·b가 완전제곱인 개수. 마지막 줄 `ANSWER: <정수>` 요구. 정답 158 |
| 반복 | 셀(모델×effort×프롬프트)당 N=3 → 5×5×3×3 = 225회 |
| 순서 | 225개 호출을 고정 시드로 한 줄 셔플. 모델·effort가 시간대에 고르게 섞이게 했습니다 |
| 동시성 | 3 |
| 요약 | 셀별 중앙값과 IQR |
| 채점 | short_fact는 "canberra" 포함 여부, reasoning은 `ANSWER:` 뒤 정수가 158인지. 속도 트랙의 부수 지표입니다 |

지표 정의

- **TTFT**: 요청 송신부터 첫 가시 텍스트 조각 수신까지. 추론 시간이 여기에 들어갑니다.
- **총시간**: 요청 송신부터 스트림 종료까지.
- **추론토큰**: 응답 usage의 `output_tokens_details.reasoning_tokens`.
- **가시 tok/s**: (출력토큰 − 추론토큰) ÷ (총시간 − TTFT). 첫 글자 이후 눈에 보이는 텍스트가 흘러나오는 속도입니다.

사전 점검으로 gpt-6.1-sol이 받는 effort 값을 확인했습니다. low·medium·high·xhigh·max는 받고, `none`·`minimal`은 HTTP 400(`not supported with the 'gpt-6.1-sol' model`)이었습니다.

### 2) Fast 모드 (67회 호출)

| 항목 | 내용 |
|---|---|
| 모델 | 위 5개 |
| 프롬프트 | `mid_250w`: TCP 혼잡 제어 약 250단어 설명 / `long_600w`: 멱등 REST API 설계 가이드 약 600단어 |
| 설계 | 기본 호출과 Fast 호출을 **짝으로 연달아** 실행. 짝 안의 순서는 반복마다 뒤집음 |
| 반복 | 조합당 3쌍. effort=low, 동시성 1 |
| ultrafast | gpt-6.1-sol × mid_250w에서 기본 vs `ultrafast` 3쌍 |
| 지표 | 짝마다 (Fast tok/s ÷ 기본 tok/s)를 구하고 그 중앙값을 배율로 씀 |

Fast 트랙의 tok/s는 출력토큰 ÷ (총시간 − TTFT)입니다(추론토큰 포함. effort=low라 대부분 0). 실험 1의 가시 tok/s와 정의가 조금 다릅니다.

**데이터 처리:** Fast 트랙에서 스트림이 끊겨 usage가 오지 않은 호출 1건(gpt-6-astra, mid_250w, 기본, rep 2)은 원본 집계와 같게 재호출본으로 교체했습니다. `fast_mode_calls.csv`에 원기록을 `stream_incomplete=true, used_in_summary=false`로 남겨 두었습니다. 실험 1(225회)에는 끊긴 스트림이 없었습니다.

## 결과

### 모델별 대표값 (실험 1, 중앙값)

| 모델 | 가시 tok/s (medium_explain, effort 전체 15회) | TTFT short_fact·low (s) | reasoning 총시간 low → max (s) | reasoning 추론토큰 low → max | reasoning 정답 |
|---|---|---|---|---|---|
| **gpt-6.1-sol** | **31.1** | 1.87 | 17.6 → 53.5 | 283 → 1399 (×4.94) | 15/15 |
| gpt-6-sol | 46.0 | 1.29 | 9.4 → 22.6 | 259 → 1034 (×3.99) | 12/15 |
| gpt-6-astra | 33.0 | 1.87 | 18.7 → 44.0 | 291 → 1195 (×4.11) | 15/15 |
| gpt-6-luna | 49.2 | 1.40 | 19.9 → 41.0 | 654 → 1145 (×1.75) | 11/15 |
| gpt-5.6-sol | 54.9 | 1.33 | 14.3 → 24.5 | 470 → 1034 (×2.20) | 15/15 |

short_fact는 5개 모델 모두 15/15(합계 75/75) 정답(Canberra)이었습니다.

### effort별 추론토큰 중앙값

**reasoning 프롬프트**

| 모델 | low | medium | high | xhigh | max | Spearman |
|---|---|---|---|---|---|---|
| **gpt-6.1-sol** | 283 | 306 | 516 | 1034 | 1399 | 1.00 |
| gpt-6-sol | 259 | 413 | 476 | 516 | 1034 | 1.00 |
| gpt-6-astra | 291 | 322 | 373 | 786 | 1195 | 1.00 |
| gpt-6-luna | 654 | 596 | 578 | 869 | 1145 | 0.60 |
| gpt-5.6-sol | 470 | 924 | 966 | 967 | 1034 | 1.00 |

**medium_explain 프롬프트**

| 모델 | low | medium | high | xhigh | max |
|---|---|---|---|---|---|
| **gpt-6.1-sol** | 0 | 51 | 291 | 1034 | 1405 |
| gpt-6-sol | 0 | 71 | 152 | 620 | 843 |
| gpt-6-astra | 0 | 0 | 100 | 835 | 1331 |
| gpt-6-luna | 0 | 49 | 110 | 348 | 482 |
| gpt-5.6-sol | 46 | 67 | 80 | 146 | 294 |

### gpt-6.1-sol 상세 (중앙값, 괄호는 IQR)

| 프롬프트 | effort | TTFT (s) | 총시간 (s) | 출력토큰 | 추론토큰 |
|---|---|---|---|---|---|
| short_fact | low | 1.9 (0.1) | 2.1 (0.1) | 6 | 0 |
| short_fact | max | 2.8 (0.3) | 3.1 (0.3) | 24 | 16 |
| medium_explain | low | 2.6 (0.3) | 15.2 (0.1) | 369 | 0 |
| medium_explain | high | 15.4 (3.1) | 27.6 (3.6) | 646 | 291 |
| medium_explain | max | 47.8 (12.3) | 59.1 (13.8) | 1754 | 1405 |
| reasoning | low | 11.2 (1.0) | 17.6 (1.4) | 485 | 283 |
| reasoning | high | 18.3 (1.7) | 26.8 (1.1) | 783 | 516 |
| reasoning | max | 47.0 (8.3) | 53.5 (7.9) | 1602 | 1399 |

75개 셀 전체는 `speed_cells_median.csv`에 있습니다. 그래프: `effort_scaling_total_s.png`(총시간), `effort_scaling_reasoning_tokens.png`(추론토큰, 로그 축).

### Fast 모드 (짝별 배율의 중앙값 [최소–최대])

| 모델 | 프롬프트 | 기본 → Fast tok/s | tok/s 배율 | 총시간 배율 |
|---|---|---|---|---|
| **gpt-6.1-sol** | 250w | 30.5 → 47.1 | ×1.65 [1.50–1.67] | ×1.31 |
| **gpt-6.1-sol** | 600w | 28.8 → 57.3 | ×1.93 [1.84–2.17] | ×1.96 |
| gpt-6-sol | 250w | 42.2 → 79.5 | ×1.91 [1.61–2.40] | ×1.64 |
| gpt-6-sol | 600w | 43.5 → 81.4 | ×1.87 [1.50–2.01] | ×1.65 |
| gpt-6-astra | 250w | 32.6 → 48.7 | ×1.58 [1.45–2.65] | ×1.42 |
| gpt-6-astra | 600w | 33.3 → 50.3 | ×1.52 [1.51–1.66] | ×1.46 |
| gpt-6-luna | 250w | 34.6 → 81.6 | ×2.48 [1.59–3.69] | ×2.02 |
| gpt-6-luna | 600w | 55.2 → 82.6 | ×1.56 [1.49–2.11] | ×1.46 |
| gpt-5.6-sol | 250w | 62.0 → 81.1 | ×1.32 [1.21–1.44] | ×1.32 |
| gpt-5.6-sol | 600w | 58.1 → 84.7 | ×1.73 [1.42–1.80] | ×1.38 |
| gpt-6.1-sol **ultrafast** | 250w | 26.5 → 26.2 | ×0.99 [0.93–1.17] | ×1.03 |

`service_tier` 허용값: `default`·`priority`·`ultrafast`는 받고, `fast`·`ultra`·`flex`·`turbo`·`auto`는 HTTP 400(`Unsupported service_tier`)이었습니다. 응답 이벤트의 `service_tier`는 priority로 불러도 항상 `default`로 돌아와서, 적용 여부는 속도로만 확인할 수 있었습니다. Codex 모델 카탈로그는 모든 모델에 Fast 하나만 제공하고, gpt-6.1-sol·gpt-6-astra의 Fast 설명은 "2x speed, increased usage"입니다.

## gpt-6.1-sol 관찰

**관측**

1. 가시 스트리밍 속도가 약 31 tok/s로, effort·프롬프트가 바뀌어도 거의 일정했습니다(reasoning 프롬프트 5단계 모두 30.8~31.4, medium_explain은 xhigh 셀 43.7 하나를 빼면 29.1~32.2). gpt-6-astra(33.0)와 같은 대역이고, gpt-6-sol·gpt-6-luna·gpt-5.6-sol(46~55)보다 느렸습니다.
2. effort를 올리면 추론토큰이 단조 증가했습니다(medium_explain·reasoning 모두 Spearman 1.00). reasoning 프롬프트의 max/low 배율 4.94는 수치상 5개 중 가장 크지만, gpt-6-astra 4.11·gpt-6-sol 3.99도 Spearman 1.00이고 N=3이라 "effort를 가장 잘 따르는 모델"이라고 가를 근거는 없습니다.
3. xhigh/max에서 추론토큰이 많고 디코딩이 느린 것이 겹쳐, reasoning max 총시간 53.5초는 5개 중 가장 길었습니다. medium_explain max 59.1초는 gpt-6-astra 59.6초와 같은 급이었습니다. 같은 reasoning 문제를 gpt-6-sol max는 22.6초에 끝냈습니다.
4. reasoning 문제는 low에서 이미 3/3 정답이었습니다. 이 난도에서는 effort를 올려 얻은 정확도 이득이 관측되지 않았습니다(천장 효과).
5. short_fact low의 TTFT는 1.87초로 gpt-6-sol(1.29)·gpt-5.6-sol(1.33)·gpt-6-luna(1.40)보다 약 0.5초 늦고 gpt-6-astra(1.87)와 같았습니다.
6. Fast 모드에서 tok/s가 ×1.65(250w)~×1.93(600w) 빨라졌습니다. 600w 기준 총시간이 33.6초 → 16.6초로 줄었습니다.
7. `ultrafast`는 에러 없이 받지만 속도 변화가 없었습니다(×0.99).

**해석**

- 긴 글을 받을 때 체감 속도는 TTFT보다 디코딩 속도가 좌우합니다. 300단어 설명을 low로 받으면 gpt-6-sol 9.7초, gpt-6.1-sol 15.2초였고, 차이 대부분이 디코딩 속도에서 나옵니다.
- 이 수준의 문제라면 low~medium이 적당해 보입니다. reasoning 총시간이 low 17.6초 → medium 19.8초로 거의 안 늘다가 high부터 가파르게 늘었습니다.
- TTFT 0.5초 차이, 느린 디코딩의 원인은 이 데이터로 판정할 수 없습니다.

## 한계

- ChatGPT Pro 구독의 Codex 환경에서 잰 값입니다. 공식 API의 지연·처리량과 다를 수 있습니다.
- 같은 시간에 다른 평가 트랙도 돌고 있었습니다. 대기열 지연이 TTFT·총시간에 섞였을 수 있고, 셔플로 모델 간 편향만 줄였습니다.
- Fast 트랙에서 부하 조건을 맞춘 것은 짝(기본·Fast) 안에서뿐입니다. 모델끼리는 순차로 돌았으므로 **모델 간 절대 tok/s 비교는 짝 설계의 보호를 받지 못합니다.** 예를 들어 250w 기본에서 gpt-6-astra 32.6 대 gpt-6.1-sol 30.5는 구분 불가입니다.
- 셀당 N=3이라 IQR과 배율 범위가 거칩니다(gpt-6-luna 250w 배율 1.59~3.69). 배율은 대략적인 크기만 보시길 권합니다.
- 프롬프트가 고정돼 있어 다른 과제에서는 effort 스케일링이 다를 수 있습니다. 정답률은 부수 지표입니다.
- 추론토큰 1034가 여러 모델·effort에서 9번 반복 관측됐습니다(516도 4번). 보고값이 버킷 단위로 잘리는지는 이 데이터로 판정할 수 없습니다.
- Fast 모드의 사용량 추가 차감 폭은 측정하지 않았습니다.
- 추론 요약 스트림은 측정하지 않았습니다.

## 파일

### `speed_calls.csv` — 실험 1 호출 단위 (225행)

| 컬럼 | 뜻 |
|---|---|
| model, effort, prompt | 모델, reasoning effort, 프롬프트 이름 |
| rep | 셀 안 반복 번호(0~2) |
| run_order | 셔플된 실행 순서(0~224) |
| concurrency_at_start | 호출 시작 시점의 동시 실행 수 |
| ttft_s | 첫 가시 텍스트까지 시간(초) |
| total_s | 스트림 종료까지 시간(초) |
| input_tokens, output_tokens | 입력·출력 토큰(출력은 추론토큰 포함) |
| reasoning_tokens | 추론토큰 |
| visible_tokens | output_tokens − reasoning_tokens |
| tok_s_overall | output_tokens ÷ total_s |
| visible_tok_s_stream | visible_tokens ÷ (total_s − ttft_s) |
| correct | 정답 여부(short_fact·reasoning만. medium_explain은 빈칸) |
| answer_parsed | short_fact는 답한 도시명, reasoning은 `ANSWER:` 뒤 정수(없으면 빈칸) |

### `speed_cells_median.csv` — 셀별 요약 (75행)

model, prompt, effort, n_ok, n_fail과 각 지표의 `_median`(중앙값)·`_iqr`(사분위 범위, N=3이라 거침). correct/graded는 셀 안 정답 수/채점 수입니다.

### `speed_effort_scaling.csv` — effort 스케일링 (15행 = 프롬프트 3 × 모델 5)

| 컬럼 | 뜻 |
|---|---|
| reasoning_tokens_median_{effort} | effort별 추론토큰 중앙값 |
| total_s_median_{effort} | effort별 총시간 중앙값(초) |
| spearman_effort_vs_reasoning_tokens | effort 순위와 추론토큰의 스피어만 상관. 전부 0이면 빈칸 |
| max_over_low_reasoning_ratio | max ÷ low 추론토큰. low가 0이면 빈칸 |
| max_minus_low_reasoning_tokens | max − low 추론토큰 |
| strictly_monotonic | effort를 올릴 때마다 추론토큰이 엄격히 증가했는지 |

### `speed_model_summary.csv` — 모델별 대표값 (5행)

위 "모델별 대표값" 표의 원자료입니다. 가시 tok/s는 medium_explain 15회의 중앙값·q1·q3, TTFT는 short_fact low 3회와 전 effort 15회 중앙값, reasoning 총시간·추론토큰은 low·max 셀 중앙값, 정답 수는 15회 기준입니다.

### `fast_mode_calls.csv` — Fast 트랙 호출 단위 (67행)

| 컬럼 | 뜻 |
|---|---|
| tier | `default`, `priority`(Fast), `ultrafast` |
| rep | 반복 번호(0~2) |
| order_in_pair | 짝 안에서 먼저(0) 불렸는지 나중(1)인지 |
| ttft_s, total_s | 첫 텍스트까지·스트림 종료까지 시간(초) |
| output_tokens, reasoning_tokens | 출력·추론 토큰. 끊긴 스트림은 빈칸 |
| tok_s | output_tokens ÷ (total_s − ttft_s) |
| stream_incomplete | 완료 이벤트 없이 끊긴 호출 |
| is_retry_of_incomplete | 끊긴 호출을 대신한 재호출 |
| used_in_summary | 요약·짝 계산에 쓰였는지 |

### `fast_mode_pairs.csv` — 짝 단위 (33행)

기본 호출과 Fast(또는 ultrafast) 호출 한 쌍이 한 행입니다. `first_in_pair`는 짝에서 먼저 불린 쪽, `speedup_tok_s` = fast_tok_s ÷ default_tok_s, `speedup_total` = default_total_s ÷ fast_total_s(1보다 크면 Fast가 빠름).

### `fast_mode_cells.csv` — 조합별 요약 (11행)

모델×프롬프트×tier마다 기본·Fast의 tok/s·총시간·TTFT 중앙값과, 짝별 배율의 중앙값·최소·최대입니다. 중앙값끼리 나눈 값이 아니라 **짝별 배율의 중앙값**이라 `fast_tok_s_median ÷ default_tok_s_median`과 약간 다를 수 있습니다.

### `fast_mode_tier_values.csv`

`service_tier`에 넣어 본 값별로 받아들여졌는지와 관측된 효과입니다.

### 그래프

- `effort_scaling_total_s.png`: 프롬프트별 effort → 총시간 중앙값
- `effort_scaling_reasoning_tokens.png`: 프롬프트별 effort → 추론토큰 중앙값(로그 축)
