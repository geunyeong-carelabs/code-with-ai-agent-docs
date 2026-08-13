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
  `Record<CancerType, ...>` (또는 `CancerType`을 포함하는 유니온을 키로 하는) 형태로 선언된 지점에서 `tsc` 컴파일 에러 발생 →
  각 에러 지점에 `BRAIN_TUMOR` 항목 수동 추가로 해소 →
  컴파일러가 강제하지 않는 지점(`src/lib/utils/cancer.ts`의 switch 4개, `src/lib/constants.ts`의 `CANCER_TYPE_OPTIONS`류, `VALID_CANCER_COMMUNITY_TYPES`, `types/shared.ts:78-88`의 `cancerTypeParamEnumSchema`, `types/board.ts:21-31`의 `BoardCancerCode`)는 체크리스트 기반으로 수동 갱신 →
  아이콘 SVG 3종(`public/icons/cancer/`, `.../line/`) 추가 및 `public/icons/index.ts` 레지스트리 등록 →
  백엔드가 `'BRAIN_TUMOR'` raw string을 URL/body 파라미터로 수용하는지 확인, Mixpanel `DT` 코드 발급 확인.
  즉 `BASE_CANCER_TYPES` 한 곳만 바꾸면 대부분은 자동/컴파일 에러로 유도되지만, switch-with-default 패턴 4곳과 별도 리터럴 배열 2곳은 컴파일러가 침묵하므로 반드시 체크리스트로 수동 확인해야 한다.

  > **1단계 실행 후 실측 정정(2026-08-13):** 최초 분석 시 `Record<CancerType, ...>` 컴파일 강제 지점을 3파일 4곳(`src/lib/codeMap.ts:18-33`, `src/lib/constants.ts:217-227`, `DoctorAppDownloadBanner.tsx:17-27`)으로 예상했으나, `types/shared.ts`/`types/board.ts` 변경 후 `pnpm ts-check` 실행 결과 아래 3개 지점이 추가로 발견됨(총 6개 지점/10개 파일). 원인 분석 단계에서 코드베이스 전수 검색 없이 진행돼 누락됨. 2단계 스코프에 반영 완료(아래 2단계 작업 내용 참고):
  > - `src/app/hospital/cancer/[cancerTypeParam]/[hospitalId]/_components/constants.ts`의 `HOSPTIAL_CANCER_GRADE_MAP: Record<CancerType, ...>`
  > - `src/components/consultation/constants.ts`의 `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP: Record<ConsultationQuestionType, IconName>`(`ConsultationQuestionType`이 `BASE_CANCER_TYPES`를 스프레드하므로 간접 영향)
  > - `src/app/surgery-review/constants/index.ts`의 `SURGERY_REVIEW_ENUMS`(타입 명시 없는 리터럴 객체, 키가 `CancerTypeEnum`의 9개 값으로 고정돼 있어 이를 `CancerType`으로 인덱싱하는 8개 파일 — `SurgeryReviewTagSection.tsx`, `SurgeryReviewFilterModal.tsx`, `SurgeryReviewFilterSort.tsx`, `SurgeryReviewListContainer.tsx`, `SurgeryReviewListItem.tsx`, `PostSurgeryReviewContainer.tsx`, `Step2Content.tsx`, `Step3Content.tsx` — 에 연쇄적으로 `tsc` 에러 발생)
- **제약 조건**:
  - `BASE_CANCER_TYPES`의 기존 9개 값은 전부 `CANCER_` 접두어(`CANCER_LIVER` 등)이지만, 신규 값은 요청자 확정에 따라 접두어 없이 `BRAIN_TUMOR`로 추가한다(기존 컨벤션과 의도적으로 다름 — 리뷰 시 오타로 오인되지 않도록 PR 설명에 명시). *(정정: 개요 최초 작성 시 "8개"로 오기재됐던 것을 4단계 검증 중 실제 배열을 세어 9개로 정정함 — `CANCER_LIVER`~`CANCER_OTHER`까지 9개 + `BRAIN_TUMOR` = 총 10개)*
  - `BASE_CANCER_TYPES`에서 파생되는 스키마(`cancerTypeWithNoneEnumSchema`, `consultationQuestionTypeEnumSchema` 등)의 파생 구조 자체는 변경하지 않음 — 값만 추가
  - API 호출은 기존과 동일하게 `proxyClient`만 사용, 별도 `fetch`/`axios` 도입 금지
  - `src/lib/utils/cancer.ts`의 switch문에 `default` fallback을 제거하거나 exhaustive 패턴(`never` 체크)으로 리팩터링하는 것은 이번 작업 범위 밖 — 이번엔 기존 패턴대로 case만 추가(리팩터링은 별도 티켓으로 제안)
