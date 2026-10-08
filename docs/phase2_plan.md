# Analog 2차 과제 프로젝트 방향 (256×256 DVS Array + AER Readout)

> 참고 논문: de Oliveira et al., "Asynchronous time-based imager with DVS sharing", AICSP 2021 (이하 [P])
> 마감: **10/30(금)** — 회로도, Layout, Post-Sim 결과, 기능 점검 결과, 면적, 전력소모, 설계 분석 자료, 10분 발표 녹화

## 0. 현재 위치

| 과제 | 내용 | [P] 대응 | 상태 |
|---|---|---|---|
| 1차 | DVS pixel (photoreceptor + differencing amp + ON/OFF comparator) + 가속 버퍼(Mc1~Mc4) | Fig. 1, Fig. 3a | 완료 (gpdk045, Virtuoso, schematic/layout/PEX) |
| 1차 미완 | 4단계 handshake 중 **2~4단계** (Ma1~Ma3, Mb1p/Mb1n, Mb2~Mb4, Mar/Mac/Mref_r) | Fig. 2, 3b | **2차 첫 작업** |
| 2차-① | 256×256 pixel array 설계 | Fig. 10 (matrix 구성) | **이번 과제** |
| 2차-② | Readout 효율 향상 AER 설계 및 적용 | Fig. 2, 3, 6, 7 (+ 개선) | **이번 과제** |

- [P]의 Capture module(Fig. 4, 5, ATIS 휘도 측정)과 DVS sharing은 **범위 밖**으로 둔다. 과제가 DVS + AER이고, 마감까지 3주뿐이다.
- [P]에서 가져올 것: ① pixel↔arbiter handshake 회로 (Fig. 2, 3), ② binary-tree arbiter (Fig. 6, 7), ③ array/주변회로 연결 구조 (Fig. 10), ④ **Verilog-A 행위 모델로 대형 array를 검증하는 방법론** (Sec. 5, 6).
- 차별화 포인트: [P]의 AER은 이벤트 1개당 row arb → col arb 왕복을 하는 **고전적 AER**이다. 이것을 baseline으로 두고, 처리량을 높인 **개선 AER**과 정량 비교하는 것을 발표의 축으로 삼는다. 과제 문구("Readout 효율 향상을 위한 AER")에 그대로 대응한다.

## 0.3 대회 요구사항 대응표 (1차 미흡 항목 포함)

최종 제출(10/30)은 1·2차 요구사항을 모두 포함해 평가되므로, 1차 보고서에서 정량 근거가 없던 항목을 2차 작업에 함께 넣는다.

| 요구사항 | 1차 보고서 상태 | 2차에서 할 일 (정량 지표) |
|---|---|---|
| 1차: DVS pixel 설계 | ✅ schematic/layout/PEX | STEP 1 handshake 완성 |
| 1차: 잡음 최소화 | △ 가속 버퍼로 라인 누설에 의한 오트리거만 다룸 | **N1** Spectre transient noise로 Vdiff 노이즈 rms 측정 (수광부 shot/열잡음, SF 대역폭). **N2** 일정 조도에서 오이벤트율(background activity, events/pixel/s) 측정. **N3** Mr 리셋 kTC / charge injection 오프셋 측정. **N4** 가속 버퍼 on/off 오트리거 비교. 개선 수단: Vbias_sf로 대역 제한, C2 크기, Mr 크기 / dummy |
| 1차: Minimum sensitivity threshold 최소화 | ✗ 수치 없음 | **T1** 이론: θ_ON ≈ exp((Vdiff0−V_ON)/(n·U_T·C1/C2))−1. **T2** 시뮬레이션: Iph 1 pA/100 pA/10 nA에서 계단 ΔIph/Iph를 sweep → 이벤트가 나는 최소 대비(%). **T3** Monte Carlo(mismatch)로 threshold σ (FPN). **결론**: θ_min = max(노이즈 기준 ~5σ_noise, mismatch 3σ) → 이 값까지 Vbias_ON/OFF를 좁힌 설정 제시 |
| 1차: In-pixel event memory | △ 명시적 구조 없음 (C2 전압과 요청 유지에 의존) | **M1** 비교기 출력 뒤 2-bit SR latch (ON/OFF) 추가: VON/VOFF로 set, Vreset(ACK_ROW·ACK_COL)으로 clear → arbiter 대기 중에 Vdiff가 변하거나 누설돼도 이벤트 / 극성 보존. **M2** 래치 없음 vs 있음: ACK 지연 sweep 시 이벤트 손실률 비교 |
| 2차: 256×256 array | ✗ | STEP 2 (타일링 layout), STEP 7 (top 조립, DRC/LVS), STEP 5 (full-scale 검증) |
| 2차: AER 방식 설계 | ✗ | STEP 1 (pixel 측 4-phase handshake), STEP 3 (라인 인터페이스, DC, 8단 tree, 인코더) = baseline AER |
| 2차: Readout 효율 향상 AER 적용 | ✗ | STEP 6: Row-burst(word-serial) AER 또는 greedy arbiter. baseline 대비 처리량(Meps), 지연(ns), pJ/event, 공정성 비교 |

