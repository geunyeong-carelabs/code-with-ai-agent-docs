# 암종(CancerType) 뇌종양(BRAIN_TUMOR) 추가 구현 계획서

## 개요

- **배경/문제**: 공용 암종 enum `BASE_CANCER_TYPES`(`types/shared.ts:51-58`)에 뇌종양이 없어, 병원 검색·상담·커뮤니티·마이페이지 등 암종 기반 전 기능에서 뇌종양 환자를 위한 흐름 자체가 존재하지 않는다(#KAN-2170, 1단계 작업 KAN-2170-1).
- **목표**: `BRAIN_TUMOR`를 `CancerType` enum에 추가하고, 파생 스키마·매핑·아이콘·UI·백엔드 연동 지점을 정합성 있게 갱신해 다른 암종과 동일한 수준으로 뇌종양을 지원한다.
- **범위**:
  - 포함: 프론트엔드 enum/타입/Zod 스키마, 매핑 객체(라벨·아이콘·경로·코드), 아이콘 에셋 등록, 관련 UI(필터/드로어/탭) 렌더링 확인
  - 제외: 백엔드 REST API 자체 로직 구현(백엔드팀이 프론트와 동시 배포로 별도 진행, 이 문서 스코프 밖)
- **원인/동작 흐름**: 이번 작업은 버그 수정이 아닌 신규 값 추가이므로 "적용 흐름"으로 대체.
  `types/shared.ts:51-58` `BASE_CANCER_TYPES`에 `'BRAIN_TUMOR'` 추가 →
  `cancerTypeEnumSchema`/`CancerType`(`types/shared.ts:63-65`) 및 이를 재사용하는 `types/hospital.ts`, `types/home.ts`, `types/info.ts`, `types/search.ts`, `types/users.ts`, `types/my-page.ts`, `types/consultation.ts`의 스키마가 자동 확장 →
  `Record<CancerType, ...>` 형태로 선언된 4곳(`src/lib/codeMap.ts:18-33`, `src/lib/constants.ts:217-227`, `DoctorAppDownloadBanner.tsx:17-27`)에서 `tsc` 컴파일 에러 발생 →
  각 에러 지점에 `BRAIN_TUMOR` 항목 수동 추가로 해소 →
  컴파일러가 강제하지 않는 지점(`src/lib/utils/cancer.ts`의 switch 4개, `src/lib/constants.ts`의 `CANCER_TYPE_OPTIONS`류, `VALID_CANCER_COMMUNITY_TYPES`, `types/shared.ts:78-88`의 `cancerTypeParamEnumSchema`, `types/board.ts:21-31`의 `BoardCancerCode`)는 체크리스트 기반으로 수동 갱신 →
  아이콘 SVG 3종(`public/icons/cancer/`, `.../line/`) 추가 및 `public/icons/index.ts` 레지스트리 등록 →
  백엔드가 `'BRAIN_TUMOR'` raw string을 URL/body 파라미터로 수용하는지 확인, Mixpanel `DT` 코드 발급 확인.
  즉 `BASE_CANCER_TYPES` 한 곳만 바꾸면 대부분은 자동/컴파일 에러로 유도되지만, switch-with-default 패턴 4곳과 별도 리터럴 배열 2곳은 컴파일러가 침묵하므로 반드시 체크리스트로 수동 확인해야 한다.
- **제약 조건**:
  - `BASE_CANCER_TYPES`의 기존 8개 값은 전부 `CANCER_` 접두어(`CANCER_LIVER` 등)이지만, 신규 값은 요청자 확정에 따라 접두어 없이 `BRAIN_TUMOR`로 추가한다(기존 컨벤션과 의도적으로 다름 — 리뷰 시 오타로 오인되지 않도록 PR 설명에 명시)
  - `BASE_CANCER_TYPES`에서 파생되는 스키마(`cancerTypeWithNoneEnumSchema`, `consultationQuestionTypeEnumSchema` 등)의 파생 구조 자체는 변경하지 않음 — 값만 추가
  - API 호출은 기존과 동일하게 `proxyClient`만 사용, 별도 `fetch`/`axios` 도입 금지
  - `src/lib/utils/cancer.ts`의 switch문에 `default` fallback을 제거하거나 exhaustive 패턴(`never` 체크)으로 리팩터링하는 것은 이번 작업 범위 밖 — 이번엔 기존 패턴대로 case만 추가(리팩터링은 별도 티켓으로 제안)
- **확정된 결정 사항** (요청자 확인 완료):
  - enum 값 이름: `BRAIN_TUMOR`
  - `cancerTypeParamEnumSchema`(`types/shared.ts:78-88`) URL slug 값: `'brain-tumor'`
  - `types/board.ts`의 `BoardCancerCode`(커뮤니티 게시판 코드)에도 추가 — 커뮤니티 게시판 신설 포함
  - 백엔드는 프론트와 동시 배포 예정 — 배포 순서 조율 불필요, 별도 대기 없이 진행
  - Mixpanel `DT` 코드: 기존 값이 100단위 증가(`CANCER_LIVER`=DT0000 ~ `CANCER_OTHER`=DT0800, `NONE`=DT9999 고정)이므로 다음 빈 값인 `DT0900`을 배정. 분석팀 공식 코드북이 별도로 있다면 그 값으로 교체(매핑 값 하나만 바꾸면 되므로 리스크 낮음)
  - 아이콘 에셋: 필요 시 Figma URL을 받아 프로젝트 관례대로 `node scripts/download-figma-image-3x.js <node-id>`로 다운로드 후 사용(디자인팀 별도 제작 요청 불필요)

## 구현 단계

### 1단계 — enum/스키마 확장

**작업 내용**
1. `types/shared.ts:51-58` `BASE_CANCER_TYPES`에 `'BRAIN_TUMOR'` 추가.
2. `types/shared.ts:78-88` `cancerTypeParamEnumSchema`에 `'brain-tumor'` 추가(자동 파생 안 됨, 별도 리터럴 배열).
3. `types/board.ts:21-31` `boardCancerCodeSchema`에 커뮤니티 게시판 코드(예: `'cancer-brain-tumor'`, 기존 `'cancer-liver'` 등과 동일 네이밍 패턴) 추가.
4. `types/consultation.ts:44-53`은 `BASE_CANCER_TYPES` 스프레드라 자동 반영됨 — 변경 불필요, 확인만.

**완료 기준**
- [ ] `pnpm ts-check` 실행 시 1단계 변경으로 인한 신규 컴파일 에러 목록이 예상한 4곳(2단계 대상)과 정확히 일치
- [ ] `cancerTypeEnumSchema.parse('BRAIN_TUMOR')`(또는 확정 값)가 통과하는지 로컬 스크립트/REPL로 확인

**단계 완료 메모**
- 이번 단계에서 실제로 한 일: {내용}
- 다음 단계 시작 전 알아야 할 것: {내용}

### 2단계 — 컴파일러가 강제하는 매핑 갱신

**작업 내용**
1. `src/lib/codeMap.ts:18-28` `CANCER_TYPE_CODE_MAP`에 `[CancerTypeEnum.BRAIN_TUMOR]: 'DT0900'` 추가(분석팀 공식 코드북 확인 시 값 교체).
2. `src/lib/codeMap.ts:30-33` `MEMBER_CANCER_TYPE_CODE_MAP`에 동일 반영(스프레드라 1번만 하면 자동 포함, 확인만).
3. `src/lib/constants.ts:217-227` `CANCER_TYPE_TO_PARAM`에 `brain-tumor` 매핑 추가.
4. `src/lib/constants.ts:229-239` `CANCER_PARAM_TO_TYPE`은 키가 `CancerTypeParam`이라 자동 에러는 안 나지만, 역매핑 누락 방지를 위해 함께 추가.
5. `src/app/doctor/cancer/[cancerTypeParam]/[doctorId]/_components/DoctorAppDownloadBanner.tsx:17-27` `DOCTOR_COUNT_BY_CANCER_TYPE`에 초기값(0 또는 실제 집계값) 추가.

**완료 기준**
- [ ] `pnpm ts-check` 통과(신규 컴파일 에러 0건)
- [ ] `pnpm build` 성공

**단계 완료 메모**
- 이번 단계에서 실제로 한 일: {내용}
- 다음 단계 시작 전 알아야 할 것: {내용}

### 3단계 — 컴파일러 미검증 매핑/자산 수동 갱신

**작업 내용**
1. `src/lib/utils/cancer.ts:11-34` `getCancerKorNameByCancerType`에 한글 라벨(예: "뇌종양") case 추가.
2. `src/lib/utils/cancer.ts:43-68` `getCancerIconNameByCancerType`, `:70-95` `getCancerWithPointColorIconNameByCancerType`에 신규 아이콘명 case 추가(아이콘 자산은 아래 4항목과 병행).
3. `src/lib/utils/cancer.ts:130-153` `getCancerPathByCancerType`에 영문 경로 slug case 추가.
4. Figma URL 수령 후 `node scripts/download-figma-image-3x.js <node-id>`로 아이콘 3종(line/filled/selected) 다운로드 → `public/icons/cancer/`, `public/icons/cancer/line/`에 배치 → `public/icons/index.ts` 임포트 및 `IconName` 레지스트리에 등록.
5. `src/lib/constants.ts:105-129` `CANCER_TYPE_OPTIONS`, `MEMBER_CANCER_TYPE_OPTIONS`에 옵션 항목 추가(`CANCER_TYPE_MAP` 등은 파생이므로 자동 반영 확인만).
6. `src/lib/constants.ts:86-96` `VALID_CANCER_COMMUNITY_TYPES`에 `'cancer-brain-tumor'` 코드 추가(커뮤니티 게시판 포함 확정에 따라 필수).

**완료 기준**
- [ ] `src/lib/utils/cancer.ts`의 4개 함수 모두 신규 값에 대해 fallback이 아닌 실제 매핑값 반환(단위 테스트 또는 콘솔 확인)
- [ ] 신규 아이콘 3종이 브라우저에서 정상 렌더링(경로 오타/404 없음)
- [ ] `pnpm lint` 통과

**단계 완료 메모**
- 이번 단계에서 실제로 한 일: {내용}
- 다음 단계 시작 전 알아야 할 것: {내용}

### 4단계 — UI 렌더링 확인

**작업 내용**
1. `src/components/drawer/CancerTypeDrawer.tsx` — 온보딩/홈 암종 선택 드로어에 뇌종양 항목 노출 확인.
2. `src/app/hospital/cancer/[cancerTypeParam]/_components/cancer-filter/CancerFilterButton.tsx` — 병원 검색 필터 탭 노출 확인.
3. `src/app/consultation/cancer/_components/consultaion-list/ConsultationCancerTypeFilter.tsx` — 상담 목록 필터 노출 확인.
4. `src/app/auth/sign-up/_components/SignUpStep.tsx`, `src/app/my-page/info/_components/edit-info/EditMyInfoForm.tsx` — 회원가입/프로필 수정 시 뇌종양 선택 후 저장·Mixpanel 전송 정상 동작 확인.
5. `src/app/community/_components/community-filter/CancerButton.tsx`, `CommunityHomeSelectFilter.tsx`, `src/app/community/[boardCode]/_components/community-filter/CommunitySelectFilter.tsx` — 뇌종양 커뮤니티 게시판 필터 노출 및 `/community/cancer-brain-tumor` 라우트 정상 진입 확인(`VALID_CANCER_COMMUNITY_TYPES` 반영 여부 포함).

**완료 기준**
- [ ] 위 4개 화면에서 로컬(`pnpm dev`) 기준 뇌종양 선택/필터링/저장 플로우 정상 동작
- [ ] 320/768/1024/1440 breakpoint에서 뇌종양 항목 레이아웃 깨짐 없음

**단계 완료 메모**
- 이번 단계에서 실제로 한 일: {내용}
- 다음 단계 시작 전 알아야 할 것: {내용}

### 5단계 — 테스트 추가(회귀 방지)

**작업 내용**
1. `src/lib/utils/cancer.test.ts`(신규) — `getCancerKorNameByCancerType` 등 4개 함수가 `BASE_CANCER_TYPES`의 모든 값에 대해 fallback이 아닌 값을 반환하는지 파라미터라이즈 테스트 추가(향후 암종 추가 시 누락을 자동으로 잡아주는 회귀 테스트).
2. `src/lib/constants.test.ts`(신규) — `CANCER_TYPE_OPTIONS`, `CANCER_TYPE_TO_PARAM`, `CANCER_PARAM_TO_TYPE`이 `BASE_CANCER_TYPES` 전체를 커버하는지 검증.

**완료 기준**
- [ ] `pnpm test` 신규 테스트 전체 통과
- [ ] `pnpm coverage` 기존 커버리지 대비 하락 없음

**단계 완료 메모**
- 이번 단계에서 실제로 한 일: {내용}
- 다음 단계 시작 전 알아야 할 것: {내용}

## 최종 산출물

- **소스**:
  - `types/shared.ts`, `types/board.ts`(수정)
  - `src/lib/codeMap.ts`, `src/lib/constants.ts`, `src/lib/utils/cancer.ts`(수정)
  - `public/icons/index.ts` + 신규 SVG 3종(추가)
  - `DoctorAppDownloadBanner.tsx`(수정)
  - `src/lib/utils/cancer.test.ts`, `src/lib/constants.test.ts`(신규 테스트)
- **문서**: `plans/task_cancer-type_KAN-2170-1.md`(본 문서), `report/task_cancer-type_KAN-2170-1_report.md`(작업 완료 후 작성)
- **통합**: `feature/KAN-2170-1` 브랜치에서 작업 → origin push → 메인 브랜치로 PR. merge·마일스톤은 요청자 영역. 백엔드는 별도 동시 배포 진행이라 이 PR 머지 시점만 신경 쓰면 됨.

## 완료 조건

- [ ] 모든 구현 단계의 완료 기준 충족
- [ ] `pnpm ts-check`, `pnpm lint`, `pnpm test` 전체 통과
- [ ] 4단계에 명시된 4개 화면에서 뇌종양 선택 시 정상 동작 확인
- [ ] 기존 8개 암종 기능 회귀 없음 확인(특히 `src/lib/utils/cancer.ts` switch문 fallback 로직 영향 없는지)
- [ ] 배포 전 백엔드가 `'BRAIN_TUMOR'` 값을 실제로 인식하는지 스테이징에서 최종 확인(동시 배포 계획이지만 순서가 어긋날 경우 대비)
- [ ] 사용자(작업 요청자) 최종 승인

---

## 결정 사항 요약 (전부 확정, 미해결 질문 없음)

1. enum 값 이름: `BRAIN_TUMOR`
2. URL slug 값: `brain-tumor`
3. 커뮤니티 게시판(`BoardCancerCode`) 이번 스코프에 포함
4. 백엔드: 프론트와 동시 배포, 별도 조율 불필요
5. Mixpanel DT 코드: `DT0900`(기존 100단위 증가 패턴의 다음 값)로 잠정 배정 — 분석팀 공식 코드북 확인되면 교체
6. 아이콘: Figma URL 요청 시 제공받아 `scripts/download-figma-image-3x.js`로 다운로드해 사용
