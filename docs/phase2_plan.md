# Analog 2차 과제 프로젝트 방향 (256×256 DVS Array + AER Readout)

> 참고 논문: de Oliveira et al., "Asynchronous time-based imager with DVS sharing", AICSP 2021 (이하 [P])
> 마감: **10/30(금)** — 회로도, Layout, Post-Sim 결과, 기능 점검 결과, 면적, 전력소모, 설계 분석 자료, 10분 발표 녹화

## 0. 현재 위치

| 과제 | 내용 | [P] 대응 | 상태 |
|---|---|---|---|
| 1차 | DVS pixel (photoreceptor + differencing amp + ON/OFF comparator), in-pixel event memory | Fig. 1 | 완료 |
| 2차-① | 256×256 pixel array 설계 | Fig. 10 (matrix 구성) | **이번 과제** |
| 2차-② | Readout 효율 향상 AER 설계 및 적용 | Fig. 2, 3, 6, 7 (+ 개선) | **이번 과제** |

- [P]의 Capture module(Fig. 4, 5, ATIS 휘도 측정)과 DVS sharing은 **범위 밖**으로 둔다. 과제가 DVS + AER이고, 마감까지 3주뿐이다.
- [P]에서 가져올 것: ① pixel↔arbiter handshake 회로 (Fig. 2, 3), ② binary-tree arbiter (Fig. 6, 7), ③ array/주변회로 연결 구조 (Fig. 10), ④ **Verilog-A 행위 모델로 대형 array를 검증하는 방법론** (Sec. 5, 6).
- 차별화 포인트: [P]의 AER은 이벤트 1개당 row arb → col arb 왕복을 하는 **고전적 AER**이다. 이것을 baseline으로 두고, 처리량을 높인 **개선 AER**과 정량 비교하는 것을 발표의 축으로 삼는다. 과제 문구("Readout 효율 향상을 위한 AER")에 그대로 대응한다.

## 1. 전체 아키텍처

```
            ┌──────────── Column latch / ON-OFF 2bit × 256 ───────────┐
            │      Column scanner (token / greedy tree) → 8b X addr     │
  Row       ├──────────────────────────────────────────────────────────┤
  arbiter   │                                                          │
  (8-level  │               256 × 256 DVS pixel array                  │──► AER out bus
  binary    │      pixel = 1차 DVS + req/ack/reset interface           │    {Y[7:0], X[7:0], POL}
  tree)     │                                                          │    + REQ/ACK (4-phase)
  → 8b Y    │                                                          │
            └──────────────────────────────────────────────────────────┘
  Bias gen (Vbias_pr, sf, cas, diff, ON, OFF, refr) — 외부 인가 또는 간단한 global bias
```

### 1.1 Pixel interface ([P] Fig. 2, 3 기반)
- 신호선: 행마다 `REQ_ROW`, `ACK_ROW` / 열마다 `REQ_ON_COL`, `REQ_OFF_COL`, `ACK_COL` (ON/OFF 열 라인 분리는 [P]와 동일).
- 1차의 in-pixel event memory(ON/OFF latch)가 row request를 구동하고, ACK_ROW를 받으면 열 request로 polarity를 내보낸다.
- **Reset ([P] Fig. 3b)**: `ACK_ROW && ACK_COL`일 때만 C2를 방전한다. refractory bias(Vref_reset)로 reset pulse 폭을 조절해 hot pixel / 이벤트 폭주를 제한한다.
- **Row request 보정 ([P] Fig. 3a)**: 256개 pixel이 한 행 라인을 공유하므로, 임계값 근처 pixel 여러 개의 전류가 합쳐져 오검출될 수 있다. 이것을 막는 Mc1~Mc4 구조(인버터로 기울기 샤프닝)를 반드시 넣는다.
- 행·열 라인은 wired-OR(pull-down) + 주변부 pull-up + 인버터 구조다 ([P] Fig. 7b). **256 fan-in 때문에 생기는 누설과 라인 커패시턴스**가 2차의 핵심 아날로그 이슈다. 아래 3장의 검증 항목에서 다룬다.

### 1.2 AER: Baseline vs 개선안

| 항목 | Baseline ([P] 방식) | 개선안 (제안) |
|---|---|---|
| 방식 | 이벤트 1개마다 row arb → col arb → reset | **Row-burst (word-serial) AER**: row 1번 승인 → 그 행의 활성 pixel 전부를 column latch로 한 번에 읽고, column scanner가 순차 출력 |
| 이벤트당 handshake | row + col 2회 | 행당 row 1회 + 이벤트당 col 1회 (row arb 비용을 분산) |
| Column 처리 | col arbiter tree | column latch + 우선순위 encoder / token scanner (fair) |
| 출력 | {Y, X, POL} | {Y} 1회 + {X, POL} 연속 (또는 매번 {Y,X,POL}로 포맷 통일) |
| 기대 효과 | 단순하고 [P]로 검증된 구조 | 장면에 edge가 있을 때 (같은 행에 이벤트가 몰릴 때) 처리량 수 배, 이벤트당 에너지 감소 |

- 참고: Lichtsteiner 2008 ([P] ref. 8)과 Brandli 2014 DAVIS ([P] ref. 13)가 쓰는 계열이다. 시간이 남으면 추가 옵션으로 **group address(예: 8-pixel 그룹 bitmap) 출력**을 넣는다 (Samsung DVS, [P] ref. 9 계열). 그룹 안에 이벤트가 여러 개면 출력 word 수가 줄어든다.
- Arbiter cell: [P] Fig. 6b/7a의 decision cell(NAND SR latch + ack 로직)을 그대로 쓰고 8단 tree(256 입력)로 구성한다. Greedy 연결(상위 ack가 유지되는 동안 같은 subtree에서 연속 승인)로 지연을 줄이되, 공정성(starvation)도 함께 측정한다.
- Handshake: 외부 4-phase REQ/ACK. 테스트벤치에서는 receiver를 Verilog-A/Verilog로 모델링하고 응답지연을 파라미터로 둔다.

