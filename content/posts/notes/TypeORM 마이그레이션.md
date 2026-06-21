---
title: TypeORM 마이그레이션
tags:
  - typeorm
  - migration
  - database
  - backend
  - nestjs
created: 2026-02-17
---

## 1. 마이그레이션

- DB 스키마 변경을 버전으로 관리하는 방식이다.

- 애플리케이션 코드는 Git으로 관리하지만, DB는 별도 저장소가 없다.

마이그레이션은 "어떤 SQL을 어떤 순서로 실행했는지"를 코드(파일)로 남겨, 환경마다 같은 순서로 같은 변경을 적용할 수 있게 한다.

- 그래서 재현 가능한 스키마 변경과 롤백 가능한 변경이 목표가 된다.
---
## 2. 왜 CLI와 DataSource가 따로 있나

- Nest(또는 Express 등) 앱은 실행 시 TypeOrmModule(또는 createConnection)으로 DB에 연결하고, 엔티티를 로드해 CRUD를 한다.

이때 쓰는 설정 파일은 "앱용 DataSource"다.

- 마이그레이션 실행은 "앱을 띄우지 않고" 스키마만 바꾸는 작업이다.

Node에서 typeorm/cli.js를 실행해, 마이그레이션 전용 설정 파일 하나만 넘겨준다(-d 경로).

- 따라서 앱용 설정과 CLI용 설정이 나뉜다.

CLI용 설정에는 migrations 배열(어디에 마이그레이션 파일이 있는지), DB 접속 정보, synchronize: false가 들어간다.

이 파일을 "마이그레이션용 DataSource"라고 부르면 된다.

정리: 앱은 앱용 설정으로 DB를 쓰고, migration:run / migration:revert / migration:generate는 CLI가 "마이그레이션용 DataSource" 파일만 로드해서 동작한다.

---
## 3. 명령어가 실행될 때 일어나는 일

  

- npm run migration:run 같은 스크립트는 결국 TypeORM CLI를 -d [DataSource 경로]와 함께 실행한다.

- CLI는 다음을 한다:

1. DataSource 파일을 로드해 DB에 연결한다.

2. migrations 배열에 맞는 파일들을 디스크에서 찾고, 파일명 앞의 숫자(타임스탬프)로 정렬한다.

3. DB에 migrations 테이블이 없으면 만들고, 이미 실행된 마이그레이션 목록을 읽는다.

4. "파일에 있는 마이그레이션 중, migrations 테이블에 기록이 없는 것"만 골라 타임스탬프 순서대로 실행한다.

5. 각 마이그레이션마다 트랜잭션을 열고, 해당 클래스의 up(queryRunner)를 호출한 뒤, 성공 시 migrations 테이블에 (timestamp, name) 한 줄 넣고 커밋한다.

실패하면 롤백하고, 그 마이그레이션 이후는 실행하지 않는다.

  

즉, 원리는

"어떤 마이그레이션이 이미 적용됐는지"를 DB의 migrations 테이블로 알고,

아직 적용되지 않은 파일만 순서대로 up() 실행한 다음, 실행한 것만 테이블에 기록하는 것이다.

  

---
## 4. migrations 테이블의 역할

- TypeORM이 자동으로 생성·사용하는 테이블이다.

- 컬럼은 보통 id, timestamp(파일명의 숫자), name(클래스명) 등이다.

- 이 테이블이 "지금까지 어떤 마이그레이션을 적용했는지"의 단일 출처가 된다.

- 따라서:

- migration:run: "파일 시스템의 마이그레이션 목록"과 "migrations 테이블"을 비교해, 테이블에 없는 것만 실행하고, 실행한 뒤 테이블에 한 줄씩 넣는다.

- migration:revert: 테이블에서 "가장 마지막에 실행된" 한 건을 찾아, 그에 해당하는 파일의 down(queryRunner)를 실행한 뒤, 그 행을 테이블에서 지운다.

  

파일명의 타임스탬프와 클래스의 name이 테이블의 (timestamp, name)와 맞아야 하므로, 파일명·클래스명 규칙이 중요한 이유가 여기 있다.

  

---
## 5. up과 down의 역할

- 각 마이그레이션 파일은 MigrationInterface를 구현한 클래스 하나를 export하고, up, down 두 메서드를 가진다.

- up: "스키마를 한 단계 앞으로 바꾸는" 변경이다.

migration:run 시, "아직 적용되지 않은" 마이그레이션에 대해서만 호출된다.

- down: "그 한 단계를 되돌리는" 변경이다.

migration:revert 시, "방금 적용된 것"으로 간주되는 마이그레이션에 대해서만 호출된다.

- 같은 파일이지만 run일 때는 up만, revert일 때는 down만 사용된다.

그래서 up에서 "컬럼 추가"를 했다면, down에서는 "그 컬럼 삭제"를 해야 일관성이 유지된다.

  

이론적으로, 모든 마이그레이션은 되돌릴 수 있도록 down을 설계하는 것이 좋다.

(실무에서는 데이터 손실·의존성 때문에 down을 비우거나 수동 복구만 적어두는 경우도 있다.)

---

## 6. migration:generate의 개념

- 엔티티는 "우리가 원하는 최종 스키마"를 코드로 표현한 것이다.

- 현재 DB는 "지금 실제로 적용된 스키마"이다.

- migration:generate는

"마이그레이션용 DataSource"에 정의된 entities와 현재 DB 스키마를 비교해서,

"엔티티에는 있지만 DB에는 없는 변경", "DB에는 있지만 엔티티에는 없는 변경"을 계산하고,

그 차이를 반영하는 새 마이그레이션 파일을 자동 생성한다.

- 따라서 "엔티티를 먼저 수정하고, generate로 차이를 마이그레이션으로 뽑는다"는 워크플로가 가능하다.

생성된 SQL이 항상 원하는 대로는 아니므로, 복잡한 변경·데이터 이전은 수동으로 마이그레이션을 작성하는 경우가 많다.

---

## 7. 요약 표

  

| 주제 | 요약 |
|------|------|
| 마이그레이션 | 스키마 변경을 파일(버전)로 관리해, 환경마다 같은 순서로 적용·롤백할 수 있게 하는 방식. |
| CLI vs 앱 | 앱은 "앱용 DataSource"로 DB 사용. 마이그레이션 run/revert/generate는 "CLI용 DataSource" 파일만 로드해 동작. |
| run | 파일 시스템의 마이그레이션 중, DB의 migrations 테이블에 없는 것만 타임스탬프 순으로 up() 실행 후 테이블에 기록. |
| revert | migrations 테이블에서 마지막 한 건을 찾아 해당 파일의 down() 실행 후, 그 기록 삭제. |
| migrations 테이블 | "이미 적용된 마이그레이션"의 단일 출처. timestamp·name으로 어떤 파일이 적용됐는지 식별. |
| up / down | up = 한 단계 전진, down = 그 한 단계 후퇴. run은 up만, revert는 down만 사용. |
| generate | 엔티티(원하는 스키마)와 현재 DB를 비교해, 차이를 반영하는 마이그레이션 파일을 자동 생성. |

  

---

이 문서는 예제·프로젝트별 설정보다 원리와 개념을 블로그에서 다시 볼 수 있도록 정리한 것이다.

- 마이그레이션 개요·설정
https://typeorm.io/migrations