---
title: "클론 프로젝트 요구사항 정의서"
date: 2026-01-03 14:22:00 +0900
categories: [cs, project]
tags: [requirements, spring-boot, jpa]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 프로젝트명 | HoHo (클론 프로젝트) |
| 설명 | 개인 맞춤형 복지 정책 추천 서비스 |
| 참고 사이트 | https://www.example.com |
| 기술 스택 | Spring Boot 3, JPA, MySQL, QueryDSL |
| Java 버전 | 17 |
| 작성일 | 2026-01-01 |
| 버전 | v1.0 |

---

## 기술 스택 상세

| 구분 | 기술 | 버전 |
| --- | --- | --- |
| Framework | Spring Boot | 3.5.x |
| ORM | Spring Data JPA | - |
| Query | QueryDSL | 5.x |
| Database | MySQL | 8.x |
| Security | Spring Security + JWT | - |
| Build | Gradle (Groovy) | - |
| Java | OpenJDK | 17 |
| UUID | uuid-creator (UUID v7) | 5.3.3 |

---

## 🎯 핵심 기능 목록

### 1. 회원 관리 (Auth)

| 기능 ID | 기능명 | 설명 | 우선순위 |
| --- | --- | --- | --- |
| AUTH-001 | 회원가입 | 이메일/비밀번호로 회원가입 | 🔴 높음 |
| AUTH-002 | 로그인 | JWT 기반 로그인 (Access + Refresh Token) | 🔴 높음 |
| AUTH-003 | 로그아웃 | Refresh Token 무효화 | 🔴 높음 |
| AUTH-004 | 토큰 재발급 | Refresh Token으로 Access Token 재발급 | 🔴 높음 |
| AUTH-005 | 회원탈퇴 | Soft Delete 처리 (deletedAt) | 🟡 중간 |

### 2. 내 정보 관리 (Member Profile)

| 기능 ID | 기능명 | 설명 | 우선순위 |
| --- | --- | --- | --- |
| PROFILE-001 | 기본 정보 등록 | 지역, 성별, 생년월일 | 🔴 높음 |
| PROFILE-002 | 학력 정보 등록 | 최종학력, 재학상태 | 🔴 높음 |
| PROFILE-003 | 직장 정보 등록 | 고용형태, 직장유무, 기업유형 | 🔴 높음 |
| PROFILE-004 | 가구 정보 등록 | 가구유형, 가구원 수, 월 소득 | 🔴 높음 |
| PROFILE-005 | 결혼/자녀 정보 | 결혼여부, 자녀 수 | 🔴 높음 |
| PROFILE-006 | 내 정보 전체 수정 | 등록된 정보 일괄 수정 | 🟡 중간 |
| PROFILE-007 | 내 정보 조회 | 등록된 정보 조회 | 🔴 높음 |

### 3. 정책 관리 (Policy)

| 기능 ID | 기능명 | 설명 | 우선순위 |
| --- | --- | --- | --- |
| POLICY-001 | 정책 목록 조회 | 전체 정책 목록 (페이징, 필터링) | 🔴 높음 |
| POLICY-002 | 신청 가능 정책 | status=ACTIVE 정책 목록 | 🔴 높음 |
| POLICY-003 | 마감된 정책 | status=CLOSED 정책 목록 | 🟡 중간 |
| POLICY-004 | 신청 예정 정책 | status=UPCOMING 정책 목록 | 🟡 중간 |
| POLICY-005 | 맞춤 정책 추천 | 내 프로필 기반 정책 추천 | 🔴 높음 |
| POLICY-006 | 정책 상세 조회 | 정책 상세 + 진행단계 + 서류 + 파일 | 🔴 높음 |
| POLICY-007 | 정책 검색 | 키워드, 지역, 카테고리 검색 | 🟡 중간 |
| POLICY-008 | 정책 스크랩 | 관심 정책 저장 | 🟡 중간 |
| POLICY-009 | 스크랩 취소 | 저장한 정책 삭제 | 🟡 중간 |
| POLICY-010 | 내 스크랩 목록 | 저장한 정책 목록 조회 | 🟡 중간 |