> **`BoardCancerCode` 값 정정(2026-08-13):** 문서 전반의 `'cancer-brain-tumor'` 예시는 최종적으로 `'brain-tumor'`로 확정됨(접두어 없음, `BASE_CANCER_TYPES`의 `BRAIN_TUMOR`와 동일하게 접두어 생략 컨벤션 통일). 아이콘 파일명도 `cancer-` 접두어 없이 `brain-tumor.svg`/`brain-tumor-line.svg`/`selected-brain-tumor.svg`로 저장. 이 정정 결과, `CancerFilterButton.tsx`/`CancerFilterItem.tsx`는 (3단계에서 `getCancerPathByCancerType()` 기반으로 고쳤던 것을) 원래의 `snakeToKebab(cancerType)` 방식으로 되돌림 — `snakeToKebab('BRAIN_TUMOR')`가 접두어 없는 `brain-tumor`를 그대로 반환해 접두어 없는 파일명과 자연히 일치하기 때문(반면 `CANCER_LIVER` 등은 여전히 `cancer-liver`로 변환되어 기존 파일명과 일치, 회귀 없음). 문서 내 나머지 `'cancer-brain-tumor'` 표기는 이 정정 이전 기록이므로 전부 `'brain-tumor'`로 읽을 것.
>
> **추가 발견 1 — `userCancerTypeStorageAtom.ts`의 하드코딩된 접두어(2026-08-13):** 위 정정 이후에도 실제 브라우저(localStorage에 이미 `BRAIN_TUMOR`가 저장된 상태)로 `/community` 진입 시 여전히 `cancer-brain-tumor`로 요청되는 회귀가 사용자 리포트로 확인됨. 원인은 `src/store/userCancerTypeStorageAtom.ts`의 `useUserCancerType()` 훅 — `boardCode`를 리터럴 문자열이 아니라 `` `cancer-${CANCER_TYPE_TO_PARAM[cancerType]}` `` 템플릿으로 동적 조합하고 있어 문자열 그대로 검색하는 `grep`으로는 안 잡혔음(1단계 실측 정정과 유사한 종류의 "컴파일러도 grep도 못 잡는" 케이스). `CANCER_TYPE_TO_PARAM`은 9개 값 전부 접두어 없는 슬러그(`liver`, ..., `brain-tumor`)를 반환하므로, 이 훅은 항상 `cancer-` 를 강제로 붙여 `brain-tumor` 케이스에서만 어긋남. `snakeToKebab(cancerType)`으로 교체해 해결(`CANCER_TYPE_TO_PARAM` import 제거) — `snakeToKebab`이 `CancerType`의 `CANCER_`/무접두어 여부를 그대로 반영하므로 9개 값 전부 대응하는 `BoardCancerCode`와 정확히 일치. 같은 하위 훅의 `setByBoardCode`(`code.replace('cancer-', '')`)는 애초에 접두어 없는 문자열에 대해서도 no-op이라 별도 수정 불필요했음. 수정 후 `cancer-` 동적 조합/`replace('cancer-', ...)` 패턴 전체 재검색해 다른 누락 지점 없음을 확인.
> **참고**: 브라우저에 예전 값(`cancer-brain-tumor`)이 이미 저장돼 있던 사용자는 localStorage(`userCancerType` 키)를 지우거나 재로그인해야 정정된 값으로 갱신됨.
>
> **추가 발견 2 — `HospitalTypeFilter.tsx`의 기존(unrelated) 버그(2026-08-13):** 사용자가 브라우저 콘솔에서 "React does not recognize the `cancerType` prop on a DOM element" 경고를 리포트. 확인 결과 `src/app/hospital/cancer/[cancerTypeParam]/_components/search-filter/HospitalTypeFilter.tsx`가 `cancerType` prop을 선언만 하고 컴포넌트 내부에서 전혀 사용하지 않은 채 `{...props}`로 최상위 `<div>`에 그대로 흘려보내고 있었음 — **이번 KAN-2170-1 작업과 무관하게 모든 암종(뇌종양 포함 9종 전부)에서 동일하게 발생하던 기존 결함**. 사용자 요청으로 이번 세션에서 같이 수정: 구조분해에서 `cancerType`을 destructure해 `...props` 스프레드에서 제외(`@typescript-eslint/no-unused-vars` 경고 1건 발생하나 기존 코드베이스에도 동일 종류 경고가 여러 건 있어 허용 범위, 빌드/lint는 통과). 이 항목은 계획서 스코프 밖의 별도 결함이라 최종 산출물 목록에는 별도 표기.
>
> **추가 발견 3 — 암종 표시 순서 변경(2026-08-13, 4단계 완료 기준 검증 중 사용자 요청):** 4단계 breakpoint 실측 중 사용자가 "암종 순서는 `기타 암`이 항상 마지막이어야 하고, `뇌종양`은 뒤에서 2번째여야 한다"는 요구사항을 검토 중 발견 — 1~3단계에서 `BRAIN_TUMOR`를 배열/객체 마지막에 단순 추가해온 탓에 실제로는 `기타 암` 뒤에 `뇌종양`이 오는 순서였음(요구사항과 반대). 아래 파일들에서 `CANCER_OTHER`/`brain-tumor` 관련 항목의 선언 순서를 `..., BRAIN_TUMOR, CANCER_OTHER` 순으로 정정:
> - `types/shared.ts`: `BASE_CANCER_TYPES`(소스 오브 트루스), `cancerTypeParamEnumSchema`
> - `types/board.ts`: `boardCancerCodeSchema`
> - `src/lib/constants.ts`: `VALID_CANCER_COMMUNITY_TYPES`, `CANCER_TYPE_OPTIONS`(UI 표시 순서 직접 영향), `COMMUNITY_MENU_ITEMS`(UI 표시 순서 직접 영향), `CANCER_TYPE_TO_PARAM`/`CANCER_PARAM_TO_TYPE`(매핑 객체 키 순서, 값 불변)
> - `src/lib/utils/cancer.ts`: 5개 switch문(`getCancerKorNameByCancerType`, `getCancerIconNameByCancerType`, `getCancerWithPointColorIconNameByCancerType`, `getCancerPathByCancerType`, `getCancerIconNameByBoardCode`)의 case 나열 순서(반환값 불변, 가독성 목적)
> - `src/app/doctor/cancer/[cancerTypeParam]/[doctorId]/_components/DoctorAppDownloadBanner.tsx`(`DOCTOR_COUNT_BY_CANCER_TYPE`), `src/app/hospital/cancer/[cancerTypeParam]/[hospitalId]/_components/constants.ts`(`HOSPTIAL_CANCER_GRADE_MAP`), `src/components/consultation/constants.ts`(`CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP`), `src/app/surgery-review/constants/index.ts`(`SURGERY_REVIEW_ENUMS`) — 4곳 모두 `Record<CancerType, ...>` lookup 객체라 순서가 기능에 영향 없으나 일관성을 위해 함께 정정
> - **의도적으로 변경하지 않은 곳**: `src/lib/codeMap.ts`의 `CANCER_TYPE_CODE_MAP`(Mixpanel `DT` 코드) — `DT0800`=`CANCER_OTHER`, `DT0900`=`BRAIN_TUMOR`는 표시 순서가 아닌 코드 발급 이력을 나타내는 값이라 재배치 대상 아님. 재배열해도 값 자체(코드 문자열)는 그대로이므로 분석 데이터 영향 없음.
> - 수정 후 `pnpm ts-check`(0 errors), `pnpm lint`(신규 에러 없음, 기존 warning 목록과 동일) 확인. Chrome DevTools MCP로 `CancerFilterButton` 바텀시트를 재확인해 `..., 폐암, 뇌종양, 기타 암` 순서로 정상 노출됨을 실측 확인.
>
> **추가 발견 4 — 뇌종양 line 아이콘 색상 버그(2026-08-13, 사용자 리포트):** 다른 9개 `-line` 아이콘은 전부 `fill="currentColor"` 방식이라 선택 시 텍스트 색상(`text-primary` 등)을 따라 자동으로 색이 바뀌는데, `public/icons/cancer/line/brain-tumor-line.svg`만 유일하게 모든 path에 `stroke="#111827"`(하드코딩 고정색)가 박혀 있어 `/find` 암종 선택 등에서 선택돼도 색이 안 바뀌는 회귀 확인(3단계에서 Figma API 직접 호출로 원본 그대로 받은 게 원인 — 다른 8개 색상 아이콘 파일도 확인해보니 정상적으로 고정색이라 문제 없었으나 `-line` 계열만 프로젝트 컨벤션과 달랐음). `stroke="#111827"` → `stroke="currentColor"` 전체 치환(좌표 데이터 불변)으로 해결, 브라우저 실측으로 정상 동작 확인.
>
> **추가 발견 5 — `/find` 검사·치료 장비 스텝 빈 화면(2026-08-13, 사용자 리포트):** 뇌종양은 백엔드에 등록된 검사/치료 장비 데이터가 없어 5·6단계(`선호하는 검사 장비/치료 장비가 있나요?`)에서 옵션 없이 버튼만 있는 빈 화면이 노출됨. 처음엔 스텝 자동 스킵으로 수정했으나 사용자 피드백에 따라 되돌리고, 대신 옵션이 0개일 때 안내 문구("검색에 지원되는 검사장비가 없어요."/"검색에 지원되는 치료장비가 없어요.")를 노출하는 방식으로 변경 — `Stepper.tsx`에 `StepperContent.emptyMessage` 필드 추가, `useFindStepperContent.tsx`의 5·6단계에 각각 문구 지정.

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
- [x] `pnpm ts-check` 실행 시 1단계 변경으로 인한 신규 컴파일 에러 목록을 전수 확인(당초 예상한 4곳과는 불일치 — 아래 완료 메모 참고, 실측 목록을 2단계 스코프로 확정)
- [x] `cancerTypeEnumSchema.parse('BRAIN_TUMOR')`(또는 확정 값)가 통과하는지 로컬 스크립트/REPL로 확인

