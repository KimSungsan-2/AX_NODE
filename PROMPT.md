# DA사업부 업무 노드 시뮬레이터 — 구현 프롬프트

> **이 문서를 LLM에 그대로 전달하면 단일 HTML 파일로 동일한 시뮬레이터를 만들 수 있습니다.**

---

## 0. 최상위 지시 (LLM에게)

다음 사양에 따라 **단일 HTML 파일** (`node_simulator.html`)을 생성하라.
파일 한 개에 HTML, CSS, JavaScript가 모두 포함되어야 하고, **외부 CDN 두 개**(vis-network, Google Fonts Nanum Gothic) 이외의 의존성은 없어야 한다. 빌드 도구·서버·프레임워크 사용 금지. 사용자가 더블클릭으로 브라우저에서 바로 열 수 있어야 한다.

---

## 1. 프로젝트 목적

삼성전자 DA(Digital Appliances)사업부의 업무 프로세스를 노드-엣지 그래프로 시각화하고, 노드(업무 단위)를 클릭으로 제거했을 때 절감되는 시간/비용/FTE를 실시간 계산하는 인터랙티브 웹앱.

핵심 메타포:
- **노드 = 업무 단위** (예: "8D 보고서 작성", "PSI 데이터 추출")
- **엣지 = 두 업무 간 핸드오프** (예: "불량 데이터 전달")
- **엣지 길이 = 병목 크기** (연간 처리 시간 h에 비례)
- **노드 제거 = 그 업무를 자동화/제거하면 얼마나 절감되는지 시뮬레이션**
- **AX 통합 = 두 노드를 캐터필러 궤도로 묶어 자동화 단위로 표시**

---

## 2. 기술 스택 (절대 변경 금지)

| 항목 | 값 |
|---|---|
| 형태 | 단일 HTML (`<!DOCTYPE html>` ~ `</html>` 한 파일) |
| 그래프 라이브러리 | vis-network 9.1.9 (CDN: jsdelivr → cdnjs → unpkg fallback) |
| 폰트 | Nanum Gothic 400/700/800 (Google Fonts) |
| 상태 관리 | Vanilla JS, 전역 `state` 객체 |
| 시간당 비용 기준 | 60,000원/h |
| FTE 기준 | 2,080 h/년 |
| 액센트 색 | 삼성 블루 `#1428A0` |

CDN fallback 로딩은 다음 순서로 시도하고, 모두 실패 시 화면 가득 안내 박스 표시(파일 다운로드 링크 포함).

```js
const sources = [
  'https://cdn.jsdelivr.net/npm/vis-network@9.1.9/standalone/umd/vis-network.min.js',
  'https://cdnjs.cloudflare.com/ajax/libs/vis-network/9.1.9/standalone/umd/vis-network.min.js',
  'https://unpkg.com/vis-network@9.1.9/standalone/umd/vis-network.min.js'
];
```

---

## 3. 레이아웃

CSS Grid 3컬럼 + 헤더:

```
┌─────────────────────────────────────────────────────────┐
│ HEADER (56px, #1428A0 배경, 흰 글씨)                     │
├──────────┬─────────────────────────────┬────────────────┤
│ LEFT     │  STATS BANNER (4 stats)     │  RIGHT         │
│ 240px    ├─────────────────────────────┤  300px         │
│          │  GRAPH (vis-network 캔버스) │                │
│ 부서필터 │                             │  절감 효과     │
│ 직군필터 │  toolbar (우상단)           │  - 시간 카드   │
│ 엣지타입 │  legend (좌하단)            │  - 비용 카드   │
│ 옵션     │                             │  - FTE 카드    │
│          │                             │  제거 이력     │
│          │                             │  버튼들        │
└──────────┴─────────────────────────────┴────────────────┘
```

CSS:
```css
.app { display: grid; grid-template-columns: 240px 1fr 300px; grid-template-rows: 56px 1fr; height: 100vh; }
.graph-wrap { display: flex; flex-direction: column; min-height: 0; overflow: hidden; }
#graph { flex: 1 1 auto; min-height: 400px; }
```
**중요**: 그래프 영역이 0 높이로 접히지 않도록 flex column + min-height 필수.

---

## 4. 색상 체계