### 4. 커뮤니티 (Community)

| 기능 ID | 기능명 | 설명 | 우선순위 |
| --- | --- | --- | --- |
| COMMUNITY-001 | 게시글 작성 | 커뮤니티 글 작성 | 🟡 중간 |
| COMMUNITY-002 | 게시글 목록 | 게시글 목록 조회 (페이징) | 🟡 중간 |
| COMMUNITY-003 | 게시글 상세 | 게시글 상세 조회 + 조회수 증가 | 🟡 중간 |
| COMMUNITY-004 | 게시글 수정 | 본인 글 수정 | 🟢 낮음 |
| COMMUNITY-005 | 게시글 삭제 | 본인 글 삭제 (Soft Delete) | 🟢 낮음 |
| COMMUNITY-006 | 댓글 작성 | 게시글에 댓글 작성 | 🟢 낮음 |
| COMMUNITY-007 | 댓글 삭제 | 본인 댓글 삭제 | 🟢 낮음 |
| COMMUNITY-008 | 내 활동 내역 | 내가 작성한 글/댓글 목록 | 🟡 중간 |

### 5. 출석 체크 (Attendance)

| 기능 ID | 기능명 | 설명 | 우선순위 |
| --- | --- | --- | --- |
| ATTEND-001 | 출석 체크 | 일일 출석 체크 (1일 1회) | 🟡 중간 |
| ATTEND-002 | 월별 출석 현황 | 월별 출석 캘린더 조회 | 🟡 중간 |
| ATTEND-003 | 연속 출석일 조회 | 연속 출석일 수 계산 | 🟢 낮음 |
| ATTEND-004 | 총 출석일 조회 | 전체 출석 횟수 조회 | 🟢 낮음 |

---

## 데이터 모델 (Entity 설계)

### PK 전략

| 구분 | PK 전략 | 적용 Entity | 이유 |
| --- | --- | --- | --- |
| 민감 데이터 | **UUID v7** | Member, MemberProfile, PolicyScrap, Post, Comment, Attendance | 보안, 예측 불가 |
| 외부/일반 데이터 | **Auto Increment** | Policy, PolicyProcess, PolicyDocument, PolicyFile, PolicyCategory | 조회 성능 |

---

### 1. Member (회원)

> PK: UUID v7

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| email | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 |
| password | VARCHAR(200) | NOT NULL | 암호화된 비밀번호 |
| name | VARCHAR(50) |  | 이름 |
| role | ENUM | NOT NULL, DEFAULT 'USER' | 권한 |
| created_at | DATETIME | NOT NULL | 가입일 |
| updated_at | DATETIME |  | 수정일 |
| deleted_at | DATETIME |  | 탈퇴일 (Soft Delete) |

**MemberRole (Enum)**

- `USER` - 일반 사용자
- `ADMIN` - 관리자

---

### 2. MemberProfile (회원 프로필)

> PK: UUID v7 \| FK: member_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| member_id | UUID | FK, UNIQUE | 회원 ID |
| region | VARCHAR(100) |  | 지역 |
| gender | ENUM |  | 성별 |
| birth_date | DATE |  | 생년월일 |
| education | ENUM |  | 최종학력 |
| education_status | ENUM |  | 재학상태 |
| employment_type | ENUM |  | 고용형태 |
| has_job | BOOLEAN |  | 직장유무 |
| company_type | ENUM |  | 기업유형 |
| household_type | ENUM |  | 가구유형 |
| household_size | INT |  | 가구원 수 |
| monthly_income | ENUM |  | 월 소득 구간 |
| marital_status | ENUM |  | 결혼여부 |
| children_count | INT |  | 자녀 수 |
| created_at | DATETIME | NOT NULL | 생성일 |
| updated_at | DATETIME |  | 수정일 |