**단계 완료 메모**
- 이번 단계에서 실제로 한 일:
  - `types/shared.ts:51-61` `BASE_CANCER_TYPES`에 `'BRAIN_TUMOR'` 추가.
  - `types/shared.ts:78-88` `cancerTypeParamEnumSchema`에 `'brain-tumor'` 추가.
  - `types/board.ts:21-31` `boardCancerCodeSchema`에 `'cancer-brain-tumor'` 추가.
  - `types/consultation.ts:44-53`은 `BASE_CANCER_TYPES` 스프레드 구조라 자동 반영됨을 확인, 코드 변경 없음.
  - 변경 전(git stash) `pnpm ts-check` 베이스라인 0건 확인 → 변경 후 재실행하여 신규 에러가 전부 이번 변경에서 기인함을 검증.
  - 임시 vitest 테스트로 `cancerTypeEnumSchema.parse('BRAIN_TUMOR')`, `cancerTypeParamEnumSchema.parse('brain-tumor')`, `boardCancerCodeSchema.parse('cancer-brain-tumor')` 3건 모두 통과 확인 후 테스트 파일 삭제(임시 파일이라 최종 산출물에는 미포함).
- 다음 단계 시작 전 알아야 할 것:
  - 당초 계획서가 예상한 컴파일 에러 지점(3파일 4곳)은 **실제보다 적게 잡힌 것으로 확인됨**. `pnpm ts-check` 실측 결과 총 6개 지점/10개 파일에서 신규 에러 발생 — 위 "1단계 실행 후 실측 정정" 인용문 및 아래 2단계 작업 내용 참고.
  - 특히 `src/app/surgery-review/constants/index.ts`의 `SURGERY_REVIEW_ENUMS`는 명시적 `Record<CancerType, ...>` 타입 주석이 없는 리터럴 객체라 grep/타입 검색으로는 안 잡히고 `tsc` 컴파일로만 드러남 — 2단계 착수 시 유사 패턴(타입 미명시 + `CancerTypeEnum.XXX` 키 사용) 추가 존재 가능성을 염두에 두고 `pnpm ts-check` 결과를 1차 소스로 삼을 것.
  - `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP`(`src/components/consultation/constants.ts`)은 값 타입이 `IconName`이라 3단계에서 아이콘 SVG 등록 전에는 최종 아이콘명을 넣을 수 없음 — 2단계에서는 컴파일 통과용 임시값(예: 기존 `'selectedCancerOther'` 재사용)으로 채우고, 3단계에서 실제 아이콘 등록 후 해당 값만 교체하는 2단계 작업 순서로 처리할 것.

### 2단계 — 컴파일러가 강제하는 매핑 갱신

