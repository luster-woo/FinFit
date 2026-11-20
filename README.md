# FinFit

금융성향 검사를 먼저 하고, 그 결과로 예·적금, 주식, 카드를 추천해 주는 서비스입니다. 상품이 수백 개라 무엇부터 봐야 할지 모르는 사회초년생을 생각하고 만들었습니다.

| 홈 화면 | 금융성향 검사 결과와 추천 |
| :---: | :---: |
| <a href="./screenshots/00_home_guest.png"><img src="./screenshots/thumb/00_home_guest.png" width="420" alt="홈"></a> | <a href="./screenshots/13_test_result.png"><img src="./screenshots/thumb/13_test_result.png" width="420" alt="검사 결과"></a> |
| **목표 저축 플래너** | **AI 금융 상담** |
| <a href="./screenshots/17_planner.png"><img src="./screenshots/thumb/17_planner.png" width="420" alt="플래너"></a> | <a href="./screenshots/21_ai_chatbot_open.png"><img src="./screenshots/thumb/21_ai_chatbot_open.png" width="420" alt="챗봇"></a> |

<sub>화면을 누르면 전체 캡처가 열립니다.</sub>

| | |
|---|---|
| 기간 | 2026.05 ~ 2026.06 (SSAFY 1학기 관통 프로젝트) |
| 팀 | 2명. 저는 팀장이고 백엔드 전체를 맡았습니다. 프론트(Vue 3)는 이진웅 님이 만들었습니다 |
| 백엔드 | Python, Django, Django REST Framework, SimpleJWT, APScheduler |
| DB | 개발은 SQLite, 배포는 PostgreSQL 16 |
| 외부 API | 금융감독원 FinLife, SSAFY GMS, Gemini 1.5 Flash, yfinance, RSS, Kakao Local·Mobility |
| 배포 | Docker Compose, nginx, gunicorn, AWS EC2 (프로젝트가 끝나고 석 달 뒤에 따로 했습니다) |

## 무엇을 하는 서비스인가

10문항 검사로 사용자의 금융성향을 4자리 코드로 만듭니다. 이 코드는 상품 추천에만 쓰지 않고 유사 사용자 통계와 저축 플래너의 상품 필터에도 씁니다. 검사를 한 번 하면 서비스 여러 곳이 그 결과를 같이 쓰는 구조입니다.

그 밖에 예·적금 검색과 비교, 목표 저축 플래너, 주식 시세와 뉴스, AI 금융 상담, 주변 은행 찾기, 커뮤니티가 있습니다.

## 만들면서 고민한 것

### 느린 외부 API를 사용자 요청에서 떼어 냈습니다

처음에는 상품 목록을 열 때마다 금융감독원 FinLife API를 불렀습니다. 그러니 목록 하나 여는 데 평균 4.5초가 걸렸습니다. 필터나 정렬도 API가 지원하지 않아서 전체를 받아 코드에서 처리해야 했습니다.

외부 API 속도는 제가 바꿀 수 없어서 부르는 위치를 옮겼습니다. 관리자만 호출할 수 있는 수집 API가 상품을 DB에 저장하고, 사용자 요청은 DB만 읽습니다.

```text
[수집] 관리자 → FinLife API → DB 저장 (update_or_create)
[조회] 사용자 → Django ORM → 검색 · 정렬 · 필터 · 비교
```

| | 바꾸기 전 | 바꾼 뒤 |
|---|:---:|:---:|
| 상품 목록 응답 | 평균 4.5초 | 평균 80ms |
| 검색 · 정렬 · 필터 | API 제약으로 제한적 | ORM으로 처리 |

응답 시간은 같은 로컬 환경에서 브라우저 Network 탭과 Postman으로 전후 각각 20번 잰 평균입니다. 대신 상품 정보는 관리자가 다시 수집해야 바뀝니다. 스케줄러로 자동 수집하지는 않았습니다.

### 추천 기준은 AI가 아니라 서비스가 정했습니다

"적합한 상품을 추천해 줘"라고만 보내면 같은 입력에도 순위와 근거가 매번 달랐습니다. 그래서 채점 기준을 직접 정해 프롬프트에 넣었습니다.

| 기준 | 비중 |
|---|:---:|
| 금리 경쟁력 | 40 |
| 수익 절대액 (만기 예상 이자) | 30 |
| 상품 유형 적합성 | 15 |
| 은행 신뢰도와 인근 여부 | 15 |