## 0.5 실행 단계 (1차 보고서 기준, 이 순서대로 설계 시작)

### STEP 1. Pixel handshake 완성 ([P] Fig. 2a, 2b, 3b) — 가장 먼저
1차 보고서 기준으로 Row Request(1단계)만 동작하고, 2~4단계는 "Arbiter 구현 이후"로 미뤄져 있다. Arbiter를 기다리지 않고, **ideal 4-phase responder(Verilog-A)**를 테스트벤치에 붙여 pixel 단독으로 먼저 완성한다.
- ON 경로 (Fig. 2a): `VON` → Ma1/Ma2 → `Vreq_row'` pull-down. `Vack_row` 수신 시 Ma3 → `VreqON_col` pull-down.
- OFF 경로 (Fig. 2b): `VOFF`는 OFF일 때 Low로 가므로 Mb1p/Mb1n 인버터로 반전한 뒤 Mb2~Mb4로 `Vreq_row'`, `VreqOFF_col`을 구동한다.
- Reset (Fig. 3b): `Mref_r` pull-up(`Vref_reset`)과 직렬 NMOS `Mar(ack_row)`·`Mac(ack_col)`로 `Vreset` → Low를 만들고, PMOS Mr이 C2를 방전한다.
- 검증: 1 pA→10 nA→1 pA 램프(5 ms)에서 ON/OFF 이벤트 수, req→ack→reset 순서, reset 이후 req 해제를 확인한다. 응답지연을 sweep(10 ns~10 µs)해서, ack가 늦어도 Vdiff 누설로 오동작하지 않는지 본다 (1차 보고서 3장 4번째 항목).
- 산출물: `dvs_pixel` 심볼 (핀: `PD, Vbias_*, Vion, Vioff, Vref_reset, REQ_ROW, ACK_ROW, REQ_ON_COL, REQ_OFF_COL, ACK_COL`)

- **STEP 1에 추가 (요구사항 대응)**: M1 in-pixel latch를 같이 넣는다 (latch 출력이 Ma1/Mb2를 구동하도록). 회로 동결 전에 N1~N3, T1~T3 시뮬레이션을 수행한다.

### STEP 2. Pixel 면적 축소 + 타일링 가능한 layout 재설계
1차 layout은 **C1/C2 unit cap 배열이 대부분의 면적**을 차지한다. 이대로 256×256으로 늘리면 die가 비현실적으로 커진다. Array 조립 전에 반드시 정리해야 한다.
- C1/C2 비율(Ad)과 unit cap 크기를 재검토한다. 총 C를 줄이고, MOM cap은 상위 metal로 올려 트랜지스터 위에 겹쳐 배치한다.
- 목표 pitch를 정하고(예: 정사각 셀), photodiode 면적과 fill factor를 함께 보고한다.
- **Abutment 규칙**: VDD/VSS와 bias 라인은 가로/세로 관통, `REQ_ROW/ACK_ROW`는 가로 metal, `REQ_*_COL/ACK_COL`은 세로 metal로 셀 경계에서 그대로 이어지게 한다.
- 이 단계에서 한 셀의 라인 RC(가로·세로)를 PEX로 뽑아둔다. ×256 하면 행·열 라인 부하가 된다 (STEP 4, 6에서 사용).

### STEP 3. 주변 회로 셀 설계 ([P] Fig. 6, 7)
1. **라인 인터페이스 (Fig. 7b)**: 행·열 라인마다 pull-up(`Vpu` bias) + 인버터 → arbiter 입력 `Req_i`. 256 pixel 누설 합과 라인 C를 기준으로 pull-up 크기를 정한다.
2. **Decision Cell (Fig. 6b + 7a)**: 교차 NAND SR latch(Req_1', Req_2')로 승자를 고르고, `Req_Out = NAND(Req_1', Req_2')`, `Ack_Out1/2 = Ack_In · (선택 쪽)`로 만든다. 2입력 동시 요청, 한쪽 유지 중 다른 쪽 도착, Ack_In off 등 모든 경우를 transistor-level로 검증한다.
3. **8단 binary tree (256 입력)**: DC 255개로 구성하고, 최상단은 `Req_Out → Ack_In` 직결이다 ([P] 동일). 행 arbiter 1개, 열 arbiter 1개를 만든다. 열은 `REQ_ON_COL | REQ_OFF_COL`을 OR해서 arbitration하고, 승자 열에서 어느 라인이 Low인지로 polarity bit를 정한다.
4. **Address encoder**: one-hot ack(256) → 8-bit 주소. ROM형 NOR encoder로 만들고, `{Y[7:0], X[7:0], POL}` 출력에 외부 `REQ/ACK`를 둔다.