**작업 내용**
1. `src/lib/codeMap.ts:18-28` `CANCER_TYPE_CODE_MAP`에 `[CancerTypeEnum.BRAIN_TUMOR]: 'DT0900'` 추가(분석팀 공식 코드북 확인 시 값 교체).
2. `src/lib/codeMap.ts:30-33` `MEMBER_CANCER_TYPE_CODE_MAP`에 동일 반영(스프레드라 1번만 하면 자동 포함, 확인만).
3. `src/lib/constants.ts:217-227` `CANCER_TYPE_TO_PARAM`에 `brain-tumor` 매핑 추가.
4. `src/lib/constants.ts:229-239` `CANCER_PARAM_TO_TYPE`은 키가 `CancerTypeParam`이라 자동 에러는 안 나지만, 역매핑 누락 방지를 위해 함께 추가.
5. `src/app/doctor/cancer/[cancerTypeParam]/[doctorId]/_components/DoctorAppDownloadBanner.tsx:17-27` `DOCTOR_COUNT_BY_CANCER_TYPE`에 초기값(0 또는 실제 집계값) 추가.
6. **(1단계 실측 정정으로 추가)** `src/app/hospital/cancer/[cancerTypeParam]/[hospitalId]/_components/constants.ts`의 `HOSPTIAL_CANCER_GRADE_MAP: Record<CancerType, keyof typeof HOSPITAL_GRADE_INFO_MAP | null>`에 `BRAIN_TUMOR: null` 추가(기존 값 없는 암종과 동일 패턴 — 병원 평가 등급 데이터 없으므로 `null`).
7. **(1단계 실측 정정으로 추가)** `src/components/consultation/constants.ts`의 `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP: Record<ConsultationQuestionType, IconName>`에 `[ConsultationQuestionTypeEnum.BRAIN_TUMOR]` 추가 — 3단계에서 아이콘 SVG/`IconName` 등록 전이므로 임시로 기존 값(예: `'selectedCancerOther'`) 재사용, 3단계 완료 후 실제 등록된 아이콘명으로 교체(TODO 주석 남길 것).
8. **(1단계 실측 정정으로 추가)** `src/app/surgery-review/constants/index.ts`의 `SURGERY_REVIEW_ENUMS`에 `[CancerTypeEnum.BRAIN_TUMOR]` 항목 추가 — 기존 유방암 외 암종과 동일하게 `{ DEPENDENCY_RULES: null, DEPENDENCY_TRIGGERS: null, ENUM_GROUPS: [], SCHEDULE_GROUPS: [], TAG_GROUPS: [], TAG_GROUPS_FOR_DETAIL_PAGE: null }` 플레이스홀더로 추가(수술후기 세부 필터는 유방암 전용 기능이라 뇌종양은 빈 값이 정상).

**완료 기준**
- [x] `pnpm ts-check` 통과(신규 컴파일 에러 0건 — 6번/7번/8번 항목 포함 총 6개 지점/10개 파일 전부 해소 확인)
- [x] `pnpm build` 성공

**단계 완료 메모**
- 이번 단계에서 실제로 한 일:
  - 1~5번(당초 계획): `codeMap.ts`(`CANCER_TYPE_CODE_MAP` `BRAIN_TUMOR: 'DT0900'`, `MEMBER_CANCER_TYPE_CODE_MAP`은 스프레드로 자동 반영 확인), `constants.ts`(`CANCER_TYPE_TO_PARAM`/`CANCER_PARAM_TO_TYPE`에 `brain-tumor` 양방향 추가), `DoctorAppDownloadBanner.tsx`(`DOCTOR_COUNT_BY_CANCER_TYPE`에 `0` 추가).
  - 6~8번(1단계 실측 정정 반영): `HOSPTIAL_CANCER_GRADE_MAP`에 `BRAIN_TUMOR: null`, `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP`에 `BRAIN_TUMOR: 'selectedCancerOther'`(TODO 주석으로 3단계 교체 예정 표시), `SURGERY_REVIEW_ENUMS`에 `BRAIN_TUMOR` null/빈 배열 플레이스홀더 추가.
  - `pnpm ts-check` 재실행하여 신규 컴파일 에러 0건 확인, `pnpm build` 전체 성공 확인(6개 지점/10개 파일 전부 해소).
- 다음 단계 시작 전 알아야 할 것:
  - `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP`의 `BRAIN_TUMOR` 값은 임시로 `'selectedCancerOther'`를 재사용 중 — 3단계에서 아이콘 SVG 3종 등록 후 실제 아이콘명(예: `selectedCancerBrainTumor`)으로 교체 필수. 해당 위치에 `TODO(KAN-2170-1 3단계)` 주석 남겨둠.
  - `SURGERY_REVIEW_ENUMS`의 `BRAIN_TUMOR` 항목은 유방암 외 8개 암종과 동일하게 전부 `null`/`[]` 플레이스홀더 — 수술후기 세부 필터가 유방암 전용 기능이라 정상이며, 향후 뇌종양 수술후기 필터를 별도로 만들 계획이 있다면 이 파일의 유방암 전용 로직(`BREAST_CANCER_SURGERY_*`)을 참고해 별도 티켓으로 진행해야 함(이번 스코프 아님).
  - 3단계는 `src/lib/utils/cancer.ts`의 switch 4개, `CANCER_TYPE_OPTIONS`류, `VALID_CANCER_COMMUNITY_TYPES`, 아이콘 SVG 등록이 남아 있음. 아이콘 등록이 끝나야 위 `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP` TODO를 마저 해소할 수 있으므로, 3단계 내에서 아이콘 등록을 먼저 완료한 뒤 이 TODO를 정리하는 순서를 권장.

### 3단계 — 컴파일러 미검증 매핑/자산 수동 갱신