**직군 색상 (노드 배경)**
```js
const JOB_COLORS = {
  '연구개발':    '#1428A0',  // 삼성 블루
  '생산·품질':   '#2E7D32',  // 딥 그린
  '구매·SCM':    '#E65100',  // 딥 오렌지
  '영업·마케팅': '#C62828',  // 딥 레드
  '기획·전략':   '#4A148C',  // 딥 퍼플
  '서비스·CS':   '#00838F',  // 딥 시안
  '인사·총무':   '#F9A825',  // 앰버
  'IT·보안':     '#37474F',  // 블루그레이
  '법무':        '#4E342E'   // 브라운
};
```

**엣지 색상 (전달 유형별)**
```js
const EDGE_COLORS = {
  '수동전달': '#EF9A9A',  // 연한 빨강 (Pain Point)
  '자동전달': '#A5D6A7',  // 연한 초록
  '구두보고': '#FFE082'   // 연한 노랑
};
```
`수동전달` 중에서도 hours가 60↑은 `#EF5350`, 100↑은 `#E53935`로 진하게 표시.

**AX 통합 (캐터필러 궤도)**
- 궤도 본체: `rgba(20, 40, 160, 0.88)` — 삼성블루
- 트레드(움직이는 점선): `#FFB300` — 앰버
- 회전 스포크: `rgba(20, 40, 160, 0.45)`
- 허브: `#FFB300`

---

## 5. 데이터 스키마

```js
// 노드
{
  id: string,            // 'C01', 'D04', 'S03' 등
  label: string,         // 줄바꿈 \n 포함 가능
  dept: string,          // 부서명 (예: '품질보증팀')
  persona: string,       // 가상의 페르소나명 (예: '이품질')
  job_type: string,      // JOB_COLORS의 9개 키 중 하나
  hours_per_year: number,
  cost_per_year: number, // = hours_per_year × 60000 (자동계산)
  system: string,        // 사용 시스템 (예: 'QMS, MES')
  ax_candidate: boolean  // true면 노드 테두리 노란색 (#FFD600)
}

// 엣지
{
  id: string,            // 'e001' 등
  from: string,          // 노드 id
  to: string,            // 노드 id
  label: string,         // (예: '불량 데이터 전달')
  hours: number,         // 핸드오프에 드는 연간 시간
  transfer_type: string  // '수동전달' | '자동전달' | '구두보고'
}
```

---

## 6. 부서 카탈로그 (DEPT_CATALOG)

직군별로 그룹화된 65개 부서. 노드 추가 모달의 datalist와 좌측 부서 필터에 사용.

```js
const DEPT_CATALOG = {
  '연구개발': ['선행개발팀', '제품개발1팀', '제품개발2팀', '제품개발3팀',
              'HW설계팀', 'SW개발팀', '펌웨어개발팀', '플랫폼개발팀',
              'UX개발팀', '디자인팀', '시험인증팀', '특허기술팀'],
  '생산·품질': ['제조1팀', '제조2팀', '제조혁신팀', '생산기술팀',
              '품질보증팀', '품질혁신팀', '신뢰성평가팀', '필드품질팀',
              '안전환경팀', '설비기술팀'],
  '구매·SCM': ['구매팀', '전략구매팀', 'SCM기획팀', '수급관리팀',
              '자재팀', '물류팀', '수출입팀', '협력사관리팀'],
  '영업·마케팅': ['국내영업팀', '해외영업팀(북미)', '해외영업팀(유럽)', '해외영업팀(아시아)',
                '영업기획팀', '채널영업팀', '온라인영업팀', 'B2B영업팀',
                '상품기획팀', '브랜드마케팅팀', '디지털마케팅팀', '콘텐츠팀'],
  '기획·전략': ['사업기획팀', '전략기획팀', '경영전략팀',
              '재무팀', '경영관리팀', '원가기획팀', '투자기획팀'],
  '서비스·CS': ['서비스기획팀', 'CS운영팀', 'AS기술팀',
              '콜센터운영', '서비스품질팀', '리퍼/RMA팀'],
  '인사·총무': ['인사기획팀', 'HRD교육팀', '노사협력팀', '조직문화팀',
              '총무팀', '시설관리팀'],
  'IT·보안': ['IT인프라팀', '시스템운영팀', 'DX기획팀',
              '데이터분석팀', '정보보안팀', 'AX추진사무국'],
  '법무': ['법무팀', '컴플라이언스팀', 'IP기획팀', '계약심사팀']
};
```

---

## 7. 초기 노드 데이터 (142개, 10단계 사이클형)

전체 라이프사이클: **소비자 인사이트 → POD/컨셉 → 개발 → 시험 → 양산준비 → 본양산 → 물류·영업 → 서비스 → 분석 → 백오피스 → (사이클 클로즈) → 다시 소비자 인사이트**

