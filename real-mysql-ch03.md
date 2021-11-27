# 사용자 및 권한

## 사용자 식별
* `MySQL`에서 사용자 계정은 아이디 + IP까지 확인
  * 접속 지점까지 계정의 일부로 취급
  * 호스트 설정
    * `'svc_id'@'127.0.0.1'` (특정 호스트)
    * `'svc_id'@'%'` (모든 호스트)
    * 위처럼 동일 계정이 호스트가 다른 경우 항상 작은 범위를 확인
* 8버전부터는 `Role` 추가

## 사용자 계정 관리
* 8버전부터 `SYSTEM_USER` 권한 소유 여부에 따라 시스템 계정(`System Account`)과 일반 계정(`Regular Account`)으로 구분
* 시스템 계정
  * 서버 내부적으로 실행되는 백그라운드 스레드와 무관, 사용자를 위한 계정
  * DB 서버 관리자를 위한 계정
  * 시스템 계정은 시스템 계정과 일반 계정을 관리(생성, 수정, 삭제 등) 가능
  * 특정 중요 작업 수행 가능
    * 계정 관리 (생성, 수정, 삭제, 권한 부여 및 제거 등)
    * 다른 세션 및 해당 세션에서 실행 중인 쿼리 강제 종료
    * 스토어드 프로그램 생성 시 `DEFINER`를 타 사용자로 설정
      * `DEFINER`는 스토어드 프로그램을 생성한 사용자를 의미
* 일반 계정
  * App 개발자 등을 위한 계정
* 시스템 계정과 일반 계정 두 개념이 도입된 것은 `SYSTEM_USER` 권한을 일반 사용자에게 부여하지 않게 하기 위함
  * 해당 책에서 `사용자`는 서버를 사용하는 주체, `계정`은 서버에 로그인하기 위한 식별자를 가리킴
* 기본 내장된 계정 (`account_locked` 상태)
  * `'mysql.sys'@'localhost'`
    * `sys` 스키마 객체들의 `DEFINER`로 사용되는 계정
  * `'mysql.session'@'localhost'`
    * `MySQL` 플러그인이 서버로 접근할 때 사용되는 계정
  * `'mysql.infoschema'@'localhost'`
    * `information_schema`에 정의된 뷰의 `DEFINER`로 사용되는 계정

계정 생성
* `CREATE USER`로 생성, `GRANT`로 권한 부여
* 계정 생성 시 옵션
  * 계정 인증 방식, 비밀번호(유효 기간, 이력 개수, 재사용 불가능 기간)
  * 기본 역할 (Role)
  * SSL 옵션
  * 계정 잠금 여부

~~~
CREATE USER 'user'@%
 IDENTIFIED WITH 'mysql_native_password' BY 'password'
 REQUIRE NONE
 PASSWORD EXPIRE INTERVAL 30 DAY
 ACCOUNT UNLOCK
 PASSWORD HISTORY DEFAULT
 PASSWORD REUSE INTERVAL DEFAULT
 PASSWORD REQUIRE CURRENT DEFAULT;
~~~
* `IDENTIFIED WITH ${인증 방식, 플러그인명}`
  * 사용자 인증 방식, 비밀번호
  * 기본 인증 방식은 `IDENTIFIED BY 'password'`
    * `Native Pluggable Authentication` (5.7 버전 이전 기본 방식)
      * 5.7 이전 버전에서 사용되던 방식
      * 단순히 비밀번호에 대한 해싱(SHA-1)값을 저장하고, 클라이언트가 보낸 값과 일치하는지 비교 인증하는 방식
      * 8 버전 이후에 적용하려면 설정 변경 필요 (또는 `my.cnf` 파일 수정)
        * `SET GLOBAL default_authentication_plugin="mysql_native_password"`
    * `Caching SHA-2 Pluggable Authentication` (8 버전 이후 기본 방식)
      * 위 방식과 가장 큰 차이는 해싱 알고리즘 차이 (SHA-2)
      * 내부적으로 `Salt` 키 사용, 동일한 키라도 해싱 계산으로 결과가 다름
        * 해싱에서 오는 성능 부하는 결괏값 메모리 캐싱을 통해 개선
      * `SSL/TLS` 또는 `RSA 키페어`를 반드시 사용
        * 따라서 SSL 옵션 활성화해야 함
      * SCRAM(`Salted Challenge Response Authentication Mechanism`) 방식 사용
        * 이 방식은 평문 비밀번호를 이용해 5,000번 이상 암호화 해시 함수를 실행해야 요청 가능 (무차별 대입 공격 방어)
        * 하지만 정상적인 유저의 연결도 느리게 함
    * `PAM Pluggable Authentication`
      * 유닉스, 리눅스 패스워드 또는 LDAP(`Lightweight Directory Access Protocol`) 같은 외부 인증을 사용할 수 있게 해주는 방식
      * 엔터프라이즈 에디션에서만 사용 가능
    * `LDAP Pluggable Authentication`
      * LDAP(`Lightweight Directory Access Protocol`) 같은 외부 인증을 사용할 수 있게 해주는 방식
      * 엔터프라이즈 에디션에서만 사용 가능