**작업 내용**
1. `src/lib/utils/cancer.ts:11-34` `getCancerKorNameByCancerType`에 한글 라벨(예: "뇌종양") case 추가.
2. `src/lib/utils/cancer.ts:43-68` `getCancerIconNameByCancerType`, `:70-95` `getCancerWithPointColorIconNameByCancerType`에 신규 아이콘명 case 추가(아이콘 자산은 아래 4항목과 병행).
3. `src/lib/utils/cancer.ts:130-153` `getCancerPathByCancerType`에 영문 경로 slug case 추가.
4. Figma URL 수령 후 `node scripts/download-figma-image-3x.js <node-id>`로 아이콘 3종(line/filled/selected) 다운로드 → `public/icons/cancer/`, `public/icons/cancer/line/`에 배치 → `public/icons/index.ts` 임포트 및 `IconName` 레지스트리에 등록.
5. `src/lib/constants.ts:105-129` `CANCER_TYPE_OPTIONS`, `MEMBER_CANCER_TYPE_OPTIONS`에 옵션 항목 추가(`CANCER_TYPE_MAP` 등은 파생이므로 자동 반영 확인만).
6. `src/lib/constants.ts:86-96` `VALID_CANCER_COMMUNITY_TYPES`에 `'cancer-brain-tumor'` 코드 추가(커뮤니티 게시판 포함 확정에 따라 필수).

**완료 기준**
- [x] `src/lib/utils/cancer.ts`의 4개 함수 모두 신규 값에 대해 fallback이 아닌 실제 매핑값 반환(단위 테스트 또는 콘솔 확인)
- [x] 신규 아이콘 3종이 브라우저에서 정상 렌더링(경로 오타/404 없음)
- [x] `pnpm lint` 통과

**단계 완료 메모**
- 이번 단계에서 실제로 한 일:
  - `getCancerKorNameByCancerType`에 `'뇌종양'`, `getCancerPathByCancerType`에 `'brain-tumor'` case 추가.
  - `CANCER_TYPE_OPTIONS`에 `{ label: '뇌종양', value: BRAIN_TUMOR }` 추가(`MEMBER_CANCER_TYPE_OPTIONS`/`CANCER_TYPE_MAP`은 스프레드/파생이라 자동 반영), `VALID_CANCER_COMMUNITY_TYPES`에 `'cancer-brain-tumor'` 추가.
  - 아이콘 3종: 사용자가 제공한 Figma node-id 3개(filled `26740-39411`, line `26662-30559`, selected `26662-30558`)를 프로젝트 공용 `scripts/download-figma-image-3x.js`가 아닌 **Figma API `format=svg` 직접 호출**로 받음(사용자 확인 후 진행) — 기존 8개 암종 아이콘이 전부 `@svgr/webpack` 기반 진짜 벡터 SVG인데, 공용 스크립트는 PNG→WebP 래스터만 지원해 호환되지 않았기 때문. 다운로드 원본에 있던 불투명 흰색 배경 `<rect>`(Figma 프레임 배경 export 부산물, 다른 8개 파일 어디에도 없음)를 제거하고 `width`/`height`를 기존 관례대로 `100%`로 정규화. `public/icons/cancer/cancer-brain-tumor.svg`, `public/icons/cancer/line/cancer-brain-tumor-line.svg`, `public/icons/cancer/selected-cancer-brain-tumor.svg`로 저장 후 `public/icons/index.ts`에 `cancerBrainTumor`/`cancerBrainTumorLine`/`selectedCancerBrainTumor` 등록.
  - `getCancerIconNameByCancerType`/`getCancerWithPointColorIconNameByCancerType`에 케이스 추가, 그리고 **계획서에 없던 추가 발견**: `getCancerIconNameByBoardCode`(`BoardCancerCode` 스위치, 4개 함수 집계에 포함 안 됐던 5번째 switch)에도 `cancer-brain-tumor` 케이스 추가.
  - 2단계에서 임시값(`selectedCancerOther`)으로 채웠던 `CONSULTATION_QUESTION_TYPE_ICON_NAME_MAP`의 `BRAIN_TUMOR`를 실제 `selectedCancerBrainTumor`로 교체, TODO 주석 제거.
  - **브라우저 실측으로 추가 버그 발견 및 수정(계획서 미기재, 4단계 대상 파일이지만 3단계 아이콘 검증 중 발견)**: `src/app/hospital/cancer/[cancerTypeParam]/_components/cancer-filter/CancerFilterButton.tsx`와 `CancerFilterItem.tsx`가 아이콘 경로를 `Icons`/`IconName` 레지스트리가 아니라 `snakeToKebab(cancerType)`으로 직접 조립(`CANCER_LIVER` → `cancer-liver`)하고 있었음. `BRAIN_TUMOR`는 확정된 설계대로 `CANCER_` 접두어가 없어 `snakeToKebab('BRAIN_TUMOR')`가 `brain-tumor`가 되어 `cancer-` 접두어가 빠진 잘못된 경로(`/icons/cancer/brain-tumor.svg`)로 요청 → 404 → 아이콘 깨짐(브라우저 스크린샷으로 확인). `snakeToKebab` 대신 이미 올바르게 매핑돼 있는 `getCancerPathByCancerType()`을 사용하도록 두 파일을 수정(기존 8개 암종은 동일한 결과값이라 회귀 없음, `BRAIN_TUMOR`만 정상화됨). Chrome DevTools로 `/hospital/cancer/liver` → 암종 필터 드로어에서 뇌종양 선택까지 재검증해 정상 렌더링 확인.
  - `pnpm ts-check`, `pnpm lint` 재실행 통과 확인.
- 다음 단계 시작 전 알아야 할 것:
  - 4단계 작업 내용 2번(`CancerFilterButton.tsx`)은 이미 이번 단계에서 수정 완료됨 — 4단계에서는 회귀 여부만 재확인하면 됨(추가 수정 불필요).
  - `snakeToKebab` 기반 경로 조립 패턴이 이번에 발견된 2개 파일 외에 더 있을 수 있음 — 4단계 UI 확인 중 다른 화면에서도 깨진 아이콘이 보이면 동일한 원인(하드코딩 문자열 조립 vs `Icons` 레지스트리/`getCancerPathByCancerType` 사용) 의심할 것.
  - 뇌종양 filled 아이콘은 최초 다운로드 후 사용자가 동일 node-id(`26740-39411`)로 재교체 요청 — 재다운로드 결과 바이트 동일(디자인 변경 없었음), `width`/`height` 정규화만 재적용.