ID 명명 규칙:
- `C01-C14`: 소비자 인사이트
- `P01-P13`: POD/컨셉
- `D01-D24`: 상품기획·개발
- `T01-T12`: 시험·인증
- `M01-M14`: 양산 준비
- `F01-F13`: 본 양산
- `L01-L16`: 물류·영업
- `S01-S14`: 서비스·CS
- `A01-A13`: 분석·피드백
- `B01-B14`: 백오피스 (Cross-Cutting)

각 단계별 노드 시간(hours_per_year) 가이드:
- 단순 자동/짧은 작업: 80~200h
- 중간 업무: 200~400h
- 핵심 업무 / 페인 포인트: 400~700h
- 메가 페인 (양산 운영, AS, 8D 등): 500~720h

**노드 작성 예시 (각 단계마다 10~15개 생성)**

```js
// PHASE 1: 소비자 인사이트 (14개)
{ id: 'C01', label: '글로벌 시장\n트렌드 조사',   dept: '영업기획팀',     persona: '최영업', job_type: '영업·마케팅', hours_per_year: 280, system: 'NielsenIQ, GfK', ax_candidate: true },
{ id: 'C02', label: '고객 VOC\n수집·정제',       dept: 'CS운영팀',       persona: '오CS',   job_type: '서비스·CS',   hours_per_year: 416, system: 'VOC시스템',     ax_candidate: true },
{ id: 'C03', label: '채널 POS\n판매 데이터 수집', dept: '영업기획팀',     persona: '최영업', job_type: '영업·마케팅', hours_per_year: 312, system: 'GSCM, POS',     ax_candidate: true },
{ id: 'C04', label: '경쟁사\n벤치마킹',         dept: '상품기획팀',     persona: '한기획', job_type: '영업·마케팅', hours_per_year: 208, system: 'PPT, 리포트',    ax_candidate: false },
{ id: 'C05', label: '소셜/리뷰\n빅데이터 분석', dept: '디지털마케팅팀', persona: '윤마케', job_type: '영업·마케팅', hours_per_year: 364, system: 'Brandwatch, GA',  ax_candidate: true },
{ id: 'C06', label: '카테고리\n매트릭스 분석', dept: '상품기획팀',     persona: '한기획', job_type: '영업·마케팅', hours_per_year: 156, system: 'Excel, BI',      ax_candidate: true },
{ id: 'C07', label: '사용 패턴\n에스노그라피', dept: 'UX개발팀',       persona: '예UX',   job_type: '연구개발',    hours_per_year: 240, system: '필드조사툴',    ax_candidate: false },
{ id: 'C08', label: '페르소나\n카드 작성',     dept: 'UX개발팀',       persona: '예UX',   job_type: '연구개발',    hours_per_year: 100, system: 'Figma',         ax_candidate: false },
{ id: 'C09', label: '글로벌 가전\n리포트 정리', dept: '해외영업팀(북미)', persona: '제해외', job_type: '영업·마케팅', hours_per_year: 156, system: 'PPT',          ax_candidate: true },
{ id: 'C10', label: '라이프스타일\n트렌드 큐레이션', dept: '디지털마케팅팀', persona: '윤마케', job_type: '영업·마케팅', hours_per_year: 180, system: 'Pinterest, IG', ax_candidate: true },
{ id: 'C11', label: '옴니채널\n고객여정 매핑', dept: 'UX개발팀',       persona: '예UX',   job_type: '연구개발',    hours_per_year: 200, system: 'Miro',         ax_candidate: false },
{ id: 'C12', label: '산업 컨퍼런스\n참가/리포팅', dept: '선행개발팀',  persona: '서선행', job_type: '연구개발',    hours_per_year: 120, system: 'PPT',          ax_candidate: false },
{ id: 'C13', label: '심층 사용자\n인터뷰',     dept: 'UX개발팀',       persona: '예UX',   job_type: '연구개발',    hours_per_year: 280, system: '녹취록툴',     ax_candidate: false },
{ id: 'C14', label: '산업 리포트\n구독/번역', dept: '상품기획팀',     persona: '한기획', job_type: '영업·마케팅', hours_per_year: 156, system: 'DeepL',        ax_candidate: true },
```