* `REQUIRE`
  * 서버 접속 시 `SSL/TLS` 채널 사용 여부 (기본값은 비암호화 채널)
  * 하지만 `Caching SHA-2` 방식이라면 암호화된 채널을 사용
* `PASSWORD EXPIRE`
  * 비밀번호 유효 기간
    * 기본값은 `default_password_lifetime`에 저장된 값 적용
  * 앱 접속용 계정 유효기간 설정은 위험할 수 있음
  * 설정 가능 옵션
    * `PASSWORD EXPIRE`
      * 계정 생성과 동시에 만료 처리
    * `PASSWORD EXPIRE NEVER`
      * 만료 기간 없음
    * `PASSWORD EXPIRE DEFAULT: default_password_lifetime`
      * 시스템 변수에 저장된 기간으로 유효기간 설정
    * `PASSWORD EXPIRE INTERVAL n DAY`
      * 유효 기간을 오늘로부터 `n`일자로 설정
* `PASSWORD HISTORY`
  * 한 번 사용한 비밀번호를 재사용하지 못하게 함
  * 설정 가능 옵션
    * `PASSWORD HISTORY DEFAULT`
      * `password_history` 시스템 변수에 저장된 수만큼 비밀번호의 이력을 저장
      * 이력에 남은 비밀번호는 재사용 금지
    * `PASSWORD HISTORY n`
      * 이력을 최근 `n`개까지만 저장
      * 이력에 남은 비밀번호는 재사용 금지
  * `mysql.password_history` 테이블 사용
* `PASSWORD REUSE INTERVAL`
  * 한 번 사용한 비밀번호의 재사용 금지 기간 설정
    * 기본값은 `password_reuse_interval`에 저장된 값 적용
  * 설정 가능 옵션
    * `PASSWORD REUSE INTERVAL`
      * `password_reuse_interval` 변수에 저장된 기간으로 설정
    * `PASSWORD REUSE INTERVAL n DAY`
      * `n`일자 이후에 비밀번호를 재사용할 수 있게 설정
* `PASSWORD REQUIRE`
  * 기존 비밀번호 만료 시 `새 비밀번호`로 변경 시 `기존 비밀번호 입력`을 필요로 하는지 여부
    * 기본값은 `password_require_current`에 저장된 값 사용
  * 설정 가능 옵션
    * `PASSWORD REQUIRE CURRENT`
      * 변경 시 현재 비밀번호를 입력하도록 설정
    * `PASSWORD REQUIRE OPTIONAL`
      * 변경 시 현재 비밀번호를 입력하지 않도록 설정
    * `PASSWORD REQUIRE DEFAULT`
      * `password_require_current` 시스템 변수 값으로 설정
* `ACCOUNT UNLOCK/LOCK`
  * 계정 생성 또는 변경 시 계정을 사용 못하게 잠글지 여부
  * `LOCK`
    * 계정을 사용하지 못하게 잠금
  * `UNLOCK`
    * 잠긴 게정을 다시 사용 가능 상태로 잠금 해제

## 비밀번호 관리

