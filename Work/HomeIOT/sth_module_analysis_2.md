# STH 모듈 중심 프로젝트 상세 분석 보고서

본 문서는 `thinq-web-framework` 프로젝트 내에서 **STH(Smart ThinQ Home, 홈 IoT 관련 모듈로 추정)** 모듈을 중심으로 특정 기능이나 로직을 작업하기 위해 반드시 알아야 할 아키텍처, 우선순위, 파일 구조 등을 아주 상세하게 분석한 가이드입니다.

---

## 🚀 1. 작업 전 필독! 최우선 파악 요소 (우선순위 Top 4)

STH 모듈은 일반적인 React 앱과 달리 자체적인 Data Management 계층(`AppData`)과 Namespace 기반 UI 컴포넌트(`STH.*`)를 엄격하게 사용하고 있습니다. 
작업을 시작하기 전 다음 4가지를 우선적으로 파악해야 합니다.

### ① 전역 데이터 및 상태 관리 패턴 (`src/wrm/STH/data/AppData.js`)
이 모듈은 `recoil`과 `redux`가 설치되어 있으나, STH 도메인 전용 데이터는 대부분 **`AppData`** 라는 static 클래스를 통해 중앙 집중식으로 관리됩니다.
- **`getType / setType` 구조**: 데이터는 `CONFIG_`, `HOME_`, `SITE_`, `SERVICE_` 접두사를 가진 상수로 분류되며, 내부적으로 `Map` 객체(`_configs`, `_homes`, `_sites`, `_services`)에 저장됩니다.
- **초기화 로직 (`AppData.init`)**: 앱이나 화면 진입 시 `AppData.init(feature)`이 호출되어 `NativeDataService`에서 홈 정보를 가져오고(`initHome`), API를 통해 단지 설정(`getSiteConfig`)을 불러옵니다.
- **데이터 로드 (`useAppService` 훅)**: React 컴포넌트 내에서는 `useAppService(feature)` 커스텀 훅을 사용하여 `Loader(로딩 스피너)` 처리, "서비스 점검 공지 확인" 등을 자동으로 수행하면서 데이터를 가져오도록 강제화되어 있습니다. 직접 `useEffect`에서 Axios를 호출하기보단 이 훅 패턴을 사용하는 것이 표준입니다.

### ② API 비동기 통신 규격 (`src/wrm/STH/data/ServiceApis.js`)
프로젝트는 `axios` 기반의 자체 Native API 브릿지를 사용하는 것으로 보이며, STH 모듈의 모든 서버 통신은 `ServiceApis` 클래스에 정의되어 있습니다.
- **단일 창구 (`_ServiceApis.request`)**: 모든 API 요청은 이 메서드를 거칩니다. 이 과정에서 오프라인 체크, 로딩 스피너 제어(`STH.Loader.add()`), 에러 발생 시 공통 팝업 노출 처리가 중앙에서 이루어집니다.
- **필수 공통 헤더 자동화**: `X_MESSAGE_ID`, `X_SERVER_CODE (homeiot)`, `X_SITE_ID`, `X_THINQ_HOME_ID` 등 백엔드가 요구하는 인증/식별 헤더들이 API 요청마다 자동으로 주입됩니다.
- **새로운 API 추가 시**: 컴포넌트에서 직접 호스팅 주소를 쓰지 말고, 반드시 `ServiceApis` 클래스 내에 static 메서드로 정의한 뒤, `AppData.useAppService` 등에 붙여서 컴포넌트로 전달해야 합니다.

### ③ 라우팅 체계 (`src/wrm/STH/routes.js`)
React Router(`react-router-dom`) 기반이며, 중앙 집중형 라우팅 배열(`_routes`)을 사용합니다.
- **Naming Rule**: STH 모듈 내 모든 경로는 `STH_Migration`, `STH_Register`, `STH_CarHistory`처럼 **`STH_` 접두사**를 가집니다.
- **`RootScreen` 래퍼**: 모든 컴포넌트는 `<RootScreen>`으로 래핑되어 렌더링 됩니다. 이 래퍼에서 `MixpanelLogger`(사용자 로그 추적) 화면 진입/이탈 이벤트를 자동으로 쏘며, 언어 다국어화(`setMultiLang`), `AppData.reset()` 등을 처리합니다.
- 새로운 화면을 추가한다면 반드시 이 파일에 `path`와 `component`를 등록해야 합니다.

### ④ 자체 UI 네임스페이스 컴포넌트 활용 (`src/wrm/STH/view/index.js`)
화면 UI를 개발할 때 일반적인 HTML 태그나 외부 라이브러리 컴포넌트를 무분별하게 혼용하지 마십시오.
- STH 모듈 전용 컴포넌트들이 `STH.ContentBox`, `STH.AppBar`, `STH.ButtonBar`, `STH.Loader` 처럼 묶여있습니다. 
- 스타일링은 **`styled-components`**를 100% 활용하고 있습니다. (인라인 스타일은 지양)
- 화면의 기본 뼈대는 `<STH.BodyContainer>`나 `<STH.ContentBox>` 안에서 `<STH.AppBar>`를 탑재하여 그리는 형태가 주를 이룹니다.

---

## 📁 2. 프로젝트 디렉토리 전체 구조 및 역할