### 4단계 — UI 렌더링 확인

**작업 내용**
1. `src/components/drawer/CancerTypeDrawer.tsx` — 온보딩/홈 암종 선택 드로어에 뇌종양 항목 노출 확인.
2. `src/app/hospital/cancer/[cancerTypeParam]/_components/cancer-filter/CancerFilterButton.tsx` — 병원 검색 필터 탭 노출 확인. *(3단계에서 아이콘 검증 중 버그 발견·수정 완료 — `CancerFilterItem.tsx` 포함, 위 3단계 완료 메모 참고. 이 단계에서는 회귀만 재확인)*
3. `src/app/consultation/cancer/_components/consultaion-list/ConsultationCancerTypeFilter.tsx` — 상담 목록 필터 노출 확인.
4. `src/app/auth/sign-up/_components/SignUpStep.tsx`, `src/app/my-page/info/_components/edit-info/EditMyInfoForm.tsx` — 회원가입/프로필 수정 시 뇌종양 선택 후 저장·Mixpanel 전송 정상 동작 확인.
5. `src/app/community/_components/community-filter/CancerButton.tsx`, `CommunityHomeSelectFilter.tsx`, `src/app/community/[boardCode]/_components/community-filter/CommunitySelectFilter.tsx` — 뇌종양 커뮤니티 게시판 필터 노출 및 `/community/cancer-brain-tumor` 라우트 정상 진입 확인(`VALID_CANCER_COMMUNITY_TYPES` 반영 여부 포함).

**완료 기준**
- [x] 위 4개 화면 — 1번(`CancerTypeDrawer`), 2번(`CancerFilterButton`/`Item`), 5번(커뮤니티 선택)은 실제 브라우저(Chrome DevTools MCP)로 뇌종양 선택/필터링/라우트 진입까지 확인. 3번(`ConsultationCancerTypeFilter`), 4번(`SignUpStep`/`EditMyInfoForm`)은 텍스트 라벨 전용 컴포넌트(`CANCER_TYPE_OPTIONS`/`MEMBER_CANCER_TYPE_OPTIONS` 파생, 아이콘·경로 조립 없음)라 코드 리뷰로 안전성 확인, 실브라우저 클릭 검증은 생략(비용 절감을 위해 사용자와 합의)
- [x] 320/768/1024/1440 breakpoint 전수 스크린샷 — **실측 완료(2026-08-13)**. `CancerFilterButton`의 `BottomSheet`(`grid grid-cols-3`, 10개 옵션)를 Chrome DevTools MCP `emulate` 뷰포트로 4개 breakpoint 모두 확인, 가로 스크롤/줄바꿈 깨짐 없음(`document.documentElement.scrollWidth`가 매 breakpoint에서 `window.innerWidth`와 일치).
  - **테스트 방법 관련 주의사항**: `resize_page` 도구는 Chrome의 최소 창 폭 제약(약 500px)에 걸려 320px 요청이 실제로는 500px로 렌더링됨 — 이 방식으로는 320px 실측이 불가능. `emulate({ viewport: '320x900x2,mobile,touch' })`로 전환해야 실제 320px 뷰포트 강제 가능(향후 320px 검증 시 `resize_page` 대신 `emulate` 사용할 것).
  - **테스트 아티팩트(실제 버그 아님)**: 320px에서 Next.js 개발 모드 플로팅 배지(`<nextjs-portal>`, "Open Next.js Dev Tools")가 화면 좌하단 고정 위치에 떠 있어 그리드 마지막 줄(순서 변경 후 마지막 항목인 `기타 암`)과 스크린샷상 겹쳐 보임. 프로덕션 빌드에는 존재하지 않는 dev-only 오버레이이므로 실제 결함 아님(`portal.style.display='none'`으로 숨기고 재확인해 정상 렌더링 확인).
  - `CancerTypeDrawer.tsx`(홈 온보딩 드로어)는 동일한 `grid-cols-3` 구조이나 "로그인 + 암종 미설정" 상태에서만 트리거되어 이번 세션에서는 실브라우저 검증 생략 — 구조가 `CancerFilterButton`과 동일(고정 3열, `max-w-screen-mobile` 캡, 반응형 분기 없음)하여 회귀 위험 낮음으로 판단, 필요시 QA 단계에서 실측 권장.

**단계 완료 메모**
- 이번 단계에서 실제로 한 일:
  - `COMMUNITY_MENU_ITEMS`(리터럴 배열, 컴파일러 미검증)에 뇌종양 항목 누락 발견·추가 — 이게 없으면 "커뮤니티 선택" 바텀시트에 뇌종양이 아예 안 뜨고, `/community/cancer-brain-tumor` 직접 진입 시 선택된 게시판 제목도 깨짐.
  - 백엔드 API 실측: `GET /board/config/{boardCode}` 계열(`/board/content/general·best·like·recommend/{boardCode}`)은 신규 `boardCode`를 아직 인식 못해 400 에러(브라우저 실제 세션으로 `cancer-liver` 200 vs `cancer-brain-tumor` 400 비교 확인) — 백엔드팀 동시 배포 필요 부분, 계획서 완료 조건에 이미 명시된 항목. `GET /hospital/content/{cancerType}/search`는 200이나 병원 수가 기존 암종과 동일(374개)해 `cancerType` 필터링이 실제로 적용 안 되는 것으로 추정 — 이것도 백엔드 확인 필요.
  - 사용자 요청으로 `boardCode` 값을 `'cancer-brain-tumor'` → `'brain-tumor'`(접두어 없음)로 재확정 — `types/board.ts`, `constants.ts`(`VALID_CANCER_COMMUNITY_TYPES`, `COMMUNITY_MENU_ITEMS`), `cancer.ts`(`getCancerIconNameByBoardCode`) 수정. 아이콘 3종 파일명도 `cancer-` 접두어 제거(`brain-tumor.svg` 등)하고 `CancerFilterButton`/`Item`을 원래의 `snakeToKebab` 방식으로 되돌림(위 "결정 사항" 섹션 정정 인용 참고).
  - **회귀 발견·수정**: `src/store/userCancerTypeStorageAtom.ts`의 `boardCode` 파생이 `` `cancer-${param}` `` 템플릿으로 하드코딩돼 있어 boardCode 재확정 이후에도 실제로는 여전히 `cancer-brain-tumor`가 만들어지던 문제(문자열 리터럴이 아니라 grep에 안 걸림) → `snakeToKebab(cancerType)`로 교체해 해결.
  - **스코프 밖 별도 결함 수정**(사용자 요청): `HospitalTypeFilter.tsx`가 쓰지도 않는 `cancerType` prop을 DOM에 그대로 흘려보내던 기존 버그(9개 암종 전부 동일 발생, 이번 작업과 무관) — 구조분해로 제외해 수정.
  - 매 수정마다 `pnpm ts-check`(전부 0 errors), `pnpm lint`(신규 warning 1건 — 의도된 구조분해 제외 패턴) 확인.