**Enum 정의**

```java
Gender: MALE, FEMALE, OTHER

Education: MIDDLE_SCHOOL, HIGH_SCHOOL, COLLEGE_2Y, COLLEGE_4Y, GRADUATE, DOCTORATE

EducationStatus: ENROLLED, ON_LEAVE, GRADUATED, DROPPED_OUT

EmploymentType: FULL_TIME, CONTRACT, PART_TIME, SELF_EMPLOYED, FREELANCE, UNEMPLOYED, STUDENT

CompanyType: LARGE, MEDIUM, SMALL, STARTUP, PUBLIC, GOVERNMENT

HouseholdType: SINGLE, COUPLE, NUCLEAR, EXTENDED, SINGLE_PARENT, MULTI_GEN

IncomeLevel: LEVEL_0, LEVEL_100, LEVEL_200, LEVEL_300, LEVEL_400, LEVEL_500, LEVEL_600, LEVEL_OVER_600

MaritalStatus: SINGLE, MARRIED, DIVORCED, WIDOWED

```

---

### 3. Policy (정책)

> PK: Auto Increment \| 외부 API 데이터

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | BIGINT | PK, AUTO | 내부 PK |
| external_id | BIGINT |  | 외부 원본 ID |
| external_uuid | VARCHAR(50) |  | 외부 UUID |
| policy_name | VARCHAR(500) | NOT NULL | 정책명 |
| summary | TEXT |  | 요약 |
| support_content | TEXT |  | 지원내용 |
| eligibility | TEXT |  | 신청자격 |
| restriction | TEXT |  | 제외대상 |
| support_amount_min | BIGINT |  | 최소 지원금액 |
| support_amount_max | BIGINT |  | 최대 지원금액 |
| region | VARCHAR(100) |  | 지역 |
| target | VARCHAR(500) |  | 지원대상 |
| age_range | VARCHAR(50) |  | 연령대 |
| founding_year_min | INT |  | 최소 설립연차 |
| founding_year_max | INT |  | 최대 설립연차 |
| founding_period | VARCHAR(50) |  | 설립기간 |
| due_date_start | DATETIME |  | 신청시작일 |
| due_date_end | DATETIME |  | 신청마감일 |
| status | ENUM |  | 상태 |
| agency | VARCHAR(200) |  | 주관기관 |
| department | VARCHAR(100) |  | 담당부서 |
| contact_phone | VARCHAR(50) |  | 연락처 |
| contact_detail | TEXT |  | 담당자 상세 |
| info_url | VARCHAR(500) |  | 상세정보 URL |
| apply_url | VARCHAR(500) |  | 신청 URL |
| apply_email | VARCHAR(100) |  | 신청 이메일 |
| apply_fax | VARCHAR(50) |  | 신청 팩스 |
| apply_offline_address | VARCHAR(500) |  | 오프라인 주소 |
| notice_file_url | VARCHAR(500) |  | 공고문 URL |
| notice_file_name | VARCHAR(300) |  | 공고문 파일명 |
| prospective_founder_yn | BOOLEAN |  | 예비창업자 대상 |
| view_count | BIGINT | DEFAULT 0 | 조회수 |
| scrap_count | BIGINT | DEFAULT 0 | 스크랩수 |
| created_at | DATETIME | NOT NULL | 생성일 |
| updated_at | DATETIME |  | 수정일 |

**PolicyStatus (Enum)**

- `UPCOMING` - 신청 예정
- `ACTIVE` - 신청 가능
- `CLOSED` - 마감
- `UNKNOWN` - 기간 미정

---

### 4. PolicyProcess (사업 진행 단계)

