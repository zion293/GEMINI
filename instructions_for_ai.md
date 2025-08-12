# AI 개발 지시서 v1.1: 프로젝트 'cattytrip'

## 1. 기본 설정 (Setup)

- **프레임워크:** Next.js 14+ (App Router)
- **언어:** TypeScript
- **스타일링:** Tailwind CSS
- **상태 관리:** Zustand
- **핵심 API:** Google Maps API, Google Gemini API

**지시 1: 프로젝트 초기화**
- `npx create-next-app@latest cattytrip --typescript --tailwind --eslint` 명령어로 프로젝트를 생성한다.
- `npm install zustand @vis.gl/react-google-maps` 라이브러리를 설치한다.

---

## 2. UI/UX 구현 (Frontend)

### 지시 2: 글로벌 스타일 및 컴포넌트 정의
- **글로벌 스타일:** `globals.css`에 Pretendard 폰트를 기본으로 설정한다.
- **재사용 컴포넌트 (`components/ui`):**
  - **Button:** Primary, Secondary 스타일을 포함하는 버튼 컴포넌트를 생성한다. 호버 및 클릭 시 미세한 모션(밝기, 위치 이동)을 추가한다.
  - **Input:** 깔끔한 밑줄 또는 옅은 배경 스타일의 입력창 컴포넌트를 생성한다. Focus 시 테두리 색상이 Primary Color로 변경된다.
  - **Card:** `rounded-xl`, `shadow-md` 스타일의 카드 컴포넌트를 생성한다.

### 지시 3: 랜딩 페이지 및 로그인/회원가입 플로우 구현
- **랜딩 페이지 (`app/page.tsx`):**
  - **Hero Section:** 배경에 동영상, 중앙에 앱 인터랙션 데모 애니메이션, 헤드라인, 서브헤드라인, Primary Button(`[ AI와 함께 여행 계획 시작하기 ]`)을 배치한다.
  - **기능 소개 섹션:** 네비게이션 바 그래픽, 예산 관리(채팅+그래프) 이미지 등을 활용하여 서비스의 깊이를 점진적으로 보여주는 섹션들을 구현한다.
- **로그인/회원가입 (`app/login/page.tsx`):**
  - 단일 `Card` 컴포넌트 내에서 로그인 폼과 회원가입 폼이 전환되도록 구현한다.
  - **모션:** 폼 전환 시, 기존 필드는 부드럽게 슬라이드 아웃되고 새 필드가 슬라이드 인 되도록 애니메이션을 적용한다.

### 지시 4: 메인 플래너 UI 구현 (`app/planner/page.tsx`)
- **레이아웃:** 상단 고정 네비게이션 바, 좌측 지도(70%), 우측 채팅(30%)의 2단 레이아웃을 구현한다.
- **네비게이션 바:** 로고, 기능 탭, 사용자 프로필 드롭다운 메뉴를 포함한다.
- **지도 뷰 (`components/Map.tsx`):**
  - Zustand 스토어의 `destinations`와 `highlightedPlaceId`를 구독한다.
  - `destinations` 배열을 지도 위에 마커로 렌더링한다.
  - 각 마커의 `mouseover`/`mouseout` 이벤트가 `highlightedPlaceId`를 업데이트하도록 설정한다.
  - `highlightedPlaceId`가 변경되면 해당 마커의 아이콘을 변경하여 하이라이트 효과를 준다.
- **채팅 뷰 (`components/Chat.tsx`):**
  - **메시지 목록:** AI(좌측, 회색 배경), 사용자(우측, 파란 배경) 말풍선을 구분하여 렌더링한다.
  - **AI 응답 처리:**
    - AI가 응답을 생성 중일 때는 **애니메이션된 '입력 중...' 인디케이터**를 표시한다.
    - AI의 텍스트 답변은 **실시간으로 타이핑되듯** 스트리밍되어 나타나도록 구현한다.
    - 답변 내 장소 이름은 **클릭 가능한 칩(Chip)**으로, 일정은 **접고 펼 수 있는 아코디언(Accordion)** UI로 렌더링한다.

---

## 3. 핵심 기능 구현 (Core Features)

### 지시 5: 위치 정보 기반 계획 기능
- **Frontend:**
  - 채팅 메시지 전송 로직에 키워드(`'퇴근', '지금', '여기'`) 분석 단계를 추가한다.
  - 키워드 감지 시, `navigator.geolocation.getCurrentPosition()`을 호출하여 위치 권한을 요청하고 좌표를 획득한다.
  - 성공 시 API 요청에 `{ coords: { lat, lng } }`를, 실패 시 `{ location_unavailable: true }`를 포함한다.
- **Backend:**
  - API에서 `coords` 또는 `location_unavailable` 플래그를 확인한다.
  - `coords`가 있으면 AI 프롬프트에 출발지 정보를 추가하고, `location_unavailable`이면 출발지를 되묻도록 AI에게 지시한다.

### 지시 6: AI 채팅 API 및 Function Calling
- **API Route (`app/api/chat/route.ts`):**
  - `POST` 요청으로 `message`, `tripId`, `coords` 등을 받는다.
  - **Gemini API 연동:**
    - 시스템 프롬프트와 함께, 아래와 같은 **호출 가능한 함수(Tools)를 정의**하여 API에 전달한다.
      - `get_plan(destination, duration, budget, interests)`
      - `update_plan(tripId, modifications)`
      - `add_expense(tripId, amount, category, description)`
      - `update_checklist_item(tripId, itemName, isChecked)`
- **응답 처리:**
  - AI가 함수 호출을 요청하면, 해당 JSON을 파싱하여 서버 내부의 DB 조작 함수를 실행한다.
  - 함수 실행 후, 성공 여부와 함께 사용자에게 보여줄 자연스러운 텍스트 응답을 다시 AI에게 요청하여 프론트엔드로 스트리밍한다.

### 지시 7: 커뮤니티 및 게이미피케이션
- **Database:** Prisma/Drizzle ORM을 사용하여 `SharedPlan`, `UserProfile`(레벨, XP, 칭호 포함), `Like`, `Bookmark` 모델을 스키마에 추가한다.
- **API:** `POST /api/plans/{planId}/share`, `GET /api/community/plans` 등 커뮤니티 기능용 API 엔드포인트를 구현한다.
- **UI:** `app/community`와 `app/profile/[userId]` 경로에 페이지를 생성하여 커뮤니티 및 프로필 기능을 구현한다.