> **다음 단계의 노드는 위 패턴을 따라 LLM이 직접 생성하라.** 각 단계별 핵심 업무는:
>
> **PHASE 2 POD/컨셉 (13개)**: POD 도출 워크샵, 컨셉 검증 FGI, 신제품 RFP 작성, 사업성 검토(NPV/IRR), 컨셉 임원 승인, 1차 검토 미팅, 디자인 시안 검토, 가격 포지셔닝 검토, 키비주얼 제작, ROI 시나리오 산출, 출시 타임라인 작성, 위험 평가, 마케팅 메시지 초안
>
> **PHASE 3 개발 (24개)**: 상품기획서, 개발 일정, 외관/UX 디자인, HW 설계, SW 개발, 펌웨어 개발, 시제품 제작, 설계 검증 DR, 회로 설계 PCB, 기구/구조 설계, IoT/연동 프로토콜, 음향/모터 튜닝, 부품 사양서, DFM/DFA 검토, 회로 시뮬레이션, PCB 레이아웃, 사출 금형 설계, SW 아키텍처, SW 단위테스트, SW 통합테스트, AI/ML 모델 개발, 시제품 안전성 검토, 사용자 매뉴얼, IP 침해 리스크 검토
>
> **PHASE 4 시험·인증 (12개)**: 신뢰성 평가, 안전/환경 인증, 글로벌 규격 테스트(UL/KC/CE), EMC/EMI 시험, 시험결과 리뷰, 가속수명(HALT), 환경 시뮬, 음향/소음 시험, 기능 검증, ESD/낙뢰 시험, 분해 실험, 사용자 베타 테스트
>
> **PHASE 5 양산 준비 (14개)**: BOM 확정, MRP, 협력사 발주, 라인 셋업, 작업표준서, 초도 양산(PP), 협력사 품질 감사, 금형 발주, 작업자 교육, SOP 등록, 지그 설계, 검사 장비 자동화, 양산 시뮬레이션, 패키징/라벨 디자인
>
> **PHASE 6 본 양산 (13개)**: 자재 IQC, 본 양산 운영(720h), 공정 품질 모니터링, 완제품 OQC, 출하 검사, SPC 통계, 공정 변경 ECN, 일일 일보, 야간조 인수인계, 주간 품질 회의, 라인 정지/리커버리, 환경/안전 점검, 자재 재고 회전 분석
>
> **PHASE 7 물류·영업 (16개)**: 출고/물류, 채널 배송, 가격 정책, 마케팅 캠페인, 채널 영업, 온라인 판매, PSI 주간 계획, 광고 소재 제작, 인플루언서 운영, 채널 인센티브 정산, 매장 진열/POP, 영업사원 교육, 수출 통관, 신제품 PRESS, B2B 거래처 운영, 채널별 재고 모니터링
>
> **PHASE 8 서비스·CS (14개)**: 고객 VOC 접수, AS 처리(624h), 필드 불량 원인 분석, 8D 보고서 작성, GQP 등록, 시정조치 확인, 콜센터 인입 분석, 출장 AS 일정, 부품/RMA, 챗봇 운영, 보증/리퍼 정책, 만족도 조사, 부품 재고 예측, AS 기사 스케줄링
>
> **PHASE 9 분석·피드백 (13개)**: 판매 데이터 분석, 매출/손익표, 사업부 검토 회의, 임원 보고, 차기 트렌드 인사이트, KPI 대시보드, 모델별 수익성, ESG 보고, 일일 영업 마감, 주간 동향 보고, 재고일 분석, 마진 압박 시뮬, 포스트모템
>
> **PHASE 10 백오피스 (14개)**: 인사 발령, OJT, 협력사 계약 심사, IP 출원 검토, 제조원가, 분기 결산, ERP 마스터 점검, 데이터 거버넌스, 채용 운영, 노사 협의, 보안 교육, 리더십 코칭, 컴플라이언스 점검, 시설 관리

---

## 8. 초기 엣지 데이터 (240+ 개)

각 phase 내부의 순차 흐름 + phase 간 핸드오프 + 사이클 클로즈 피드백.

**핵심 페인 포인트 엣지 (반드시 포함)**

```js
{ id: 'e073', from: 'T05', to: 'D08', label: 'NG → 재설계 루프',    hours: 208, transfer_type: '수동전달' },
{ id: 'e160', from: 'S03', to: 'S04', label: '원인 → 8D',           hours: 208, transfer_type: '수동전달' },
{ id: 'e190', from: 'A05', to: 'C01', label: '◆ 차기 트렌드 인풋',  hours: 208, transfer_type: '수동전달' },
{ id: 'e011', from: 'C02', to: 'P01', label: 'VOC 정리본',          hours: 156, transfer_type: '수동전달' },
{ id: 'e150', from: 'L05', to: 'S01', label: '판매처 VOC',          hours: 156, transfer_type: '수동전달' },
{ id: 'e036', from: 'D03', to: 'D04', label: '디자인-HW 연동',      hours: 156, transfer_type: '수동전달' },
{ id: 'e165', from: 'S04', to: 'D08', label: '필드불량 → 재설계',   hours: 156, transfer_type: '수동전달' },
```