고수준 비밀번호
* 쉽게 유추 가능한 단어를 사용하지 않게 조합을 강제하거나 금칙어 설정 등 기능
* 비밀번호 유효성 체크 규칙 사용 시 `validate_password` 컴포넌트 설치 (서버에 내장되어 있음)
  * `INSTALL COMPONENT 'file://component_validate_password';`
  * 5.7 이전 버전은 컴포넌트가 아닌 플러그인 형태
    * 큰 차이 없이 동일 기능을 제공, 시스템 변수 이름에만 차이가 존재
    * 가능하면 플러그인의 단점을 보완한 컴포넌트를 사용할 것을 권고
* 확인
  * `SELECT * FROM mysql.component;`
  * `SHOW GLOBAL VARIABLES LIKE 'validate_password%';`
* 비밀번호 정책
  * `LOW`
    * 길이만 검증
      * `validate_password.length`
  * `MEDIUM`
    * 길이를 검증하며 숫자, 대소문자, 특수문자 배합을 검증
      * `validate_password.mixed_case_count`
      * `validate_password.number_count`
      * `validate_password.special_char_count`
  * `STRONG`
    * `MEDIUM` 레벨 + 금칙어 포함여부까지 검증
      * `validate_password.dictionary_file`
    * 텍스트 파일로 금지할 금칙어를 저장
    * `SET GLOBAL validate_password.dictionary_file='prohibitive_word.data'`
    * `SET GLOBAL validate_password.policy='STRONG'`

이중 비밀번호
* 기존에는 서버가 실행 중인 상태에서 비밀번호 변경은 불가능했음
* 이러한 문제점을 극복하기 위해 계정의 비밀번호로 2개의 값을 동시에 사용할 수 있는 기능 추가
  * 여기서 `Dual`은 2개의 비밀번호 중 하나만 일치하면 로그인이 통과됨을 의미
* `Primary (최근)`, `Secondary (이전)`로 구분
* 사용 시 `RETAIN CURRENT PASSWORD` 옵션 추가
  ~~~
  # 'old_password'가 `Primary` 비밀번호로 설정됨
  ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

  # 'old_password'는 `Secondary` 비밀번호가 되고, 'new_password'는 `Primary` 비밀번호가 됨
  ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;

  # `Secondary` 비밀번호 삭제
  ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD;
  ~~~

## 권한
* 5.7 이전 버전은 권한은 글로벌 권한과 객체 단위 권한으로 구분됨
  * 글로벌 권한
    * 데이터베이스나 테이블 이외의 객체에 적용되는 권한
    * `GRANT` 명령에서 특정 객체를 명시하지 않아야 함
    * `ALL` 사용 시 글로벌 수준에서 가능한 모든 권한 부여
  * 객체 권한
    * 데이터베이스나 테이블을 제어하는데 필요한 권한
    * `GRANT` 명령으로 권한을 부여할 때 반드시 특정 객체를 명시해야 함
    * `ALL` 사용 시 해당 객체에 적용될 수 있는 모든 객체 권한 부여
* 8 버전부터 동적 권한 추가 됨
  * 5.7 이전 버전까지는 `SUPER` 권한이 필수 권한이었으나 8 버전부터 동적 권한으로 쪼개짐
