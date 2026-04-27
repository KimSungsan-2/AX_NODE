# 회사 LLM에 PROMPT.md 전달하는 방법

회사 보안망에서 GitHub Pages 사이트(`https://kimsungsan-2.github.io/AX_NODE/`)가 차단되어 작동하지 않을 때 대안.

## 옵션 A: PROMPT.md 한 번에 전달

가장 간단한 방법. 회사 LLM(예: 사내 ChatGPT, Claude, Gemini)에 다음과 같이 입력:

```
첨부한 PROMPT.md 사양에 따라 단일 HTML 파일을 만들어줘.
파일명: node_simulator.html
모든 인터랙션(노드 제거, AX 캐터필러 궤도 애니메이션, 필터, 모달)이 동작해야 함.
```

그리고 PROMPT.md 내용을 그대로 붙여넣기.

## 옵션 B: 출력 토큰 제한이 있는 LLM의 경우 (분할 요청)

회사 LLM이 한 번에 긴 HTML 출력을 못 하면 단계별로 요청:

### 1차 요청 — 골격
```
첨부한 PROMPT.md 사양에 따라 HTML 골격, CSS, vis-network 로딩 fallback,
부팅 함수까지만 작성해줘. JavaScript의 데이터(initialNodes/initialEdges)와
인터랙션 함수는 placeholder로 두면 돼.
```

### 2차 요청 — 데이터
```
PROMPT.md의 §7과 §8 가이드에 따라 142개 노드와 240개 엣지를 생성해줘.
JS 배열 형태로:
const initialNodes = [...];
const initialEdges = [...];
```

### 3차 요청 — 인터랙션
```
PROMPT.md §9 인터랙션과 §10 AX 캐터필러 트랙 알고리즘을 구현해줘.
nodeToVis, edgeToVis, removeNode, applyAXToSelection, drawAXTracks,
drawPairTrack, drawWheelOutline 함수 포함.
```

### 4차 요청 — 합치기
```
1~3차 출력을 모두 결합한 단일 node_simulator.html 파일로 만들어줘.
```

## 옵션 C: 기존 HTML 파일 복사

가장 빠름. `node_simulator.html` 파일 자체를 메일 첨부나 USB로 회사로 가져가서 더블클릭으로 실행.

회사 PC에서 vis-network CDN 자체가 차단되면:
1. 집에서 `https://cdn.jsdelivr.net/npm/vis-network@9.1.9/standalone/umd/vis-network.min.js` 다운로드
2. HTML 파일 폴더에 `vis-network.min.js`로 저장
3. HTML 상단의 fallback 로더 부분을 제거하고 `<script src="vis-network.min.js"></script>` 한 줄로 교체
4. 폴더 통째로 회사 PC로 복사

## 옵션 D: 사내 SharePoint/위키에 올리기

HTML 파일이 사내 SharePoint나 Confluence 같은 곳에 첨부될 수 있다면 그곳에 업로드. 사내망에서 접속 가능.

## 트러블슈팅

### LLM이 데이터를 다 안 만들어주고 "..." 으로 생략하면
다음 프롬프트로 강제:
```
초기 데이터에 ... 으로 생략하지 말고 142개 노드와 240개 엣지를 모두 풀어서 작성해줘.
```

### 그래도 안 되면
PHASE별로 나눠서 요청:
```
PROMPT.md §7의 PHASE 1 (소비자 인사이트) 14개 노드와 그 사이의 엣지만 먼저 만들어줘.
```

### 캐터필러 트랙 애니메이션이 작동 안 하면
PROMPT.md §10의 의사코드를 그대로 복사해서:
```
이 의사코드를 정확히 그대로 구현해줘. R = Math.max(r1, r2),
top = theta - π/2, bot = theta + π/2 등 모든 수식을 그대로 유지.
```

## 검증 체크리스트

생성된 HTML을 더블클릭으로 열어서 다음을 확인:

- [ ] 그래프가 그려지는가? (142개 노드 보임)
- [ ] 노드 클릭으로 누적 선택되는가?
- [ ] 노드 제거 시 우측 절감 카운터가 올라가는가?
- [ ] 두 노드 선택 후 ★ AX 적용 → 캐터필러 궤도가 회전하는가?
- [ ] 좌측 부서 필터의 직군 그룹이 접히는가?
- [ ] 절감 효과 패널 헤더 클릭으로 접히는가?
- [ ] 결과 클립보드 복사 시 Excel에 붙여넣기 가능한가?

문제가 있으면 PROMPT.md §15 (완성 기준)와 §16 (흔한 실수)을 참조해서 LLM에 재요청.
