# 소개
* `MySQL` 경쟁력은 가격, 비용 등
* 어떤 `DBMS`가 좋은가?
  * 안정성
  * 성능과 기능
  * 커뮤니티나 인지도
* `DBMS` 랭킹
  * [DBMS 랭킹 차트](https://db-engines.com/en/ranking)
  * 기준
    * 웹 사이트 언급 횟수 (`website metions`)
    * 검색 빈도 (`search frequency`)
    * 기술 토론 빈도 (`technical disucssion frequency`)
    * DBMS별 구인 (`current job offers`)
    * 전문가 인맥 (`professional network profiles`)

# 설치와 설정

## 설치
`$ brew install mysql`
* 기존에 `mysql@5.7` 등 다른 버전이 설치되어 있다면 해당 명령어로 설치해도 8버전이 설치되지 않음
* 기존 버전 삭제 후 설치
* 설치 시점(`2021/11/13`) 버전 `8.0.27`

클린 셧다운 개념을 통해 서버 재시작 시 트랜잭션 복구 과정 진행하지 않음
* MySQL 시작/종료 시 `InnoDB` 스토리지 엔진의 버퍼 풀 내용이 백업/복구 과정이 내부적으로 실행됨
* 버퍼 풀에 적재된 데이터 파일의 데이터 페이지에 대한 메타 정보를 백업하면 용량이 적어 백업 자체는 빠름
* 서버 시작시엔 디스크에 있는 데이터 파일을 모두 읽어 적재하므로 느림

서버 연결
* `$ mysql -uroot -p --host=localhost --socket=/tmp/mysql.sock`
  * 소켓을 사용해 접속 (`Unix domain socket` - `Inter Prcoess Communication` 방식 이용)
* `$ mysql -uroot -p --host=127.0.0.1 --port=3306`
  * TCP/IP를 통해 접속 (일반적으로 포트 명시)
* `$ mysql -uroot -p`
  * 기본값으로 `localhost` 적용
* 기본 비밀번호 사용
  * `root`
* 서버 기동 시 생성되는 유닉스 소켓 파일은 리부팅하지 않으면 생성되지 않음
  * 삭제하지 않도록 주의

접속
* `--initalize-insecure` 옵션으로 초기화되면 비밀번호 없이 로그인 가능
  * `--initalize` 옵션이면 로그 파일에 기록된 비밀번호 사용
* 데이터베이스 목록 확인
  * `$ show databases;`
* 원격 서버에서 접속 가능 여부만 확인하는 경우
  * `Telnet`
    * `$ telnet 10.2.40.61 3306`
  * `Netcat`
    * `$ nc 10.2.40.61 3306`

스키마, 테스트 데이터 설정
* 스키마 생성
  * `CREATE DATABASE employees DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;`
  * 툴에서 기본 스키마 설정
* `employees.sql` 파일 추가
  * `Datagrip`에서는 해당 데이터소스 > 우클릭 > `Run SQL script` 선택하여 파일 추가

## MySQL 서버 업그레이드
방식
* [1] 인플레이스(`In-Place Upgrade`) 업그레이드
  * 서버의 데이터 파일을 그대로 두고 업그레이드
  * 버전 등 제약 사항이 있지만 업그레이드 시간 단축
* [2] 논리적 업그레이드
  * `mysqldump` 등 도구를 이용해 서버의 데이터를 `SQL`, 텍스트 파일로 덤프 후 새로운 서버에 덤프 데이터를 적재
  * 제약 사항이 많이 없지만 시간이 많이 소요

버전 별 업그레이드
* 동일 메이저에서 마이너 버전 간 업그레이드
  * 대부분 데이터 파일 변경 없이 진행, 여러 버전을 건너뛰는 업그레이드 가능
  * 프로그램 재설치만 필요
* 메이저 버전 간 업그레이드
  * 대부분 데이터 파일의 변경 필요, 반드시 직전 버전에서만 업그레이드 가능
  * 이 경우 데이터를 백업 후, 새 서버에 데이터를 적재시키는 논리적 업그레이드가 유리할 수 있음
* 인플레이스 업그레이드는 특정 마이너 버전에서만 가능한 경우도 존재
  * `GA(General Availability)` 버전이 아닌 경우

8.0 업그레이드 시 고려사항
* 사용자 인증 방식 변경
  * `Caching SHA-2 Authentication`이 새로운 기본 인증 방식
  * `Native Authentication`이 기존 방식
    * 기존 방식을 사용하려면 `--default-authentication-plugin=mysql_native_password` 활성화
* 8.0 버전과 호환성 체크
  * 손상된 FRM 파일이나 호환되지 않은 데이터 타입, 함수 등을 `mysqlcheck` 유틸리티를 이용해 확인
  * `$ mysqlcheck -u root -p --all-databases --check-upgrade`
* 외래키명 길이
  * 8 버전에서는 64자로 제한
* 인덱스 힌트
  * 성능 테스트 필히 수행할 것 (버전에 따라 더 느려질 수 있음)
* `Group By`에 사용된 정렬 옵션
  * `ASC | DESC` 등 변경 필요
* 파티션을 위한 공용 테이블스페이스
  * 파티션의 각 테이블스페이스를 공용 테이블스페이스에 저장할 수 없음
  * `ALTER TABLE .. REORGANIZE` 명령으로 개별 테이블스페이스를 사용하도록 변경

업그레이드
* 데이터 딕셔너리 업그레이드
  * 5.7 버전까지는 데이터 딕셔너리 정보가 `FRM` 확장자 파일로 별도로 관리
  * 8.0 버전부터는 데이터 딕셔너리 정보가 `InnoDB` 테이블로 저장
* 서버 업그레이드
  * 서버의 시스템 데이터베이스(아래)의 테이블 구조를 8.0 버전에 맞게 변경
    * `performance_schema`
    * `information_schema`
    * mysql 데이터베이스

## 서버 설정

설정 파일 구성
* 설정 파일(`my.cnf`) 경로 확인
  * `Default options are read from the following files in the given order:` 확인
    * 설정 시점 `/etc/my.cnf /etc/mysql/my.cnf /usr/local/etc/my.cnf ~/.my.cnf`
    * 1, 2, 4번째 경로는 어느 MySQL이나 동일하게 검색됨
    * 3번째 경로는 컴파일될 때 내장
    * 일반적으로 서버용 설정 파일은 1, 2번째 경로를 많이 사용
    * 따라서 하나의 물리 서버에서 여러 MySQL 서버를 띄운다면 1, 2번째 경로는 충돌이 발생할 수 있음

시스템 변수 특징
* MySQL은 설정 파일의 내용을 읽어 초기화한 후 이러한 값을 변수(`System Variables`)에 저장
* 글로벌 변수와 세션 변수를 구분해야 함
* 변수 별 항목
  * `Cmd-Line`
    * 서버의 명령형 인자로 설정될 수 있는 지 여부
  * `Option file`
    * 설정 파일인 `my.cnf` 등으로 제어할 수 있는지 여부
  * `System Var`
    * 시스템 변수인지 여부
    * `-`, `_` 구분 주의 (현재는 `_`로 맞춰가는 중)
  * `Var Scope`
    * 시스템 변수의 적용 범위
  * `Dynamic`
    * 시스템 변수가 동적인지 구분하는 변수

글로벌 변수, 세션 변수
* 글로벌 변수
  * 하나의 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수
* 세션 범위의 시스템 변수
  * 서버에 접속할 때 기본으로 부여하는 옵션의 기본값을 제어하는 데 사용됨
  * 기본 값을 그대로 사용하지만 클라이언트 필요에 따라 개별 커넥션 단위로 변경할 수 있음
    * `autocommit` 등이 대표적인 예시
  * 한번 연결된 커넥션의 세션 변수는 서버에서 강제로 변경할 수 없음
  * 서버의 설정 파일(`my.cnf`, `my.ini` 등)에 명시해 초기화할 수 있는 변수는 대부분 범위가 `Both`라고 명시되어 있음
    * 서버가 기억하고 있다가 실제 커넥션이 생성될 때 기본값으로 사용
  * 순수 범위가 `Session`인 변수는 서버 설정 파일에 초깃값을 명시할 수 없음
    * 커넥션이 만들어진 후부터 해당 커넥션에서만 유효한 설정 변수를 의미

정적 변수, 동적 변수
* 시스템 변수는 서버가 기동 중인 상태에서 변경 가능 여부에 따라 동적 변수와 정적 변수로 구분됨
* 서버 시스템 변수는 다음 두 가지 경우로 구분 가능
  * `디스크`에 저장돼 있는 설정 파일(`my.cnf`, `my.ini` 등)을 변경하는 경우
  * 이미 기동중인 서버의 `메모리`에 있는 서버 변수를 변경하는 경우
* 설정 파일 내용을 변경하더라도 서버 재시작해야 반영됨
  * `SET` 명령 등으로 값 변경 가능
  * `SHOW GLOBAL VARIABLES LIKE '%max_connections%';`
  * `SET GLOBAL MAX_CONNECITONS=500;`
  * 위 변경은 기동 중인 인스턴스에서만 유효
* `GLOBAL` 키워드를 통해 글로벌 변수 조회, 변경
  * 생략했을 때는 세션 변수를 타겟으로 함
* 동적으로 변경 가능한 변수라면 서버 리부팅이 필요하지 않음
* 변수 범위가 `Both`(글로벌이면서 세션)이면 글로벌 값만 변경하면 이미 존재하는 커넥션 세션 변숫값은 변경되지 않고 유지됨
  * 확인 테스트
    * `SHOW GLOBAL VARIABLES LIKE 'join_buffer_size';`
    * `SHOW VARIABLES LIKE 'join_buffer_size';`
  * 글로벌 변수 값만 변경하면 글로벌 변수에만 적용됨
    * `SET GLOBAL JOIN_BUFFER_SIZE = 524288;`

`SET PERSIST`
* 8버전부터는 `SET PERSIST` 명령으로 인스턴스에 값 적용과 함께 설정 파일까지 변경
* 변수 값 변경과 동시에 설정 파일 내용 변경
  * `SET PERSIST MAX_CONNECTIONS = 5000;`
  * `mysqld-auto.cnf` 파일에 변경 내용 추가 기록, 서버 재시작 시 `my.cnf` + `mysqld-auto.cnf`
* `SET PERSIST` 명령은 세션 변수에는 적용되지 않음
  * 시스템 변수를 변경하면 서버는 자동으로 글로벌 변수의 변경으로 인식하여 변경
* `SET PERSIST_ONLY`
  * 현재 서버 인스턴스에는 적용하지 않고 다음 재시작을 위해 `mysqld-auto.cnf` 파일만 변경
  * 정적인 변수의 값을 영구적으로 변경 가능
  * `SET PERSIST`은 정적인 변수는 실행중인 서버에서 변경할 수 없음
* 참조
  * `performance_schema.variables_info` 뷰
  * `performance_schema.persisted_variables` 테이블
* 삭제
  * 파일을 직접 수정할 때 내용상 오류가 있으면 서버가 시작되지 않음
  * 따라서 `RESET PERSIST` 명령 사용하는 것이 좋음
    * `RESET PERSIST max_connections;`
    * `RESET PERSIST IF EXISTS max_connections;`
    * `RESET PERSIST;`