처음에는 금리가 수익성을 가장 직접 보여 준다고 보고 금리 비중을 가장 크게 뒀습니다. 그랬더니 금리는 높은데 납입 한도가 낮거나 기간이 짧아서 실제로 받는 이자는 적은 상품이 위로 올라왔습니다. 금리는 비율이고 이자는 금액이라, 만기 예상 이자를 별도 항목으로 뺐습니다.

기준을 넣은 뒤에는 목돈·기간 조합 50가지를 두 번씩 돌려 추천 상품, 순위, 근거 문장이 같게 나오는지 확인했습니다. 가중치가 정답이라는 뜻은 아니고, 사회초년생이라는 대상에 대한 가설을 숫자로 옮긴 것입니다.

### AI가 답을 안 줘도 추천은 나옵니다

추천, 상담, 뉴스 번역, 주간 브리핑이 전부 외부 AI API(SSAFY GMS)에 걸려 있었습니다. 추천만큼은 멈추면 안 된다고 보고 넘어갈 경로를 미리 만들어 뒀습니다.

```text
GMS → (실패) → Gemini 1.5 Flash → (실패) → 규칙 기반 점수
실패로 보는 경우: 요청 예외, 20초 타임아웃, 빈 응답, JSON 파싱 실패
```

AI가 JSON 대신 코드블록으로 감싸 보내면 벗겨 내고 파싱하고, 후보에 없는 상품 ID를 말하면 버립니다.

발표 이틀 전에 팀이 같이 쓰는 GMS 사용량 한도가 다 떨어졌습니다. 그날은 Gemini도 응답하지 않아서 추천은 규칙 기반 점수로 결과를 냈습니다. 대비해 두지 않은 번역, AI 상담, 주간 브리핑은 그대로 멈췄고, 가벼운 모델로 바꾸고 사용량을 더 받아서 발표 전에 복구했습니다. 그 뒤에 세 기능에도 다른 모델로 넘어가는 경로를 붙였습니다.

### 성향 검사에는 역방향 문항을 넣었습니다

모든 문항을 같은 방향으로 물으면 읽지 않고 한쪽으로만 답할 수 있어서, Q2·Q6·Q9는 반대로 묻고 채점할 때 뒤집습니다. 모두 최고점, 모두 최저점, 역방향 문항만 반대로 고른 응답을 넣어 보니 보정을 빼면 성향이 거꾸로 나왔습니다. 발표가 끝난 뒤 이 로직에 단위 테스트 18개를 붙였습니다.

### 뉴스 수집과 주간 브리핑

RSS 7곳에서 3시간마다 뉴스를 모아 정제하고, 매주 월요일에 AI가 주간 브리핑을 만듭니다. 개발 서버가 코드를 다시 읽을 때마다 스케줄러가 두 번 등록되는 문제가 있어서 실행 프로세스를 구분해 막았습니다. 이 방식은 배포할 때 다시 문제가 됐습니다(아래 배포 참고).

## 구조

```text
                    Vue 3
                      │ REST · JWT
                      ↓
                Django / DRF
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
   SQLite / PG     상품 수집       주식 · 뉴스 수집
                  (관리자 API)      (스케줄러)
                      ↓              ↓
                 FinLife API    yfinance · RSS

   추천 · 상담 · 번역 · 브리핑 → GMS → Gemini → 규칙 기반 점수(추천만)
```

API 전체 목록은 [API 명세](./docs/api.md)에 있습니다.

## 아직 남은 문제

- 로그아웃할 때 Refresh Token을 블랙리스트에 넣는 코드를 백엔드에 만들었지만, 프론트가 refresh 토큰을 보내지 않아서 실제 로그아웃 경로에서는 동작하지 않습니다.
- Gemini인지 판별하는 코드가 세 곳에 복사돼 있고 주소 문자열에 기댑니다. 다시 한다면 공급자를 설정값으로 두고 호출 함수를 하나로 모으겠습니다.
- 주식은 종목 조회, 관심 종목, 시세 차트까지 만들었고 모의 매수·매도는 기간 안에 붙이지 못했습니다.

## 실행 방법

```bash
git clone https://github.com/luster-woo/finance_pjt.git
cd finance_pjt

# 백엔드
pip install -r requirements.txt
cd BE
python manage.py migrate
python manage.py runserver

# 프론트 (새 터미널)
cd FE
npm install
npm run dev
```

루트의 `.env`에 아래 값을 넣습니다. `.env.example`을 참고하면 됩니다.

