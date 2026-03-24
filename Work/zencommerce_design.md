# ZenCommerce: 고성능 실무형 이커머스 플랫폼 설계

## 1. 프로젝트 개요
Next.js의 최신 기능을 집약하여 제작하는 대규모 이커머스 실무급 프로젝트입니다. 성능 최적화, 보안, 데이터 무결성 등을 중점적으로 다룹니다.

## 2. 핵심 기술 스택
*   **Framework**: Next.js 15 (App Router)
*   **Language**: TypeScript
*   **Styling**: Vanilla CSS (CSS Modules)
*   **Database**: PostgreSQL + Prisma (ORM)
*   **Authentication**: NextAuth.js
*   **State Management**: Zustand
*   **Payments**: Stripe 모의 연동

## 3. 주요 기능 (Phase 1: MVP)
1.  **상품 검색 및 브라우징**: Server Components 기반의 빠른 렌더링과 Dynamic Metadata 기반 SEO 최적화.
2.  **인증 및 권한**: Middleware를 활용한 보호된 경로 관리(Admin, Checkout).
3.  **장바구니 및 주문**: Server Actions를 이용한 데이터 변경 및 `revalidatePath` 캐시 무효화.
4.  **Admin 대시보드**: CRUD 기능이 포함된 관리자 전용 페이지.

## 4. 학습 포인트
*   **RSC (React Server Components)**: 서버와 클라이언트 컴포넌트의 적절한 분리와 상호작용.
*   **Data Fetching**: Next.js 캐싱 정책(ISR, SSG, SSR)의 실제 활용.
*   **Server Actions**: API 라우트 없이 폼 제출 및 데이터 변경 처리.
*   **Next/Image**: 이미지 최적화를 통한 웹 성능 점수 극대화.

---
*본 설계안에 동의하시면 `exit_plan_mode`를 호출하여 실제 프로젝트 구성을 시작하겠습니다.*