## 2. 설계 산출물 (2차 제출 항목 매핑)

| 제출 항목 | 산출물 |
|---|---|
| 회로도 | pixel(1차 + AER interface), arbiter DC cell / 8-level tree, column latch + scanner + encoder, pull-up/인버터 line driver, top 256×256 |
| Layout | pixel cell(pitch 확정), arbiter cell, column cell, top array 조립 (mosaic/instance array), DRC/LVS clean |
| Post-Sim | pixel / arbiter / column cell은 RC extracted. Array는 extracted 라인 RC를 반영한 계층적 시뮬레이션 |
| 기능 점검 | 1×1 → 2×2 → 8×8 transistor-level, 256×256은 Verilog-A pixel + 실제 주변회로 (3장) |
| 면적 | pixel pitch, fill factor, 주변회로 면적, 전체 die 면적 표 |
| 전력 | 정적 전력(pixel bias × 65,536), 이벤트당 에너지(pJ/event), 이벤트율에 따른 전력 곡선 |
| 분석 | Baseline vs 개선 AER 처리량, 지연, 공정성, 에너지 비교. [P] Table 1처럼 소자 수 표 |

## 3. 검증 전략 ([P] Sec. 5–6 방법론 차용)

256×256을 transistor-level로 시뮬레이션하는 것은 불가능하다 ([P]는 4×4에도 2시간, 8×8은 수일이 걸린다고 보고). 그래서 **계층적으로 검증**한다.

1. **L1. Pixel + interface (transistor, post-layout)**: [P] Fig. 11/13 자극(1 pA→10 nA→1 pA 램프, 50 µs / 5 ms / 500 ms)으로 Vdiff, ON/OFF, reset 파형을 확인하고 이벤트 수를 센다. 1차 결과와 연속성이 있어야 한다.
2. **L2. 소형 array (transistor)**: 2×2, 4×4, (가능하면) 8×8을 arbiter와 함께 돌린다. 동시 요청 충돌 해결, ack→reset 순서, 오검출 없음을 확인한다.
3. **L3. Verilog-A pixel 모델**: [P] Listing 1–5 구조(log 차분, ON/OFF cross, row→col request, ack 후 reset)에 **1–5 ns pseudo-random 지연**을 넣는다 (동시 요청 때문에 arbiter가 실패하는 시뮬레이션 artefact 방지, [P] Sec. 5). L1/L2 결과와 이벤트 수 / 타이밍을 비교해 보정하고, 오차를 [P] Table 3처럼 표로 제시한다.
4. **L4. 256×256 top**: Verilog-A pixel × 65,536 + transistor-level(또는 extracted) arbiter / column 회로로 돌린다. 시간이 부족하면 주변회로를 Verilog로 두고, 한 행·한 열 경로만 transistor로 두는 mixed 구성을 쓴다. 자극:
   - 균일 랜덤 이벤트(이벤트율 sweep) → 최대 처리량(Meps), 포화점
   - 이동 edge / bar (같은 행에 이벤트가 몰리는 경우) → burst AER 효과
   - 단일 pixel 지연 → latency
5. **L5. 라인 특성**: 256-pixel 행 / 열 라인의 extracted RC와 off-pixel 255개의 누설 합으로 pull-up 크기, 최악 rise/fall 시간, 오검출 margin을 산출한다 (corner: SS/FF, 온도).

**핵심 지표 (발표용)**: 최대 이벤트율 [Meps], 이벤트당 지연 [ns], pJ/event, 정적 전력 [µW], pixel pitch [µm], fill factor [%], baseline 대비 개선율.

## 4. 일정 (10/08 → 10/30, 약 3주)

| 기간 | 작업 |
|---|---|
| 10/08–10/11 | 아키텍처 확정(본 문서), pixel interface 회로도, arbiter DC cell, column latch 회로도, pixel pitch 목표 설정 |
| 10/12–10/15 | L1/L2 시뮬레이션, Verilog-A pixel 모델 작성 및 보정(L3), pixel / arbiter cell layout 착수 |
| 10/16–10/20 | Baseline / 개선 AER의 256×256 행위 시뮬레이션(L4)과 처리량 비교, column / scanner layout |
| 10/21–10/25 | Top array 조립, DRC/LVS, PEX, post-sim(L1, L2, L5), 전력 / 면적 집계 |
| 10/26–10/28 | 결과 분석, 그래프, 표 작성. 버퍼 |
| 10/29–10/30 | 10분 PPT, 녹화, 제출 |

**리스크와 대응**
- 시간 부족 → 개선 AER은 row-burst 하나에 집중한다. group address는 옵션이다.
- Top-level 시뮬레이션이 무거움 → Verilog-A / Verilog 비중을 늘린다. transistor-level 증거는 L1, L2, L5로 확보한다.
- Pixel 면적 증가 → interface 트랜지스터 수를 최소화하고, 소자 수 표를 [P] Table 1 형식으로 정리한다.

## 5. 팀 분담 (예시)
- A: pixel interface + reset + row request 보정, pixel layout, L1/L5
- B: arbiter tree + column latch/scanner, 주변회로 layout, L2
- C: Verilog-A 모델, 256×256 top 시뮬레이션, 처리량 / 전력 분석, 발표 자료