- 다음 단계 시작 전 알아야 할 것:
  - 브라우저 localStorage에 예전 값(`cancer-brain-tumor`)이 이미 저장된 테스트 계정/기기는 `userCancerType` 키를 지우거나 재로그인해야 정정된 값을 받음.
  - 백엔드 확인 필요 API 2건(위 참고) — 5단계 테스트 작성이나 배포 전 최종 확인 때 반드시 재확인.
  - 320/768/1024/1440 breakpoint 실측은 아직 미완 — 5단계 진행 전이나 QA 단계에서 채워야 함.

### 5단계 — 테스트 추가(회귀 방지)

**작업 내용**
1. ~~`src/lib/utils/cancer.test.ts`(신규)~~, ~~`src/lib/constants.test.ts`(신규)~~ — **사용자 요청으로 파일 2개 대신 `src/lib/cancerType.test.ts` 1개로 통합**(작업 내용 정정, 아래 완료 메모 참고).

**완료 기준**
- [x] `pnpm test` 신규 테스트 전체 통과 — `cancerType.test.ts` 83개 테스트 전부 통과(기존 nursing hospital 테스트 12개 실패는 이번 세션에서 건드리지 않은 사전 존재 실패, `git status`로 무관함 확인)
- [x] `pnpm coverage` 기존 커버리지 대비 하락 없음 — `@vitest/coverage-v8` 미설치 상태였어서 설치 후 재확인(아래 완료 메모 참고). 전체 리포지토리 `pnpm coverage`는 위 사전 존재 nursing 실패 때문에 실행 자체가 중단돼 리포트가 안 나옴(이번 작업 스코프 밖 — 별도 티켓으로 제안). `cancerType.test.ts` 단독 스코프로 커버리지 실행 결과 `src/lib/constants.ts` 100%, `types/shared.ts` 93.75%, `src/lib/utils/cancer.ts` 63.41%(테스트 대상 4개 함수 외 나머지 함수는 이번 스코프 밖이라 미커버 — 정상)

**단계 완료 메모**
- 이번 단계에서 실제로 한 일:
  - 원래 계획(파일 2개 분리)을 사용자가 검토하며 "오늘 작업한 것(순서 변경, 아이콘 색상 버그, 아이콘 레지스트리 등록)을 다 잡아주는지" 질문 → 분석 결과 원안은 스키마 파싱/순서 불변식/아이콘 레지스트리 등록 여부를 전혀 커버하지 못함을 확인, 사용자 요청으로 `src/lib/cancerType.test.ts` 하나로 통합하며 범위 확장(SVG 파일 자체의 색상 속성 정적 검사는 사용자 요청으로 제외).
  - 최종 구성(5개 `describe` 블록, 83개 테스트):
    1. 스키마 & 소스오브트루스 — `cancerTypeEnumSchema`/`cancerTypeParamEnumSchema`/`boardCancerCodeSchema` 파싱 및 `BASE_CANCER_TYPES`와의 개수 일치
    2. 순서 불변식 — `BASE_CANCER_TYPES`/`CANCER_TYPE_OPTIONS`/`VALID_CANCER_COMMUNITY_TYPES`/`COMMUNITY_MENU_ITEMS` 전부 마지막이 `CANCER_OTHER`, 그 바로 앞이 `BRAIN_TUMOR`인지 검증(오늘 있었던 순서 정정 작업의 회귀 방지)
    3. `cancer.ts` 4개 함수 값 검증 — `Record<CancerType, string>`으로 기대값을 선언해, 향후 새 암종이 `BASE_CANCER_TYPES`에 추가되면 이 파일이 **타입 컴파일 에러**로 막혀 값을 채우기 전엔 `pnpm ts-check`/`pnpm test`가 통과 안 되도록 강제(사람이 정답을 정해야 하는 영역이라 완전 자동화 대신 강제 입력 방식 채택)
    4. 매핑 커버리지 — `CANCER_TYPE_OPTIONS`/`CANCER_TYPE_TO_PARAM`/`CANCER_PARAM_TO_TYPE`(역매핑 라운드트립 포함)/`VALID_CANCER_COMMUNITY_TYPES`/`COMMUNITY_MENU_ITEMS`가 `BASE_CANCER_TYPES`와 개수·멤버십 일치
    5. 아이콘 레지스트리 등록 여부 — 하드코딩된 아이콘명이 아니라 `getCancerIconNameByCancerType`/`getCancerWithPointColorIconNameByCancerType`의 **반환값**을 그대로 사용해 `Icons` 레지스트리에 존재하는지 검증 → 3번 항목에서 새 암종의 아이콘명을 채우기만 하면 이 검증도 자동으로 같이 확장됨(하드코딩 없이 완전 동적)
  - `@vitest/coverage-v8`가 `package.json`에 아예 없어 `pnpm coverage` 실행 자체가 안 되던 상태 발견 → 설치된 `vitest@4.0.17`과 버전 일치시켜 `@vitest/coverage-v8@4.0.17` 추가 설치(`package.json` devDependencies에 반영).
  - 설치 후 전체 리포지토리 `pnpm coverage` 실행 시 기존(사전 존재) nursing hospital 테스트 12개 실패로 러너가 비정상 종료돼 커버리지 리포트가 생성되지 않음 확인 — `git status`로 해당 테스트/컴포넌트 파일들이 이번 세션에서 전혀 수정되지 않았음을 확인해 무관함을 검증. `pnpm vitest run --coverage src/lib/cancerType.test.ts`로 스코프를 좁혀 커버리지 도구 자체는 정상 동작함을 확인(위 완료 기준 항목 참고).