> PK: Auto Increment \| FK: policy_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | BIGINT | PK, AUTO | 내부 PK |
| policy_id | BIGINT | FK, NOT NULL | 정책 ID |
| external_id | BIGINT |  | 외부 ID |
| step | INT | NOT NULL | 단계 순서 |
| process_name | VARCHAR(200) |  | 단계명 |
| date_begin | DATETIME |  | 시작일 |
| date_end_raw | VARCHAR(100) |  | 종료일 (원본) |
| subject_yn | BOOLEAN |  | 대상 여부 |
| fixed_period_yn | BOOLEAN |  | 고정 기간 여부 |
| dday_yn | BOOLEAN |  | D-day 표시 여부 |

---

### 5. PolicyDocument (신청 서류)

> PK: Auto Increment \| FK: policy_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | BIGINT | PK, AUTO | 내부 PK |
| policy_id | BIGINT | FK, NOT NULL | 정책 ID |
| external_id | BIGINT |  | 외부 ID |
| document_name | VARCHAR(300) |  | 서류명 |
| write_type | ENUM |  | 작성유형 |
| required_yn | BOOLEAN |  | 필수 여부 |
| file_url | VARCHAR(500) |  | 양식 파일 URL |
| extension | VARCHAR(20) |  | 확장자 |
| file_size | BIGINT |  | 파일 크기 |
| origin_file_name | VARCHAR(300) |  | 원본 파일명 |

**DocumentWriteType (Enum)**

- `WRITE` - 작성 (양식 제공)
- `USER_UPLOAD` - 사용자 업로드
- `ISSUE` - 민원 서류

---

### 6. PolicyFile (첨부 파일)

> PK: Auto Increment \| FK: policy_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | BIGINT | PK, AUTO | 내부 PK |
| policy_id | BIGINT | FK, NOT NULL | 정책 ID |
| file_url | VARCHAR(500) |  | 파일 URL |
| origin_file_name | VARCHAR(300) |  | 원본 파일명 |
| extension | VARCHAR(20) |  | 확장자 |
| file_size | BIGINT |  | 파일 크기 |
| encrypted_yn | BOOLEAN |  | 암호화 여부 |

---

### 7. PolicyCategory (카테고리)

> PK: Auto Increment \| FK: policy_id<br>원본 배열 필드 정규화

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | BIGINT | PK, AUTO | 내부 PK |
| policy_id | BIGINT | FK, NOT NULL | 정책 ID |
| category_type | ENUM | NOT NULL | 카테고리 유형 |
| category_value | VARCHAR(200) | NOT NULL | 카테고리 값 |

**PolicyCategoryType (Enum)**

- `SUPPORT_AREA` - 지원분야 (사업화, R&D, 투자·융자 등)
- `TARGET_TYPE` - 대상유형 (중소기업, 중견기업 등)
- `INTEREST` - 관심분야 (바우처지원, 해외진출 등)
- `TECH` - 기술분야 (전기수소차, AI 등)

---

### 8. PolicyScrap (정책 스크랩)

> PK: UUID v7 \| FK: member_id, policy_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| member_id | UUID | FK, NOT NULL | 회원 ID |
| policy_id | BIGINT | FK, NOT NULL | 정책 ID |
| created_at | DATETIME | NOT NULL | 스크랩일 |

**제약조건**: UNIQUE(member_id, policy_id) - 중복 스크랩 방지

---

### 9. Post (게시글)

> PK: UUID v7 \| FK: member_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| member_id | UUID | FK, NOT NULL | 작성자 ID |
| title | VARCHAR(200) | NOT NULL | 제목 |
| content | TEXT | NOT NULL | 내용 |
| view_count | BIGINT | DEFAULT 0 | 조회수 |
| created_at | DATETIME | NOT NULL | 작성일 |
| updated_at | DATETIME |  | 수정일 |
| deleted_at | DATETIME |  | 삭제일 (Soft Delete) |

---

### 10. Comment (댓글)