**사이클 클로즈 엣지 (◆ 표시, 반드시 포함)**
```js
{ from: 'A05', to: 'C01' },  // 차기 트렌드 → 시장조사
{ from: 'A05', to: 'C04' },  // 경쟁 재정의
{ from: 'A05', to: 'C06' },  // 카테고리 재정의
{ from: 'A04', to: 'P01' },  // 차기 POD 시드
{ from: 'A01', to: 'C03' },  // 데이터 재활용
{ from: 'S06', to: 'C02' },  // VOC 라이프사이클
{ from: 'S12', to: 'C02' },  // 만족도 → VOC
{ from: 'S12', to: 'P02' },  // 만족도 → FGI
{ from: 'A11', to: 'M02' },  // 재고일 → MRP
{ from: 'F11', to: 'D14' },  // 정지원인 → DFM
{ from: 'A13', to: 'C13' },  // 사후평가 → 인터뷰
```

엣지 hours 분포 가이드:
- 자동 짧은 핸드오프: 26~52h
- 수동 핸드오프: 52~104h
- 페인 포인트: 104~208h

각 노드는 평균 3~5개의 incoming/outgoing 엣지를 가져야 한다 (총 240~260개).

---

## 9. UI 인터랙션 명세

### 9.1 그래프 영역
- vis-network로 렌더링
- 노드 hover → 툴팁 (업무명/부서/담당자/시간/비용/시스템)
- 노드 클릭 → **누적 토글 선택** (이미 선택되어 있으면 해제)
- 빈 캔버스 클릭 → 전체 선택 해제
- 노드 우클릭 → 컨텍스트 메뉴 표시:
  - "이 노드 제거" (빨강)
  - "연결 노드 모두 선택"
  - "노드 편집"
  - "★ AX 적용 / 해제"
  - "상세 정보"

### 9.2 노드 제거 애니메이션
- 0.4초 동안 scale 1→0 (cubic ease-out)
- 연결 엣지 opacity 1→0
- 완료 후 vis-network DataSet에서 실제 제거 → 물리 시뮬레이션 재안정화 → 우측 카운터 카운트업

### 9.3 우측 패널 카운터
3개 카드 (그라데이션 배경, 카운트업 애니메이션 0.5s):
- **연간 절감 시간**: 주황 그라데이션 (`#E65100` → `#BF360C`)
- **연간 절감 비용**: 삼성블루 그라데이션 (`#1428A0` → `#0A1872`)
- **FTE 절감**: 그린 그라데이션 (`#2E7D32` → `#1B5E20`)

계산식:
```
saved_hours = sum(removed_node.hours_per_year + sum(connected_edge.hours))
saved_cost  = saved_hours × 60000
saved_fte   = saved_hours / 2080
```

### 9.4 좌측 필터
- 부서 필터: 9개 직군별 그룹으로 묶고 그룹마다 접기/펼치기 가능. 각 그룹 헤더는 직군 색상 점 + 부서 개수 배지 표시. 상단에 `전체 / 사용중 / 해제` 일괄 버튼
- 직군 필터: 9개 체크박스 (색상 점 표시)
- 엣지 타입: 수동/자동/구두 체크박스
- 옵션: AX 후보만 보기 토글
- **모든 panel-section은 헤더 클릭 시 접기/펼치기** (caret ▾ 회전)
- 접힘 상태는 localStorage에 저장 (key: `da_sim_collapsed_sections`)

### 9.5 노드/엣지 추가/편집 모달
- "+ 노드 추가" 버튼 → 모달: 업무명, 부서, 담당자, 직군, 시간, 시스템, AX 후보 입력
- 부서 입력은 `<datalist>` 자동완성 (전체 카탈로그 + 사용 중 부서)
- 부서 선택 시 직군 자동 매칭 (DEPT_TO_JOB 매핑)
- 새 노드 ID 자동 생성: 직군 prefix 기반 (`S/Q/M/P/R/C/H/I/L`) + 다음 번호
- 노드 ID 생성 시 정확히 같은 prefix가 이미 사용 중이면 다음 번호로 증가
- "+ 엣지 연결" 버튼 → 모달: from/to 노드 드롭다운, 라벨, 시간, 전달 유형
- 우클릭 → "노드 편집" → 같은 모달이 기존 값 채워진 상태로 열림

