# CLAUDE.md — 해저탐험 웹 게임

경훈의 개인용 '해저탐험(Deep Sea Adventure)' 웹 게임. 응답과 UI 문구는 한국어로 작성한다.

## 작업 방식

- 수정하면 먼저 `index.html` 파일을 경훈에게 보여 준다. **커밋·푸시는 경훈이 요청할 때만** 한다(아티팩트 갱신도 요청 시에만).
- 푸시할 때마다:
  - `index.html`의 `VERSION`을 0.1씩 올린다 (예: 1.0 → 1.1). 경훈이 버전을 지정하면 그 값을 쓴다.
  - `BUILD_DATE`를 푸시하는 날짜(`yyyy-mm-dd`, 한국 시간)로 바꾼다. 시작 화면 맨 아래 가운데에 `v버전 · 날짜`로 표시된다.
  - claude.ai 아티팩트 '해저탐험'도 같은 HTML로 갱신한다 (https://claude.ai/artifact/S658b8p1FUnjpu7nYDingy).
  - README.md의 버전 표에 한 줄 추가한다.

## 구조

- **단일 파일** `index.html` (HTML + CSS + JS). 빌드 도구 없음, GitHub Pages(`main`)로 배포.
- 외부 의존성은 GSAP 3.12.5(cdnjs) 하나뿐. Phaser 등 게임 엔진은 쓰지 않기로 결정함(가벼움 우선).
- 그래픽은 CSS/SVG로 직접 그림(원작 일러스트 사용 금지). 효과음은 Web Audio 합성(`SFX` 객체).

## 코드 지도 (index.html `<script>`)

- `S` — 전체 게임 상태: `players`, `path`(칸 배열: `{id, type:'chip'|'blank', chips:[{level,value}]}`), `air`, `round`, `start`, `cur`, `depleted`, `alarmShown`, `returnOrder`.
- 플레이어: `{idx, name, isAI, color, pos(0=잠수함), dir:'down'|'up', held:[stack], scored:[stack], returned}`. stack은 칩 배열(잃은 보물 묶음은 3개짜리 stack, 1개로 취급).
- 보드 배치: `CURVE` 설정 + `curvePts`(물결 가로줄 + 반원 U턴) → `spacePos(i)`가 곡선 길이 `STEP` 간격으로 좌표 반환. `worldH()`, `drawPath()`.
- 렌더: `layout()`(칸 DOM을 id 기준으로 재사용·애니메이션), `buildDivers()/updateDiver()`, `renderHUD()`(상단 산소 게이지 + 현재 플레이어 보유 보물 줄).
- 카메라: `#world`에 GSAP x/y/scale. `camOverview()`(전체 맞춤), `camHome()`(기본 배율 = 전체의 **150%**, 현재 플레이어 중심, 보드 밖 안 보이게 보정), `follow()`(줌 없이 화면 밖일 때만 이동), `setupPan()`(드래그·휠·두 손가락 핀치).
- 턴 흐름: `gameLoop()` → `playTurn(p)` → `oxygenStep` → 방향(`askDirection` / `aiDir`) → `rollDice` → `moveDiver` → 탐색(`digStep` / `dropStep` / `aiSearch`) → `endRound()` → `showFinal()`.
- 시작: `startGame()` → `rouletteOrder()`(1번만 룰렛, 나머지 랜덤, 시작 버튼). 차례 화면 이름 칸은 3명씩(4명은 2명씩) 줄 맞춤, 휴대폰 세로(≤560px)에서는 순서 목록 2단.
- 배경: `ambient()`(거품·작은 물고기) + `sealife()`(상어·바다거북 SVG `swimmer`).

## 확정된 규칙 해석 (바꾸지 말 것)

- 산소가 0 이하가 되어도 **그 턴은 끝까지 진행**하고, 턴이 끝나면 라운드 종료(나무위키 기준).
- 잠수함(pos 0)에 한 번도 나가지 않은 플레이어는 안전.
- 다음 라운드 선: 가장 깊이 남아 있던 잠수부, 모두 귀환했으면 마지막 귀환자.
- 잃은 보물은 섞어서 3개씩 묶어 path 끝에 추가, 빈 칸은 제거 후 당김.

## 사용자가 정한 연출 요구사항

- 줌인은 **보물 캘 때만**(기본 배율 대비 약 1.4배, 과하지 않게). 이동·산소 감소 때는 줌인하지 않음.
- 산소 감소 메시지는 **화면 중앙에 하나만**(`centerMsg`). 주사위 굴리기 버튼도 화면 중앙.
- "산소부족! 산소부족!"은 **한 줄**, 산소 5 이하가 될 때 **라운드당 한 번만** + 3초 사이렌.
- 플레이어 현황 카드는 표시하지 않음. 말에 ↓/↑ 방향 화살표와 보유 개수 배지.
- 턴 시작 배너·방향 선택 카드는 화면 크기에 맞춰 작게(clamp 사용). 주사위는 75px(처음 50px에서 1.5배로 키움).
- 캔 자리는 X가 아니라 **열린 빈 보물상자**.

## 테스트

- Playwright + 로컬 Chrome으로 전원 AI 3라운드 자동 진행해 `#final.show`까지 오류 없는지 확인.
- GSAP CDN이 막힌 환경에서는 `npm i gsap@3.12.5` 후 `page.route('**/gsap.min.js', ...)`로 로컬 파일 제공.
- 속도 단축: `setTimeout` 지연을 나누고 `gsap.globalTimeline.timeScale(n)` (evaluate에서 반환값으로 timeline을 돌려주면 직렬화로 멈추니 `; 0`을 붙일 것).
- 시작 시 룰렛 화면의 `#rStart` 버튼을 눌러야 게임이 시작됨.