> PK: UUID v7 \| FK: post_id, member_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| post_id | UUID | FK, NOT NULL | 게시글 ID |
| member_id | UUID | FK, NOT NULL | 작성자 ID |
| content | TEXT | NOT NULL | 댓글 내용 |
| created_at | DATETIME | NOT NULL | 작성일 |
| deleted_at | DATETIME |  | 삭제일 (Soft Delete) |

---

### 11. Attendance (출석)

> PK: UUID v7 \| FK: member_id

| 컬럼명 | 타입 | 제약조건 | 설명 |
| --- | --- | --- | --- |
| id | UUID | PK | UUID v7 |
| member_id | UUID | FK, NOT NULL | 회원 ID |
| attend_date | DATE | NOT NULL | 출석일 |
| created_at | DATETIME | NOT NULL | 생성일 |

**제약조건**: UNIQUE(member_id, attend_date) - 1일 1회 출석

---

## API 명세

### 인증 API (`/api/v1/auth`)

| Method | Endpoint | 설명 | 인증 |  |
| --- | --- | --- | --- | --- |
| POST | `/signup` | 회원가입 | ❌ |  |
| POST | `/login` | 로그인 | ❌ | 고객은 **카카오 소셜 로그인**을 통해 간편하게 회원가입 및 로그인을 할 수 있다. 별도의 이메일/비밀번호 입력 없이 카카오 계정으로 서비스를 이용할 수 있다. |
| POST | `/logout` | 로그아웃 | ✅ | 고객은 로그아웃 버튼을 클릭하여 현재 세션을 종료할 수 있다. 로그아웃 시 Refresh Token이 무효화되어 해당 기기에서 자동 로그인이 해제된다. |
| POST | `/reissue` | 토큰 재발급 | ❌ (Refresh Token) | 고객은 로그인 후 Access Token(1시간)이 만료되더라도, Refresh Token(7일)을 통해 자동으로 토큰을 재발급 받아 로그인 상태를 유지할 수 있다. |
| DELETE | `/withdraw` | 회원탈퇴 | ✅ | 고객은 더 이상 서비스를 이용하지 않을 경우 회원탈퇴를 할 수 있다. 탈퇴 시 개인정보는 즉시 삭제되지 않고 30일간 보관 후 완전 삭제된다(Soft Delete). 탈퇴 후 30일 이내 재가입 시 기존 데이터를 복구할 수 있다. |

### 회원 API (`/api/v1/members`)

| Method | Endpoint | 설명 | 인증 |  |
| --- | --- | --- | --- | --- |
| GET | `/me` | 내 정보 조회 | ✅ | 고객은 마이페이지에서 자신의 기본 정보(이름, 이메일, 가입일 등)를 확인할 수 있다. |
| PUT | `/me` | 내 정보 수정 | ✅ | 고객은 자신의 이름을 변경할 수 있다. 카카오 연동 이메일은 변경할 수 없다. |
| GET | `/me/profile` | 프로필 조회 | ✅ | 고객은 자신이 등록한 상세 프로필 정보(지역, 나이, 학력, 직장, 가구 정보 등)를 확인할 수 있다. |
| PUT | `/me/profile` | 프로필 수정 | ✅ | 고객은 자신의 상황이 변경되었을 때(이사, 취업, 결혼 등) 프로필 정보를 수정할 수 있다. 수정 즉시 맞춤 정책 추천 결과에 반영된다. |
| POST | `/me/profile` | 프로필 최초 등록 | ✅ | 신규 가입 고객은 맞춤 정책 추천을 받기 위해 프로필 정보를 등록할 수 있다. 프로필 등록은 필수는 아니지만, 등록하지 않으면 맞춤 추천 기능을 이용할 수 없다. |

### 정책 API (`/api/v1/policies`)