```env
SECRET_KEY=
FINLIFE_API_KEY=
GMS_API_KEY=
GMS_API_URL=
GEMINI_API_KEY=
KAKAO_REST_API_KEY=
VITE_KAKAO_MAP_KEY=
```

서버를 띄운 뒤 관리자 계정으로 아래 두 API를 호출하면 상품 데이터가 들어옵니다.

```text
/products/save-products/
/products/save-saving-products/
```

## 배포

프로젝트 기간에는 배포를 하지 못했습니다. 석 달 뒤에 운영 구성을 따로 만들어 AWS EC2(t3.small, Ubuntu 24.04)에 올렸습니다.

```text
브라우저 :80
   │
 nginx ── /         → Vue dist (SPA)
       ── /api/*    → gunicorn (접두사 제거)
       ── /admin/   → gunicorn
       ── /static/  → collectstatic 결과
            │
          web (Django + gunicorn 워커 3) ── db (PostgreSQL 16)
          sched (APScheduler)            ──┘
```

| | 개발 | 운영 |
|---|---|---|
| DB | SQLite 파일 | PostgreSQL 컨테이너 (볼륨 분리) |
| 정적 파일 | Django가 직접 제공 | collectstatic 후 nginx |
| 스케줄러 | runserver 프로세스 안 | 전용 컨테이너 1개 |
| 비밀값 | `.env` | 서버의 `.env.prod` (git 제외) |
| 프론트 API 주소 | `http://127.0.0.1:8000` | `/api` (상대 경로) |

가장 신경 쓴 건 스케줄러입니다. 개발 때 중복 실행을 막던 `RUN_MAIN`은 runserver가 만드는 값이라 gunicorn에는 없습니다. 그대로 올렸다면 스케줄러가 오류도 없이 꺼졌을 겁니다. 그렇다고 web 컨테이너에서 켜면 워커 3개가 같은 수집을 세 번 돌립니다. 웹 서버는 늘려도 주기 작업은 하나만 돌아야 해서 컨테이너를 따로 뺐습니다.

```bash
cp .env.prod.example .env.prod   # SECRET_KEY, ALLOWED_HOSTS, POSTGRES_PASSWORD
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --build
```

외부에서 잰 `/api/products/` 응답은 0.40초였고, 컨테이너 네 개의 메모리는 합쳐서 469MB였습니다. 도메인이 없어 HTTPS는 붙이지 않았습니다. 확인을 마친 뒤 인스턴스는 종료해서 지금 접속할 주소는 없습니다.

절차는 [배포 문서](./deploy/README.md)에, 서버에서 막혔던 네 군데와 원인은 [배포일지](./deploy/배포일지.md)에 적었습니다.

## 문서

| 문서 | 내용 |
|---|---|
| [설계서](./docs/설계서.md) | 기획 의도, 기능 설계, 흐름도, DB 모델링 |
| [API 명세](./docs/api.md) | 전체 엔드포인트와 요청·응답 예시 |
| [기술 스택](./docs/tech-stack.md) | 기술을 고른 이유 |
| [트러블슈팅](./docs/trouble-shooting.md) | 개발하면서 막힌 문제와 해결 과정 |
| [배포](./deploy/README.md) · [배포일지](./deploy/배포일지.md) | 운영 구성, EC2 배포 절차, 막힌 지점 |

## 돌아보며

추천 쪽이 가장 만족스럽습니다. AI에 판단을 다 맡기지 않고 기준을 먼저 정해 넣었더니 결과가 일관됐고 근거도 같이 뽑을 수 있었습니다.

외부 API를 여럿 쓰면서 장애와 응답 변경을 직접 겪은 것도 남았습니다. yfinance는 비공식 라이브러리라 응답 구조가 어느 날 바뀌었고, GMS는 한도가 떨어졌습니다. 외부 서비스가 늘 정상이라는 가정은 이 프로젝트에서 버렸습니다.

아쉬운 건 배포입니다. 로컬에서 돌아가면 끝이라고 보고 일정에 넣지 않았습니다. 처음부터 배포를 생각했다면 `RUN_MAIN`처럼 개발 서버에만 있는 전제를 코드에 넣지 않았을 겁니다. 환경 차이는 배포할 때 발견하는 것보다 처음부터 가정하지 않는 편이 쌉니다.

## 팀

| 이름 | 역할 |
|---|---|
| 이윤우 | 팀장, 백엔드 전체 (Django API 설계와 구현, 배포) |
| 이진웅 | 프론트엔드 전체 (Vue 3) |