* 권한 부여
  * `GRANT privilege_list ON db.table TO 'user'@'host';`
  * 8 버전부터는 존재하는 사용자가 아니면 에러 발생
  * `GRANT OPTION` 권한은 다른 권한과 달리 마지막에 `WITH GRANT OPTION`을 명시해 부여
  * `privilege_list`에는 구분자(`,`)를 써서 권한 여러 개를 동시에 명시 가능
  * `TO` 뒤에는 권한을 부여할 대상 사용자 명시
  * `ON` 뒤에는 어떤 DB의 어떤 오브젝트 권한을 부여할지 결정 가능
  * 글로벌 권한
    * `GRANT SUPER ON *.* TO 'user'@'localhost';`
    * `ON` 절에 항상 `*.*`(`MySQL` 서버 전체를 의미) 사용
    * `CREATE USER`, `CREATE ROLE` 등은 항상 `*.*`로만 대상을 사용할 수 있음
  * DB 권한
    * `GRANT EVENT ON *.* TO 'user'@'localhost';`
    * `GRANT EVENT ON employees.* TO 'user'@'localhost';`
    * 서버에 존재하는 모든 DB에 권한을 부여하기 때문에 위처럼 모두 사용 가능
    * 여기서 DB는 내부에 존재하는 테이블뿐만 아니라 스토어드 프로그램까지 모두 포함
    * DB 권한만 부여하는 경우 테이블까지 명시할 수는 없음 (`db.table` 불가능)
    * 특정 DB에 대해서만 권한 부여 가능 (`db.*` 가능)
  * 테이블 권한
    * `GRANT SELECT,INSERT,UPDATE,DELETE ON *.* TO 'user'@'localhost';`
    * `GRANT SELECT,INSERT,UPDATE,DELETE ON employees.* TO 'user'@'localhost';`
    * `GRANT SELECT,INSERT,UPDATE,DELETE ON employees.department TO 'user'@'localhost';`
    * 모든 DB, 특정 DB의 오브젝트, 특정 테이블에 대해서 권한을 부여 가능
    * 특정 칼럼에 부여할 수 있는 권한 `SELECT`, `INSERT`, `UPDATE` (각 권한 뒤에 칼럼 명시)
    * `employees` DB의 `department` 테이블에서 `dept_name` 칼럼 업데이트 불가 권한 부여
      * `GRANT SELECT,INSERT,UPDATE(dept_name) ON employees.department TO 'user'@'localhost';`
* 일반적으로 테이블이나 칼럼 단위 권한은 잘 사용하지 않음
  * 칼럼 하나에 대해서만 권한을 설정하더라도 전체 성능에 영향을 줄 수 있음
    * 칼럼 단위 권한이 하나라도 설정되면, 나머지 모든 테이블의 모든 칼럼에 대해서도 권한 체크하기 때문
  * 따라서 `GRANT` 명령보다는 권한을 허용하고자 하는 특정 칼럼만으로 별도의 뷰를 만드는 것도 좋은 방법
    * 뷰 자체에 대한 권한만 체크
* 권한 확인 시 `SHOW GRANT` 명령뿐 아니라 권한 관련 테이블을 참조하여 확인 가능
      
## 역할
* 빈 껍데기 선언
  * `CREATE ROLE role_emp_read, role_emp_write;`
* `GRANT` 명령으로 실질적으로 권한 부여
  * `role_emp_read` 역할
    * `GRANT SELECT ON employees.* TO role_emp_read;`
  * `role_emp_write` 역할
    * `GRANT INSERT,UPDATE,DELETE ON employees.* TO role_emp_write;`
* 계정에 역할을 부여
  * 계정 생성
    * `CREATE USER reader@'127.0.0.1' IDENTIFIED By 'Jaenyeong123!';`
    * `CREATE USER writer@'127.0.0.1' IDENTIFIED By 'Jaenyeong123!';`
  * 역할 부여
    * `GRANT role_emp_read TO reader@'127.0.0.1';`
    * `GRANT role_emp_read, role_emp_write TO writer@'127.0.0.1';`
  * 확인
    * `SHOW GRANTS;`
    * `SELECT current_role();`
  * 역할 활성화
    * `SET ROLE 'role_emp_read';`
    * 역할 활성화를 자동으로 변경
      * `SET GLOBAL activate_all_roles_on_login=ON;`
* 서버 내부적으로 역할과 계정은 동일한 객체로 취급됨
  * 단지 하나의 사용자 계정에 다른 사용자 계정이 가진 권한을 병합해서 제어 가능
  * 실제 권한과 사용자 계정이 구분없이 저장됨
    * `SELECT user,host,account_locked FROM mysql.user;`
  * 하나의 계정에 다른 계정의 권한을 병합하기만 하면 되기 때문에 역할과 계정을 구분할 필요가 없음
  * 역할 생성 시 호스트를 명시하지 않으면 자동으로 모든 호스트가 자동으로 추가
  * 역할과 계정을 구분하려면 프리픽스, 키워드 등을 추가해 구분할 것을 권장
  * 역할의 호스트 부분은 큰 의미가 없음 (하지만 로그인하는 실제 계정처럼 사용하면 중요해짐)
* 역할과 계정은 직무를 분리할 수 있게 해 보안을 강화하는 용도
