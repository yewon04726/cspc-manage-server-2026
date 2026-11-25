# CSPC 관리 사이트 (Backend)

CSPC 학회의 **실습실 관리 시간표 생성, 출결 기록, 지각 자동 처리, 컴퓨터(PC) 상태와 수리 요청**을 처리하는 REST API 서버입니다.

- 프론트엔드: [cspc-manage-2026](https://github.com/yewon04726/cspc-manage-2026)
- 배포: cspc 학회 서버 (https://api-manage.cspc.me)

---

## 왜 만들었나

학회원들이 시간을 나누어 실습실을 관리했지만 **시간표만 있고 관리가 이루어졌는지 확인하는 체계가 없었습니다.**
컴퓨터 점검 결과도 컴퓨터 번호만 적힌 엑셀 파일로는 체계적으로 정리하기 어려웠습니다.

이 서버는 다음을 담당합니다.

1. 학기 설정 한 번으로 주차별 관리 시간표를 생성
2. 관리 시작/끝을 **실습실 네트워크에서만** 기록하고, 지각을 자동으로 판정
3. 컴퓨터별 정보와 고장 요청을 기록

---

## 주요 기능

### 1. 학기·시간표 관리
- 개강일과 종강일을 입력하면 **주차별 날짜(시작일~종료일)를 생성**합니다.
- 기본 시간표(요일·교시별 기본 담당자)를 **모든 주차에 복사**해 주차별 시간표를 만듭니다.
- 특정 주차의 담당자를 **대타로 변경**할 수 있습니다.
- 새 학기를 시작할 때 출결 기록, 시간표, 주차 정보, 학회원별 지각 횟수를 한 번에 초기화합니다.

### 2. 출결 기록과 지각 판정

| 항목 | 규칙 |
| --- | --- |
| 관리 시작·종료 기록 | 서버가 요청 IP를 확인해 **실습실 공인 IP에서만** 허용 |
| 출석 행 생성 | 수업 시작 1분 전부터 종료 10분 후까지 담당자 기준으로 자동 생성 |
| 지각 판정 | 시작 10분이 지나도록 관리 시작이 없거나, 종료 10분이 지나도록 관리 종료가 없으면 지각 |
| 지각 횟수 | 지각이 확정되면 학회원의 `late_count`를 1 증가 |
| 중복 방지 | 이미 기록된 시작/종료는 다시 기록하지 않고, 이미 지각 처리된 행은 지각 횟수를 중복으로 올리지 않음 |

- 판정은 사람이 아니라 **DB 스케줄러(`pg_cron`)가 매 분 실행하는 SQL 함수**(`sql/attendance_cron.sql`)가 처리합니다.
- 홈 화면의 **지각 랭킹**은 `late_count` 내림차순으로 제공합니다.

### 3. 실습실 PC 관리
- 호수(location)별 컴퓨터 목록과 상세 정보(제조사, 모델, 시리얼 넘버)를 조회·수정합니다.
- 같은 위치에 **같은 컴퓨터 번호가 중복되면 409로 거절**합니다.
- 컴퓨터별 **고장 신고**(카테고리, 제목)를 등록·삭제하고, 고장난 컴퓨터만 모아서 조회합니다.

### 4. 기타
- 회의 안건, 건의사항, 비품 요청을 등록·조회·삭제하고 비품 요청은 완료 처리할 수 있습니다.

---

## 기술 스택

| 영역 | 구성 |
| --- | --- |
| Runtime / Framework | Node.js, Express 5 |
| Database | Supabase (PostgreSQL) |
| 스케줄링 | pg_cron (Supabase) |
| 라이브러리 | `@supabase/supabase-js`, `cors`, `dotenv` |
| 개발 도구 | nodemon |
| 배포 | Render |

---

## API

응답은 모두 `{ success, data, message }` 형식입니다.

### 시간표·출결

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/api/schedules/meta` | 학기 정보(주차 목록) 조회 |
| GET | `/api/schedules/:week` | 주차별 시간표와 출결 조회 (`today`로 이번 주 조회 가능) |
| PATCH | `/api/schedules/change` | 대타 변경 |
| POST | `/api/attendance/:weekly_id/start` | 관리 시작 기록 (실습실 IP만) |
| PATCH | `/api/attendance/:weekly_id/end` | 관리 종료 기록 (실습실 IP만) |

### 학회원

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/api/users` | 학회원 목록 |
| GET | `/api/users/ranking` | 지각 랭킹 |
| GET | `/api/users/:id` | 학회원 정보 |
| PATCH | `/api/users/:id` | 이름, 색상, 지각 횟수 수정 |
| GET | `/api/users/:id/logs` | 개인 출결 로그 |

### 실습실

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/api/lab?location=` | 호수별 컴퓨터 목록 |
| GET | `/api/lab/broken` | 고장난 컴퓨터 목록 |
| GET | `/api/lab/:id` | 컴퓨터 상세와 고장 요청 목록 |
| PATCH | `/api/lab/:id` | 컴퓨터 정보 수정 |
| POST | `/api/lab/:id/reports` | 고장 신고 등록 |
| DELETE | `/api/lab/reports/:id` | 고장 신고 삭제 |

### 관리자

| Method | Path | 설명 |
| --- | --- | --- |
| POST | `/api/admin/setup-weeks` | 개강·종강일로 주차와 시간표 생성 |
| PATCH | `/api/admin/schedules/manager` | 기본 시간표 담당자 일괄 수정 |
| POST | `/api/admin/schedules/init` | 학기 데이터 초기화 |
| POST | `/api/admin/users` | 학회원 추가 |
| DELETE | `/api/admin/users/:id` | 학회원 삭제 |

### 회의·건의·비품

| Method | Path | 설명 |
| --- | --- | --- |
| GET, POST, DELETE | `/api/agendas` | 회의 안건 |
| GET, POST, DELETE | `/api/suggestions` | 건의사항 |
| GET, POST, PATCH, DELETE | `/api/supplies` | 비품 요청 (PATCH는 완료 처리) |

---

## 데이터 모델

| 테이블 | 내용 |
| --- | --- |
| `Admin_Users` | 학회원 (이름, 표시 색상, 지각 횟수) |
| `Semester_Weeks` | 주차별 시작일·종료일 |
| `Timetable_Slots` | 기본 시간표 (요일, 교시, 시간, 기본 담당자) |
| `Weekly_Schedules` | 주차별 시간표 (담당자, 대타 여부) |
| `Shift_Attendance` | 출결 (시작·종료 시각, 지각 여부) |
| `Lab_Computers` | 실습실 컴퓨터 정보 |
| `Repair_Requests` | 컴퓨터 고장 신고 |
| `Meeting_Agendas`, `Suggestions`, `Supply_Requests` | 회의 안건, 건의사항, 비품 요청 |


---

## 프로젝트 구조

```text
index.js                      라우팅과 서버 시작
controllers/
├─ scheduleController.js      시간표 조회, 대타, 관리 시작/종료
├─ adminController.js         학기 설정, 학기 초기화, 학회원 추가/삭제
├─ userController.js          학회원 조회·수정, 지각 랭킹, 출결 로그
├─ labController.js           컴퓨터 정보, 고장 신고
├─ agendaController.js        회의 안건
├─ suggestionController.js    건의사항
└─ supplyController.js        비품 요청
middleware/auth.js            실습실 IP 확인
config/supabaseClient.js      Supabase 연결
sql/attendance_cron.sql       출석 행 생성, 지각 판정을 매 분 실행하는 DB 함수
```

---

## 실행 방법

```bash
npm install
npm run dev     # nodemon으로 개발 서버 실행
npm start       # 일반 실행
```

### 환경 변수

프로젝트 루트에 `.env`를 만듭니다. 실제 값은 Git에 올리지 않습니다(`.gitignore` 적용).

```env
SUPABASE_URL=
SUPABASE_KEY=
PORT=3000
```

### 출결 자동 처리 설정

Supabase SQL Editor에서 `sql/attendance_cron.sql`을 한 번 실행하면 `attendance-jobs-every-minute` 작업이 등록됩니다.

---

## 알려진 한계

- **관리자 API에 인증이 없습니다.** `/api/admin/*`은 경로로만 구분되어 있고 로그인이나 권한 확인이 없어, 이후 관리자 인증을 추가할 계획입니다.
- **실습실 IP 확인**은 한 개의 공인 IP를 코드에 지정하고 있어 네트워크가 바뀌면 코드를 수정해야 합니다. 환경 변수로 분리하는 것이 개선 과제입니다.
- 자동 테스트가 없습니다.
- 테이블 생성 스키마는 Supabase에서 관리하고 있어 이 저장소에는 포함되어 있지 않습니다.

---

## 내 역할

- 학기 설정과 주차별 시간표 생성 로직 구현
- 출결 기록, 지각 판정(DB 스케줄러), 지각 랭킹 구현
- 실습실 PC 정보와 고장 신고 API 구현
- 프론트엔드와 API 형식 협의
- DB 설계

---

원본 저장소: [CSPCLAB/cspc-manage-server-2026](https://github.com/CSPCLAB/cspc-manage-server-2026)