| Method | Endpoint | 설명 | 인증 |  |
| --- | --- | --- | --- | --- |
| GET | `/` | 정책 목록 (필터링/페이징) | ❌ | 고객은 전체 복지 정책 목록을 조회할 수 있다. 한 페이지에 20개씩 표시되며, 스크롤하여 더 많은 정책을 볼 수 있다(무한 스크롤). |
| GET | `/{id}` | 정책 상세 | ❌ | 고객은 정책의 상세 정보를 확인할 수 있다. 상세 정보에는 지원내용, 신청자격, 제출서류, 신청방법, 진행일정, 문의처 등이 포함된다. |
| GET | `/recommend` | 맞춤 정책 추천 | ✅ | 프로필을 등록한 고객은 자신의 **나이, 지역, 직업, 소득** 등의 조건에 맞는 정책을 추천받을 수 있다. 프로필 미등록 시 해당 기능을 이용할 수 없으며, 프로필 등록 안내 메시지가 표시된다. |
| GET | `/search` | 정책 검색 | ❌ | 고객은 키워드(정책명, 지원내용 등)로 원하는 정책을 검색할 수 있다. |
| POST | `/{id}/scrap` | 정책 스크랩 | ✅ | 고객은 관심 있는 정책을 스크랩하여 나중에 다시 확인할 수 있다. 스크랩한 정책은 마이페이지에서 모아볼 수 있다 |
| DELETE | `/{id}/scrap` | 스크랩 취소 | ✅ | 고객은 더 이상 관심 없는 정책의 스크랩을 취소할 수 있다. |
| GET | `/scraps` | 내 스크랩 목록 | ✅ | 고객은 자신이 스크랩한 정책 목록을 모아볼 수 있다. 마감된 정책도 함께 표시되며, 마감 여부가 표시된다. |

**정책 목록 필터 파라미터**

```
?status=ACTIVE          // 상태 필터
&region=서울             // 지역 필터
&categoryType=TECH      // 카테고리 유형
&categoryValue=AI       // 카테고리 값
&keyword=바우처           // 키워드 검색
&page=0&size=20         // 페이징
&sort=dueDateEnd,asc    // 정렬

```

### 커뮤니티 API (`/api/v1/posts`)

| Method | Endpoint | 설명 | 인증 |  |
| --- | --- | --- | --- | --- |
| GET | `/` | 게시글 목록 | ❌ | 고객은 커뮤니티에서 다른 사용자들이 작성한 게시글 목록을 볼 수 있다. 최신순으로 정렬되며, 제목, 작성자, 작성일, 조회수가 표시된다. |
| POST | `/` | 게시글 작성 | ✅ | 로그인한 고객은 커뮤니티에 게시글을 작성할 수 있다. 정책에 대한 질문, 후기, 정보 공유 등 자유롭게 글을 작성할 수 있다. |
| GET | `/{id}` | 게시글 상세 | ❌ | 고객은 게시글을 클릭하여 전체 내용을 확인할 수 있다. 조회 시 조회수가 1 증가한다. |
| PUT | `/{id}` | 게시글 수정 | ✅ | 고객은 자신이 작성한 게시글을 수정할 수 있다. 다른 사람의 글은 수정할 수 없다. |
| DELETE | `/{id}` | 게시글 삭제 | ✅ | 고객은 자신이 작성한 게시글을 삭제할 수 있다. 삭제된 글은 목록에서 사라지지만, 데이터베이스에는 30일간 보관된다(Soft Delete). |
| POST | `/{id}/comments` | 댓글 작성 | ✅ | 로그인한 고객은 게시글에 댓글을 작성할 수 있다. 댓글을 통해 다른 사용자와 소통할 수 있다. |
| DELETE | `/{postId}/comments/{commentId}` | 댓글 삭제 | ✅ | 고객은 자신이 작성한 댓글을 삭제할 수 있다. |
| GET | `/me` | 내 게시글 목록 | ✅ | 고객은 마이페이지에서 자신이 작성한 게시글 목록을 확인할 수 있다. |
| GET | `/me/comments` | 내 댓글 목록 | ✅ | 고객은 마이페이지에서 자신이 작성한 댓글 목록을 확인할 수 있다. |