### 9.6 다른 액션 버튼
- 우상단 툴바: `+ 노드 추가`, `+ 엣지 연결`, `★ AX 적용`, `선택 일괄 제거`, `화면 맞춤`
- 우측 하단: `결과 클립보드 복사`, `전체 복원`, `시뮬레이션 초기화`

---

## 10. ★ AX 캐터필러 트랙 애니메이션 (핵심 기능)

### 의미
사용자가 두 노드를 선택하고 "★ AX 적용" 버튼을 누르면, 그 두 노드를 **하나의 자동화 단위로 묶어** 시각화한다. 두 노드가 좌우 바퀴가 되고, 그 둘을 감싸는 **포크레인 캐터필러 궤도** 모양의 애니메이션이 그려진다.

### 데이터 구조
```js
state.axPairs = [{ a: nodeId, b: nodeId }, ...];
state.selectionOrder = [];  // 클릭 순서 보존
```

### 애니메이션 루프
```js
network.on('afterDrawing', drawAXTracks);
function startAXAnimation() {
  function tick() {
    if (state.axPairs.length > 0) network.redraw();
    requestAnimationFrame(tick);
  }
  tick();
}
```

### 트랙 그리기 알고리즘 (페어 단위)
두 노드 위치 `p1, p2`, 노드 반경 `r1, r2`가 주어졌을 때:

```js
function drawPairTrack(ctx, p1, p2, r1, r2, phase, spin) {
  const dx = p2.x - p1.x, dy = p2.y - p1.y;
  const theta = Math.atan2(dy, dx);              // 두 노드를 잇는 축의 각도
  const R = Math.max(r1, r2);                    // 통일 반경

  const top = theta - Math.PI / 2;               // 축 기준 "위쪽" 수직 각도
  const bot = theta + Math.PI / 2;               // 축 기준 "아래쪽" 수직 각도
  const cosT = Math.cos(top), sinT = Math.sin(top);
  const cosB = Math.cos(bot), sinB = Math.sin(bot);

  // 1) 외곽 트랙 (캡슐 = 두 원의 외접 stadium)
  ctx.strokeStyle = 'rgba(20, 40, 160, 0.88)'; ctx.lineWidth = 6;
  ctx.beginPath();
  ctx.moveTo(p1.x + R*cosT, p1.y + R*sinT);  // 위쪽 직선 시작
  ctx.lineTo(p2.x + R*cosT, p2.y + R*sinT);  // 위쪽 직선 끝
  ctx.arc(p2.x, p2.y, R, top, bot, false);   // p2 주위 앞쪽 반원
  ctx.lineTo(p1.x + R*cosB, p1.y + R*sinB);  // 아래쪽 직선
  ctx.arc(p1.x, p1.y, R, bot, top, false);   // p1 주위 뒤쪽 반원
  ctx.closePath(); ctx.stroke();

  // 2) 움직이는 트레드 (앰버 점선이 흐름)
  ctx.strokeStyle = '#FFB300'; ctx.lineWidth = 8;
  ctx.setLineDash([12, 9]);
  ctx.lineDashOffset = -phase;  // phase = (performance.now()/1000) * 60
  // 위와 같은 path를 다시 그려서 stroke
  ctx.setLineDash([]);

  // 3) 내부 하이라이트 (depth)
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.55)'; ctx.lineWidth = 1.2;
  // 평행한 두 직선만 짧게

  // 4) 두 바퀴의 림 + 회전 스포크 (FILL 하지 않음 — 노드가 가려지면 안 됨)
  drawWheelOutline(ctx, p1.x, p1.y, R - 4, spin);   //  spin = (now/1000) * 1.6 rad
  drawWheelOutline(ctx, p2.x, p2.y, R - 4, -spin);  // 반대로 회전 (구름 효과)

  // 5) "★ AX 통합" 라벨 (두 노드 사이 위쪽)
  const midX = (p1.x + p2.x) / 2;
  const midY = (p1.y + p2.y) / 2;
  const labelOffset = R + 18;
  ctx.font = 'bold 12px "Nanum Gothic"';
  ctx.fillStyle = '#FFB300';
  ctx.strokeStyle = 'rgba(20, 40, 160, 0.95)'; ctx.lineWidth = 4;
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.strokeText('★ AX 통합', midX + cosT*labelOffset, midY + sinT*labelOffset);
  ctx.fillText('★ AX 통합', midX + cosT*labelOffset, midY + sinT*labelOffset);
}

function drawWheelOutline(ctx, cx, cy, r, angle) {
  ctx.save();
  ctx.strokeStyle = '#1428A0'; ctx.lineWidth = 2.8;
  ctx.beginPath(); ctx.arc(cx, cy, r, 0, Math.PI*2); ctx.stroke();
  ctx.translate(cx, cy); ctx.rotate(angle);
  ctx.strokeStyle = 'rgba(20, 40, 160, 0.45)'; ctx.lineWidth = 1.8;
  for (let i = 0; i < 6; i++) {
    const a = (i / 6) * Math.PI * 2;
    ctx.beginPath(); ctx.moveTo(0, 0);
    ctx.lineTo(Math.cos(a) * (r-2), Math.sin(a) * (r-2)); ctx.stroke();
  }
  ctx.fillStyle = '#FFB300';
  ctx.beginPath(); ctx.arc(0, 0, Math.max(3, r*0.12), 0, Math.PI*2); ctx.fill();
  ctx.restore();
}
```