### STEP 4. 소형 array 통합 검증 (transistor-level)
- 2×2 → 4×4 → 8×8 (8×8에서는 arbiter를 3단으로 잘라서 사용).
- 시나리오: (a) 한 pixel 램프 → 주소 일치, (b) 모든 pixel 동일 램프 → 충돌 해결 / 누락 없음, (c) 1 pixel만 상수 + 나머지 램프 ([P] Fig. 12/16 대응), (d) 가속 버퍼 off/on 비교로 오트리거 방지 효과를 정량화한다 (1차 주장 검증).
- 출력 주소 stream을 CSV로 저장 → Python으로 [P]식 재구성 파형과 RMSE.

### STEP 5. 256×256 검증 (behavioral + transistor 혼합)
65,536개 pixel을 Verilog-A로 넣으면 Spectre로도 무겁다. 아래 두 축으로 나눈다.
- **5a. 주변 회로 full-scale**: 행·열 arbiter(256 입력 transistor-level)와 STEP 2의 라인 RC ×256을 함께 넣는다. pixel은 "요청 생성기" Verilog-A(이벤트 시각 리스트 → req, ack 받으면 해제)로 대체한다. 최악 지연, 최대 처리량(events/s), 공정성을 측정한다.
- **5b. Pixel 행위 모델**: [P] Listing 1–5 구조(log 차분, ON/OFF cross, row→col req, ack 후 reset, **1–5 ns 랜덤 지연**)로 만들고, STEP 1 결과와 이벤트 수 / 타이밍 오차를 표로 비교한다 ([P] Table 3).
- (가능하면) 5c: 영상 입력으로 Python 시스템 레벨 시뮬레이션 → 이벤트율 분포를 5a의 자극으로 사용한다.

### STEP 6. AER 효율 개선 (차별화) — STEP 3~5의 baseline이 동작한 뒤 착수
- Baseline 처리량을 5a로 측정한 뒤 **Row-burst AER**(1.2절)을 적용하고 같은 자극으로 비교한다.
- 시간이 부족하면 최소안: **Greedy arbiter**(같은 subtree 연속 승인) + ON/OFF 동시 처리. 이것만으로도 처리량 / 지연 개선을 수치로 제시할 수 있다.

### STEP 7. Top layout + Post-sim + 정리
- 256×256 array(mosaic) + 행 arbiter(좌) + 열 arbiter/encoder(상/하) + bias / IO 핀을 배치하고, DRC/LVS를 돌린다.
- Post-sim: pixel, DC, 라인 인터페이스는 PEX. Top은 STEP 5에 extracted 라인 RC를 반영한다.
- 면적 표(pixel, 주변, 전체), 전력(정적: pixel bias ×65,536 + 주변, 동적: pJ/event), 소자 수 표 ([P] Table 1 형식).

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

| 기간 | STEP | 작업 |
|---|---|---|
| 10/08–10/11 | 1, 3-1 | pixel handshake 2~4단계 완성(ideal responder), 라인 인터페이스 회로 |
| 10/09–10/14 | 2 | pixel 면적 축소 + 타일링 layout, 셀 PEX (STEP 1과 병행) |
| 10/10–10/15 | 3-2~4 | Decision cell, 256-tree, encoder 회로도 + 단위 검증 |
| 10/15–10/19 | 4, 5b | 2×2/4×4/8×8 통합 시뮬레이션, Verilog-A pixel 모델 |
| 10/18–10/22 | 5a, 6 | 256 full-scale 주변 회로 시뮬레이션, baseline vs 개선 AER |
| 10/21–10/26 | 7 | 주변 layout, top 조립, DRC/LVS/PEX, post-sim, 면적/전력 |
| 10/27–10/30 | — | 분석 정리, 10분 PPT, 녹화, 제출 |

**리스크와 대응**
- 시간 부족 → 개선 AER은 row-burst 하나에 집중한다. group address는 옵션이다.
- Top-level 시뮬레이션이 무거움 → Verilog-A / Verilog 비중을 늘린다. transistor-level 증거는 L1, L2, L5로 확보한다.
- Pixel 면적 증가 → interface 트랜지스터 수를 최소화하고, 소자 수 표를 [P] Table 1 형식으로 정리한다.

## 5. 팀 분담 (예시)
- A: STEP 1 → STEP 2 (pixel handshake 완성, 면적 축소 layout, 셀 PEX)
- B: STEP 3 → STEP 7 주변부 (라인 인터페이스, DC, tree, encoder 회로도 / layout)
- C: STEP 5 → STEP 6 (Verilog-A 모델, 256 full-scale 시뮬레이션, AER 개선 비교, 발표 자료)
- STEP 4는 A+B 합류, STEP 7 top 조립은 전원