### 출석 API (`/api/v1/attendance`)

| Method | Endpoint | 설명 | 인증 |  |
| --- | --- | --- | --- | --- |
| POST | `/` | 출석 체크 | ✅ | 고객은 매일 1회 출석 체크를 할 수 있다. 출석 체크 버튼을 누르면 오늘 날짜로 출석이 기록된다. 같은 날 중복 출석은 불가능하며, 이미 출석한 경우 안내 메시지가 표시된다. |
| GET | `/` | 월별 출석 현황 | ✅ | 고객은 캘린더 형태로 월별 출석 현황을 확인할 수 있다. 출석한 날짜에 체크 표시가 되어 한눈에 출석 현황을 파악할 수 있다. |
| GET | `/streak` | 연속 출석일 조회 | ✅ | 고객은 현재 연속으로 출석한 일수를 확인할 수 있다. 하루라도 출석을 놓치면 연속 출석일이 초기화된다. |
| GET | `/total` | 총 출석일 조회 | ✅ | 고객은 서비스 가입 이후 총 출석한 일수를 확인할 수 있다. |

---

## 패키지 구조

```
com.hoho.api
│
├── common
│   ├── config
│   │   ├── JpaConfig.java
│   │   ├── QueryDslConfig.java
│   │   ├── SecurityConfig.java
│   │   └── SwaggerConfig.java
│   ├── entity
│   │   └── BaseTimeEntity.java
│   ├── exception
│   │   ├── GlobalExceptionHandler.java
│   │   ├── ErrorCode.java
│   │   └── CustomException.java
│   ├── response
│   │   └── ApiResponse.java
│   ├── security
│   │   ├── jwt
│   │   │   ├── JwtTokenProvider.java
│   │   │   └── JwtAuthenticationFilter.java
│   │   └── UserDetailsServiceImpl.java
│   └── util
│       └── UuidV7Generator.java
│
├── domain
│   ├── auth
│   │   ├── controller
│   │   │   └── AuthController.java
│   │   ├── service
│   │   │   └── AuthService.java
│   │   └── dto
│   │       ├── SignupRequest.java
│   │       ├── LoginRequest.java
│   │       └── TokenResponse.java
│   │
│   ├── member
│   │   ├── controller
│   │   │   └── MemberController.java
│   │   ├── service
│   │   │   └── MemberService.java
│   │   ├── repository
│   │   │   ├── MemberRepository.java
│   │   │   └── MemberProfileRepository.java
│   │   ├── entity
│   │   │   ├── Member.java
│   │   │   ├── MemberProfile.java
│   │   │   ├── MemberRole.java
│   │   │   └── enums/
│   │   │       ├── Gender.java
│   │   │       ├── Education.java
│   │   │       └── ... (기타 Enum)
│   │   └── dto
│   │       ├── MemberResponse.java
│   │       └── ProfileUpdateRequest.java
│   │
│   ├── policy
│   │   ├── controller
│   │   │   └── PolicyController.java
│   │   ├── service
│   │   │   ├── PolicyService.java
│   │   │   └── PolicyScrapService.java
│   │   ├── repository
│   │   │   ├── PolicyRepository.java
│   │   │   ├── PolicyRepositoryCustom.java
│   │   │   ├── PolicyRepositoryImpl.java (QueryDSL)
│   │   │   └── PolicyScrapRepository.java
│   │   ├── entity
│   │   │   ├── Policy.java
│   │   │   ├── PolicyProcess.java
│   │   │   ├── PolicyDocument.java
│   │   │   ├── PolicyFile.java
│   │   │   ├── PolicyCategory.java
│   │   │   ├── PolicyScrap.java
│   │   │   ├── PolicyStatus.java
│   │   │   ├── PolicyCategoryType.java
│   │   │   └── DocumentWriteType.java
│   │   └── dto
│   │       ├── PolicyListResponse.java
│   │       ├── PolicyDetailResponse.java
│   │       └── PolicySearchCondition.java
│   │
│   ├── community
│   │   ├── controller
│   │   │   └── PostController.java
│   │   ├── service
│   │   │   ├── PostService.java
│   │   │   └── CommentService.java
│   │   ├── repository
│   │   │   ├── PostRepository.java
│   │   │   └── CommentRepository.java
│   │   ├── entity
│   │   │   ├── Post.java
│   │   │   └── Comment.java
│   │   └── dto
│   │       ├── PostRequest.java
│   │       ├── PostResponse.java
│   │       └── CommentRequest.java
│   │
│   └── attendance
│       ├── controller
│       │   └── AttendanceController.java
│       ├── service
│       │   └── AttendanceService.java
│       ├── repository
│       │   └── AttendanceRepository.java
│       ├── entity
│       │   └── Attendance.java
│       └── dto
│           └── AttendanceResponse.java
│
└── ExampleApiApplication.java

```