### 페어 추가 로직
```js
function applyAXToSelection() {
  const ids = state.selectionOrder.filter(id => state.selectedNodeIds.has(id));
  if (ids.length < 2) { showToast('노드 2개 이상 선택'); return; }
  // 선택 순서대로 연쇄 페어 생성: (A,B), (B,C), (C,D)...
  // 모두 이미 묶여 있으면 일괄 해제 (토글)
  // ...
}
```

### 정리
- 노드 제거 시 그 노드를 포함한 페어 자동 제거
- 시뮬레이션 초기화 시 모든 페어 제거

---

## 11. 엣지 길이 = 병목 크기 (필수)

```js
function edgeToVis(e) {
  const length = 80 + e.hours * 2.2;     // 처리시간이 클수록 화살표가 길어짐
  const width  = (e.transfer_type === '수동전달' ? 1.5 : 1.0) + Math.min(4.5, e.hours / 28);
  let color = EDGE_COLORS[e.transfer_type];
  if (e.transfer_type === '수동전달' && e.hours >= 100) color = '#E53935';
  else if (e.transfer_type === '수동전달' && e.hours >= 60) color = '#EF5350';
  return {
    id: e.id, from: e.from, to: e.to,
    label: `${e.label} · ${e.hours}h`,
    arrows: { to: { enabled: true, scaleFactor: 0.7 } },
    color: { color, highlight: '#1428A0' },
    width, length,
    dashes: e.transfer_type === '구두보고',
    font: { size: 9, color: '#444', face: 'Nanum Gothic', strokeWidth: 3, strokeColor: '#fff', align: 'middle' },
    smooth: { type: 'continuous', roundness: 0.2 },
    _data: e
  };
}
```

---

## 12. vis-network 옵션

```js
{
  autoResize: true,
  nodes: {
    shape: 'box',
    size: 24,
    widthConstraint: { minimum: 90, maximum: 130 }
  },
  edges: { smooth: { type: 'continuous' } },
  physics: {
    enabled: true,
    solver: 'barnesHut',
    barnesHut: {
      gravitationalConstant: -18000,
      centralGravity: 0.1,
      springLength: 130,
      springConstant: 0.015,
      damping: 0.55,
      avoidOverlap: 0.5
    },
    stabilization: { iterations: 1000, updateInterval: 30, fit: true },
    minVelocity: 0.6,
    maxVelocity: 35
  },
  interaction: { hover: true, multiselect: true, tooltipDelay: 200 },
  layout: { improvedLayout: true }
}
```

노드 변환:
```js
function nodeToVis(n) {
  return {
    id: n.id,
    label: n.label,
    color: {
      background: JOB_COLORS[n.job_type] || '#6B7280',
      border: n.ax_candidate ? '#FFD600' : JOB_COLORS[n.job_type],
      highlight: { background: JOB_COLORS[n.job_type], border: '#FF1744' }
    },
    borderWidth: n.ax_candidate ? 3 : 1,
    borderWidthSelected: 4,
    font: { color: '#fff', size: 12, face: 'Nanum Gothic', bold: { color: '#fff', size: 12 } },
    shape: 'box', margin: 10,
    shadow: { enabled: true, color: 'rgba(0,0,0,0.15)', size: 6, x: 1, y: 2 },
    title: buildTooltipDom(n),
    _data: n
  };
}
```

---