- 다음 단계 시작 전 알아야 할 것:
  - 전체 리포지토리 `pnpm coverage`가 실행 안 되는 사전 존재 문제(nursing hospital 테스트 12개 실패)는 이번 KAN-2170-1 스코프 밖 — 별도 티켓으로 리포트하는 것을 권장(계획서 완료 조건의 "pnpm test 전체 통과"는 이 사전 존재 실패로 인해 리포지토리 전체 기준으로는 충족 안 됨, 신규 추가분 기준으로는 충족).
  - `package.json`에 `@vitest/coverage-v8` devDependency 추가됨 — PR diff에 포함되므로 리뷰 시 "왜 이 패키지가 추가됐는지" 설명 필요(커버리지 도구 자체가 누락돼 있던 사전 존재 이슈를 이번에 메꿈).

## 최종 산출물

- **소스**:
  - `types/shared.ts`, `types/board.ts`(수정)
  - `src/lib/codeMap.ts`, `src/lib/constants.ts`, `src/lib/utils/cancer.ts`(수정)
  - `public/icons/index.ts` + 신규 SVG 3종(추가)
  - `DoctorAppDownloadBanner.tsx`(수정)
  - `src/app/hospital/cancer/[cancerTypeParam]/[hospitalId]/_components/constants.ts`(수정, 1단계 실측 정정으로 추가)
  - `src/components/consultation/constants.ts`(수정, 1단계 실측 정정으로 추가)
  - `src/app/surgery-review/constants/index.ts`(수정, 1단계 실측 정정으로 추가)
  - `src/store/userCancerTypeStorageAtom.ts`(수정, 4단계 회귀 발견으로 추가)
  - `src/app/hospital/cancer/[cancerTypeParam]/_components/search-filter/HospitalTypeFilter.tsx`(수정, 스코프 밖 별도 결함)
  - `public/icons/cancer/line/brain-tumor-line.svg`(수정, 4단계 사용자 리포트 — line 아이콘 색상 버그)
  - `src/components/stepper/Stepper.tsx`, `src/app/find/hook/useFindStepperContent.tsx`(수정, 4단계 사용자 리포트 — 빈 옵션 안내 문구)
  - `src/lib/cancerType.test.ts`(신규 테스트, 5단계 — 원래 계획한 `cancer.test.ts`+`constants.test.ts` 2개에서 1개로 통합)
  - `package.json`(수정, 5단계 — `@vitest/coverage-v8` devDependency 추가)
- **문서**: `plans/task_cancer-type_KAN-2170-1.md`(본 문서), `report/task_cancer-type_KAN-2170-1_report.md`(작업 완료 후 작성), [암종(CancerType) 추가 작업 가이드](https://carelabs-healo.atlassian.net/wiki/spaces/HO/pages/302710791)(Confluence, 향후 암종 추가 작업 표준 절차)
- **통합**: `feature/KAN-2170-1` 브랜치에서 작업 → origin push → 메인 브랜치로 PR. merge·마일스톤은 요청자 영역. 백엔드는 별도 동시 배포 진행이라 이 PR 머지 시점만 신경 쓰면 됨.

## 완료 조건

- [x] 모든 구현 단계의 완료 기준 충족(1~5단계)
- [x] `pnpm ts-check`, `pnpm lint` 전체 통과. `pnpm test`는 신규/수정분(전부 통과) 기준 충족하나, 리포지토리 전체 기준으로는 이번 세션과 무관한 사전 존재 nursing hospital 테스트 12개 실패가 있어 완전한 "전체 통과"는 아님(위 5단계 완료 메모 참고, 별도 티켓 권장)
- [x] 4단계에 명시된 4개 화면에서 뇌종양 선택 시 정상 동작 확인(+ 4단계 중 발견된 line 아이콘 색상 버그, `/find` 빈 옵션 화면 수정 포함)
- [x] 기존 9개 암종 기능 회귀 없음 확인(특히 `src/lib/utils/cancer.ts` switch문 fallback 로직, `SURGERY_REVIEW_ENUMS`의 유방암 전용 필터 로직 영향 없는지) — `cancerType.test.ts`로 회귀 테스트 커버
- [x] 배포 전 백엔드가 `'BRAIN_TUMOR'` 값을 실제로 인식하는지 스테이징에서 최종 확인(동시 배포 계획이지만 순서가 어긋날 경우 대비) — 사용자 확인 완료(2026-08-13)
- [x] 사용자(작업 요청자) 최종 승인

---

## 결정 사항 요약 (전부 확정, 미해결 질문 없음)

1. enum 값 이름: `BRAIN_TUMOR`
2. URL slug 값: `brain-tumor`
3. 커뮤니티 게시판(`BoardCancerCode`) 이번 스코프에 포함
4. 백엔드: 프론트와 동시 배포, 별도 조율 불필요
5. Mixpanel DT 코드: `DT0900`(기존 100단위 증가 패턴의 다음 값)로 잠정 배정 — 분석팀 공식 코드북 확인되면 교체
6. 아이콘: Figma URL 요청 시 제공받아 `scripts/download-figma-image-3x.js`로 다운로드해 사용