---

## 🗓️ 개발 일정

| 단계 | 작업 내용 | 예상 기간 | 산출물 |
| --- | --- | --- | --- |
| **1단계** | 프로젝트 스켈레톤 | 1일 | 프로젝트 구조, 공통 설정 |
| **2단계** | Entity 생성 | 1일 | 전체 Entity, Enum |
| **3단계** | 회원가입/로그인 | 2-3일 | Auth API, JWT |
| **4단계** | 프로필 CRUD | 2일 | Member API |
| **5단계** | 정책 조회/검색 | 3-4일 | Policy API, QueryDSL |
| **6단계** | 정책 추천 로직 | 2-3일 | 추천 알고리즘 |
| **7단계** | 스크랩 기능 | 1일 | Scrap API |
| **8단계** | 커뮤니티 | 2-3일 | Post/Comment API |
| **9단계** | 출석 체크 | 1일 | Attendance API |
| **10단계** | 테스트 & 버그 수정 | 2-3일 | 테스트 코드 |

**총 예상 기간: 약 2-3주**

---

## 비기능 요구사항

### 보안

- [x] JWT 기반 인증/인가 (Access Token + Refresh Token)
- [x] 비밀번호 암호화 (BCrypt)
- [x] 민감 데이터 PK는 UUID v7 사용
- [ ] CORS 설정
- [ ] Rate Limiting (선택)

### 데이터

- [x] Soft Delete 적용 (Member, Post, Comment)
- [x] Auditing (createdAt, updatedAt 자동 관리)
- [x] 페이징 처리 (Slice/Page)

### API

- [x] API 응답 형식 통일 (ApiResponse)
- [x] 전역 예외 처리 (GlobalExceptionHandler)
- [ ] API 문서화 (Swagger/OpenAPI)

### 쿼리

- [x] QueryDSL 동적 쿼리 적용 (정책 검색)
- [x] N+1 문제 해결 (fetch join, @EntityGraph)
- [ ] 쿼리 성능 최적화

---

## Spring Initializr 설정

### 기본 설정

```
Project: Gradle - Groovy
Language: Java
Spring Boot: 3.5.x
Packaging: Jar
Java: 17

Group: com.hoho
Artifact: hoho-api
Name: hoho-api
Package name: com.hoho.api

```

### Dependencies

- Spring Web
- Spring Data JPA
- MySQL Driver
- Spring Security
- Validation
- Lombok
- Spring Boot DevTools

### 수동 추가 (build.gradle)

```groovy
// QueryDSL
implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"

// JWT
implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.3'
runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.3'

// UUID v7
implementation 'com.github.f4b6a3:uuid-creator:5.3.3'

// Swagger (선택)
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'

```