## 13. 결과 클립보드 복사 (export)

TSV 형식으로 클립보드 복사 (Excel 즉시 붙여넣기 가능):

```
[DA사업부 업무 노드 제거 시뮬레이션 결과]

▌제거 노드: N개

No.  업무명  부서  직군  업무시간(h)  연결시간(h)  합계(h)  절감비용(원)
1    ...
...

▌연간 절감 시간: ###,### h
▌연간 절감 비용: ###,###,### 원 (시간당 60,000원 기준)
▌FTE 절감     : #.## 명 (2,080h/년 기준)
```

---

## 14. 부팅 순서

```js
function boot() {
  if (typeof vis === 'undefined' || !vis.DataSet) { setTimeout(boot, 80); return; }
  state.nodes = new vis.DataSet();
  state.edges = new vis.DataSet();
  loadInitialData();      // initialNodes/Edges → DataSet
  buildFilters();         // 좌측 패널
  initNetwork();          // vis.Network 생성 + 이벤트 바인딩 + AX afterDrawing
  recomputeSavings();
  renderRemovedList();
  updateStatsBanner();
  updateSelectionUI();
  setupCollapsibleSections();
  setTimeout(() => { network.redraw(); network.fit({ animation: false }); }, 200);
  setTimeout(() => network.fit({ animation: { duration: 600 } }), 800);
  window.addEventListener('resize', () => network.redraw());
}
boot();
```

---

## 15. 완성 기준 (Acceptance Criteria)

1. ✅ 142개 노드 / 240+ 엣지가 그래프로 표시됨
2. ✅ 노드 클릭 → 누적 선택, 빈공간 클릭 → 전체 해제
3. ✅ 노드 제거 → 절감 카운터 즉시 반영
4. ✅ Undo / 전체 복원 / 시뮬레이션 초기화 동작
5. ✅ 부서/직군/엣지 타입 필터 동작
6. ✅ 노드 추가/편집 모달 동작 (직군 자동 매칭)
7. ✅ 결과 클립보드 복사 동작
8. ✅ 모든 panel-section 헤더 클릭 시 접기/펼치기
9. ✅ 접힘 상태가 localStorage에 저장되어 새로고침 후에도 유지
10. ✅ **★ AX 적용**: 두 노드 선택 → 버튼 클릭 → 캐터필러 궤도가 두 노드를 감싸며 회전 시작
11. ✅ AX 페어 트랙은 노드가 움직여도 따라가며 각도 자동 조정
12. ✅ 엣지 길이가 hours에 비례 (`80 + hours * 2.2`)
13. ✅ 엣지 색상: hours 60↑ `#EF5350`, 100↑ `#E53935` (수동 전달 페인 포인트 강조)
14. ✅ vis-network CDN 로딩 실패 시 안내 박스 표시 (jsdelivr → cdnjs → unpkg fallback)
15. ✅ 데스크탑 1920×1080 이상 환경에서 정상 동작

---

## 16. 흔한 실수 방지

- ❌ `state.nodes = new vis.DataSet()`을 전역에서 즉시 호출 → vis 미로드 시 에러
  → ✅ boot() 안에서 vis가 로드된 뒤 호출
- ❌ 그래프 div에 `height: calc(100% - 41px)` 사용 → 부모 0 높이면 캔버스 안 보임
  → ✅ flex column + min-height: 400px
- ❌ vis-network 단일 클릭이 선택 리셋
  → ✅ click 이벤트에서 직접 selection 토글하고 setSelection으로 동기화
- ❌ AX 트랙 그릴 때 wheel을 fill → 노드가 가려짐
  → ✅ stroke만 (drawWheelOutline)

---

## 17. 추가 컨벤션

- 모든 사용자 표시 텍스트는 **한국어**
- 시간 단위 표시: `1,234 h`, 비용: `₩123,456,789` 또는 `123,456,789 원`
- 토스트 메시지 위치: 화면 하단 중앙, 2.2초 후 자동 사라짐
- 배경색: `#F5F6F8`
- 카드/패널 흰색, 테두리 `#E5E7EB`, 그림자 `0 2px 6px rgba(0,0,0,0.06)`

---

## 18. 라이선스/저작권

이 시뮬레이터의 데이터는 가상의 페르소나/부서 구조로 구성되며, 어떤 실제 회사 내부 정보도 포함하지 않는다. 그대로 공개 가능.

---

**구현 완료 후 단일 HTML 파일 하나만 산출하라. 그 외 파일/폴더 생성 금지.**
