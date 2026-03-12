# STH 모듈 분석 리포트 (`src/wrm/STH`)

## 1. 모듈 디렉토리 및 구조
STH 모듈은 홈IoT 단지 연동 및 편의 기능(엘리베이터, 방문자 조회, 차량 예약 등)을 제공하는 프론트엔드 모듈로 보입니다. 

- **`routes.js`**: 모듈의 라우팅(진입점)을 정의합니다. `/STH_...` 형태의 URL 경로와 화면(Screen) 컴포넌트를 매핑합니다. 모든 화면은 `<RootScreen>`으로 래핑되어 초기화 로직을 수행합니다.
- **`data/`**: 모듈의 전역 상태 관리 및 API 통신을 담당합니다.
  - **`AppData.js`**: 인메모리 전역 상태 저장소입니다. `CONFIG`, `HOME`, `SITE`, `SERVICE` 등의 도메인별로 데이터를 캐싱하고 관리합니다. (React Context 대신 전역 Map 객체 사용)
  - **`ServiceApis.js`**: 백엔드 REST API 호출을 집중 관리하는 클래스입니다. `axios`를 래핑하여 인증 헤더(`x-site-id`, `x-thinq-home-id` 등)를 공통으로 주입하고 에러를 처리합니다.
- **`screens/`**: 실제 사용자에게 보여지는 화면(UI) 컴포넌트들이 기능별(차량 이력, 방문자, 상가 등)로 폴더에 나뉘어 존재합니다. (`Main.js`, `carhistory/`, `visitor/` 등)
- **`support/`, `view/`, `interface/`**: 기타 유틸리티(`MixpanelLogger`, `AppLogger`), 공유 UI 컴포넌트, 인터페이스 파일들이 위치합니다.

## 2. 주요 실행 플로우 (Execution Flow)

1. **라우팅 및 초기화 (`routes.js`)**
   - 사용자가 특정 화면(예: `STH_CEN01_Main`)에 접근하면 `react-router`를 통해 매핑된 컴포넌트가 렌더링됩니다.
   - 렌더링 시 최상단에 있는 `<RootScreen>` 컴포넌트가 `useLayoutEffect`를 통해 `AppData.reset()`을 호출하여 기존 데이터를 초기화하고, 언어 설정 및 접근 로그(Mixpanel)를 기록합니다.

2. **데이터 및 상태 로딩 (`AppData.js`)**
   - 화면 컴포넌트 혹은 내부적으로 사용하는 훅(`useAppService`)에서 `AppData.init(feature)`을 호출합니다.
   - `init()` 함수는 다음의 흐름으로 작동합니다:
     - `initHome()`: 네이티브 앱/기기에서 현재 활성화된 홈 정보(homeId 등)를 가져옵니다.
     - `initSite()`: `ServiceApis`를 통해 홈의 상세 정보, 사이트(단지) 설정(`getSiteConfig`), 사용자 홈 정보(`getUserHomeInfo`)를 병렬로 API 호출하여 가져옵니다.
   - 가져온 데이터는 `AppData` 내부의 Map 객체(`_homes`, `_sites` 등)에 저장됩니다.

3. **서비스 API 호출 (`ServiceApis.js` & `useAppService`)**
   - **데이터 패치 훅**: `AppData.js`에서 제공하는 `useAppService` 훅은 내부적으로 `useNetwork`를 사용하여 백엔드 데이터를 가져오고 로딩 스피너(Loader)를 자동으로 띄우고 숨깁니다.
   - **통신**: `ServiceApis.request` 메서드를 거쳐 실제 API가 호출되며, 요청 헤더에 트래킹 ID(`x-message-id`)와 인증/식별값들을 주입합니다. 응답 에러 처리는 `showErrorReport` 유틸을 통해 공통으로 수행됩니다.

4. **화면 렌더링 (`screens/`)**
   - API 통신 및 데이터 로딩이 끝나면 React 상태가 변경되고, `screens` 디렉토리 아래의 화면 컴포넌트들이 준비된 데이터를 바탕으로 UI를 그립니다. (예: `carhistory` 화면에서 출입 이력 목록 렌더링)

## 요약
STH 모듈은 **`routes.js`**를 진입점으로 삼아, 화면(Screen) 진입 시 **`AppData.init()`**을 호출하여 사용자 및 단지(Site) 정보를 로드하고, 내부의 전역 캐시(Map)에 저장한 뒤, **`ServiceApis.js`**를 통해 서비스별 세부 데이터를 가져와 렌더링하는 명확한 데이터 단방향 플로우를 가집니다.