```text
c:\Users\User\Desktop\HomeIOT\source\home_iot_dev\web\
 ├── package.json               # 각종 스크립트(npm run thinq_tpa) 및 의존성 모듈 정의 (React 17, recoil, styled-components 등 사용)
 └── src/
      └── wrm/
           └── STH/             # ★ 우리가 분석하는 메인 모듈 "Smart ThinQ Home"
                ├── routes.js   # 해당 모듈의 진입점. (모든 화면 라우팅, 로깅, 초기화 수행)
                │
                ├── data/       # 앱의 심장부 (데이터 상태, API 통신 레이어)
                │    ├── AppData.js     # 글로벌 상태값, 설정값 보관 (Domain-driven Maps)
                │    ├── ServiceApis.js # IoT, EmpBackend, Thinq2 연동 API 정의서
                │    └── Validator.js   # 기능 권한(Feature validation) 관련 로직 추측 (SiteConfig 연계)
                │
                ├── screens/    # 실제 페이지 단위 뷰 (도메인별 폴더링)
                │    ├── carhistory/     # 방문차량 출입 이력
                │    ├── carreservation/ # 방문차량 예약
                │    ├── living/         # 생활 서비스
                │    ├── mall/           # 아파트 상가 소식 / 검색
                │    ├── migration/      # 앱 마이그레이션 안내 및 처리
                │    ├── register/       # 세대 연동 (OTP 발급, 단지 검색, 이용약관 동의)
                │    ├── visitor/        # 방문자 이력
                │    ├── siteinfo/       # 단지 정보 
                │    └── Main.js, DevSettings.js 등 루트 화면들
                │
                ├── view/       # STH 공통 UI 라이브러리 (Namespace로 제공)
                │    ├── index.js             # STH 객체로 모든 컴포넌트를 묵어 export
                │    ├── STH_AppBar.js        # 상단 네비게이션 바
                │    ├── STH_ContentBox.js    # 본문 영역 컨테이너 (그리드(Grid)/반응형 기기 분기 지원)
                │    ├── STH_ButtonBar.js     # 하단 고정 버튼 영역
                │    ├── STH_Loader.js        # 비동기 통신 시 화면을 막는 스피너
                │    └── STH_EmptyContainer.js # 리스트가 비어있을 때 (데이터 없음) 표시 UI
                │
                ├── support/    # 유틸리티 성격의 코드들
                │    ├── ErrorReporter.js     # 에러 발생 시 공통 포맷 로깅/팝업 
                │    ├── MixpanelLogger.js    # Mixpanel 클리핑 
                │    ├── Utils.js             # 기타 유틸 (약관 체크, 공지 체크, 라우팅 헬퍼)
                │    ├── string.extension.js  # i18next 다국어 번역 관련 지원 파일
                │    └── test/                # API 테스트, 자동화/UI 테스트용 화면 모음
                │
                ├── interface/  # Native(Android/iOS) TPA 껍데기와의 브릿지 인터페이스 함수
                └── resource/   # 아이콘, Lottie 애니메이션 (json), 이미지 자산
```

---

## 🛠️ 3. 특정 기능/로직 개발 시나리오 가이드

새로운 기능(가령 `공지사항 조회 기능`)을 모듈에 추가한다고 가정할 때 따라야 할 **Standard Workflow**는 다음과 같습니다.

1. **API 정의 (`data/ServiceApis.js`)**: 
   - 백엔드에 요청할 `getNoticeList` 메서드를 `_ServiceApis` 내부에 구현.
   - `ServiceApis` 클래스에 static 메서드로 한번 더 매핑하여 외부로 export.
2. **상태 키값 할당 (`data/AppData.js`)**:
   - `AppData.Type` (예: `SERVICE_NOTICE_LIST`)에 상수를 정의합니다.
   - `useAppService` 내의 `services` 맵 객체에 키값과 정의해둔 API 함수 포인터를 연결해둡니다.
3. **UI 화면 제작 (`screens/{도메인}/...`)**:
   - `screens/notice/NoticeList.js`를 작성합니다.
   - 데이터는 훅(`const noticeData = useAppService(AppData.Type.SERVICE_NOTICE_LIST)`)을 사용해 비동기 및 에러 처리를 위임하고, 오직 화면 그리기에만 집중합니다.
   - 마크업은 `<STH.AppBar>`, `<STH.ContentBox>`, `<STH.EmptyContainer>`를 조합하여 통일성을 맞춥니다.
4. **라우팅 추가 (`routes.js`)**:
   - `path: 'STH_NoticeList'`를 부여하고 컴포넌트를 연결하여 앱에서 진입할 수 있게 구성합니다.

## 💡 4. 개발 시 반드시 고려해야 할 주의사항 (Gotchas)
- **로컬 스토리지 등에 직접 접근 자제**: 보안 및 상태 관리 파편화 방지를 위해 `AppData`나 `NativeDataService` 쪽을 우회해서 거치도록 설계되어 있습니다.
- **Mixpanel 주의**: 화면 이탈이나 진입 시 `routes.js`의 `RootScreen`에서 1차 로깅이 작동합니다. 각 화면 내 특정 버튼 클릭 시에는 `MixpanelLogger.js`의 함수를 명시적으로 호출해야 합니다.
- **반응형 개발 대응**: `STH_ContentBox` 내부 코드를 보면 태블릿 접속 여부(`isTablet`), 화면 가로모드 여부(`isLandscape`)를 분기하는 로직이 있습니다. 태블릿 및 가로/세로 전환이 중요한 요건인 프로젝트이므로 가로 폭 하드코딩 등은 피해야 합니다.
