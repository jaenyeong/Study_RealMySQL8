# 아키텍처
MySQL 엔진
* 사람의 머리 역할 담당

스토리지 엔진
* 사람의 손, 발 역할 담당
* 핸들러 API를 구현한다면 누구든지 스토리지 엔진 구현 가능

## MySQL 엔진 아키텍처
전체 구조
* MySQL 서버
  * MySQL 엔진
    * SQL 문장 분석, 최적화 등 처리
    * `ANSI` 표준 문법 지원
    * 구성
      * 커넥션 핸들러
        * 클라이언트의 접속, 쿼리 요청 처리
        * 쿼리 실행기에서 데이터 처리 시 하는 요청을 핸들러(`Handler`)이라 함
          * 이때 사용되는 API를 핸들러 API라 함
        * 핸들러 API 작업 확인
          * `SHOW GLOBAL STATUS LIKE 'Handler%';`
      * SQL 인터페이스
      * SQL 파서
      * SQL 옵티마이저
      * 캐시 & 버퍼
  * 스토리지 엔진
    * 실제 데이터를 디스크 스토리이제 저장, 읽어 오는 처리
      * `CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB`
    * 성능 향상을 위해 키 캐시(`MyISAM`)나 버퍼 풀(`InnoDB`) 같은 기능을 내장
    * 구성
      * InnoDB
      * MyISAM
      * Memory
      * 기타
* 운영체제 하드웨어
  * 데이터 파일, 로그 파일

스레딩 구조
* MySQL은 프로세스가 아닌 스레드 기반으로 작동
  * 포그라운드(`Foreground`) 스레드
  * 백그라운드(`Background`) 스레드
  * 확인
    ~~~sql
    SELECT thread_id, name, type, processlist_user, processlist_host
      FROM performance_schema.threads
     ORDER BY type, thread_id;
    ~~~
  * 결과
    ~~~
    +-----------+---------------------------------------------+------------+------------------+------------------+
    | thread_id | name                                        | type       | processlist_user | processlist_host |
    +-----------+---------------------------------------------+------------+------------------+------------------+
    |         1 | thread/sql/main                             | BACKGROUND | NULL             | NULL             |
    |         2 | thread/mysys/thread_timer_notifier          | BACKGROUND | NULL             | NULL             |
    |         4 | thread/innodb/io_ibuf_thread                | BACKGROUND | NULL             | NULL             |
    |         5 | thread/innodb/io_log_thread                 | BACKGROUND | NULL             | NULL             |
    |         6 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
    |         7 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
    |         8 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
    |         9 | thread/innodb/io_read_thread                | BACKGROUND | NULL             | NULL             |
    |        10 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
    |        11 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
    |        12 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
    |        13 | thread/innodb/io_write_thread               | BACKGROUND | NULL             | NULL             |
    |        14 | thread/innodb/page_flush_coordinator_thread | BACKGROUND | NULL             | NULL             |
    |        15 | thread/innodb/log_checkpointer_thread       | BACKGROUND | NULL             | NULL             |
    |        16 | thread/innodb/log_flush_notifier_thread     | BACKGROUND | NULL             | NULL             |
    |        17 | thread/innodb/log_flusher_thread            | BACKGROUND | NULL             | NULL             |
    |        18 | thread/innodb/log_write_notifier_thread     | BACKGROUND | NULL             | NULL             |
    |        19 | thread/innodb/log_writer_thread             | BACKGROUND | NULL             | NULL             |
    |        24 | thread/innodb/srv_lock_timeout_thread       | BACKGROUND | NULL             | NULL             |
    |        25 | thread/innodb/srv_error_monitor_thread      | BACKGROUND | NULL             | NULL             |
    |        26 | thread/innodb/srv_monitor_thread            | BACKGROUND | NULL             | NULL             |
    |        27 | thread/innodb/buf_resize_thread             | BACKGROUND | NULL             | NULL             |
    |        28 | thread/innodb/srv_master_thread             | BACKGROUND | NULL             | NULL             |
    |        29 | thread/innodb/dict_stats_thread             | BACKGROUND | NULL             | NULL             |
    |        30 | thread/innodb/fts_optimize_thread           | BACKGROUND | NULL             | NULL             |
    |        31 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
    |        32 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
    |        33 | thread/mysqlx/worker                        | BACKGROUND | NULL             | NULL             |
    |        37 | thread/innodb/buf_dump_thread               | BACKGROUND | NULL             | NULL             |
    |        38 | thread/innodb/clone_gtid_thread             | BACKGROUND | NULL             | NULL             |
    |        39 | thread/innodb/srv_purge_thread              | BACKGROUND | NULL             | NULL             |
    |        40 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
    |        41 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
    |        42 | thread/innodb/srv_worker_thread             | BACKGROUND | NULL             | NULL             |
    |        44 | thread/sql/signal_handler                   | BACKGROUND | NULL             | NULL             |
    |        45 | thread/mysqlx/acceptor_network              | BACKGROUND | NULL             | NULL             |
    |        43 | thread/sql/event_scheduler                  | FOREGROUND | event_scheduler  | localhost        |
    |        47 | thread/sql/compress_gtid_table              | FOREGROUND | NULL             | NULL             |
    |        48 | thread/sql/one_connection                   | FOREGROUND | root             | localhost        |
    +-----------+---------------------------------------------+------------+------------------+------------------+
    ~~~
    * 39개의 결과 (책 내용과 다를 수 있음)
      * `foregroud`는 3개 이 중 `one_connection` 스레드만 실제 사용자의 요청을 처리하는 스레드
      * 동일한 이름의 스레드가 2개 이상 있는 경우는 MySQL 서버 설정에 따라 동일 작업이 여러 스레드로 병렬 처리되는 경우
    * 커뮤니티 에디션에서 사용되는 모델은 전통적인 스레드 모델
      * 엔터프라이즈 에디션에서는 스레드 풀 모델도 사용 가능
      * 둘의 차이는 포그라운드 스레드와 커넥션 관계
        * 전통적 스레드 모델은 커넥션 별로 포그라운드 스레드가 하나씩 생성
        * 스레드 풀 모델은 1:1 관계가 아닌 하나의 스레드가 여러 개의 커넥션 요청을 전담
* 포그라운드 스레드(클라이언트 스레드)
  * 최소한 MySQL 서버에 접속된 클라이언트 수만큼 존재
    * 주로 사용자가 요청하는 쿼리 문장 처리
  * 커넥션 종료 시 스레드 캐시로 되돌아 감
    * 일정 수 이상 대기 스레드가 있으면 캐시로 되돌아가지 않고 종료
    * 이때 스레드 캐시 유지할 최대 스레드 수는 `thread_cache_size` 시스템 변수로 설정
  * 포그라운드 스레드는 데이터를 MySQL 데이터 버퍼나 캐시로 가져옴
    * 캐시에 없는 경우 직접 디스크의 데이터나 인덱스 파일로부터 읽어 처리
    * `MyISAM` 테이블은 디스크 작업까지 포그라운드 스레드가 처리
    * `InnoDB` 테이블은 데이터 버퍼나 캐시까지만 처리 포그라운드 스레드가 처리
      * 나머지 버퍼로부터 디스크까지 작업은 백그라운드 스레드가 처리
  * MySQL에서는 사용자 스레드와 포그라운드 스레드가 동일한 의미
    * 클라이언트가 MySQL 서버에 접속하게 되면 요청을 처리할 스레드를 생성해 클라이언트에게 할당
    * 이 스레드가 앞단에서 통신하기 때문에 포그라운드 스레드라고 함
    * 또한 사용자가 요청한 작업을 처리하기 때문에 사용자 스레드라고도 함
* 백그라운드 스레드
  * 다음과 같은 작업이 백그라운드에서 처리됨 (`MyISAM` 경우는 해당하지 않을 수 있음)
    * 인서트 버퍼를 병합하는 스레드
    * 로그를 디스크로 기록하는 스레드
    * `InnoDB` 버퍼 풀의 데이터를 디스크에 기록하는 스레드
    * 데이터를 버퍼로 읽어 오는 스레드
    * 잠금이나 데드락을 모니터링하는 스레드
  * 로그 스레드, 버퍼의 데이터를 기록하는 스레드가 특히 중요
    * 5.5 버전부터 데이터 쓰기, 읽기 스레드 수를 2개 이상 지정 가능
      * `innodb_write_io_threads`, `innodb_read_io_threads` 시스템 변수로 스레드 수 설정
    * 쓰기 스레드는 많은 작업을 백그라운드에서 처리 되어 일반 내장 디스크 사용 시 `2~4`개 정도 설정 권장
      * `DAS`, `SAN` 같은 스토리지 사용 시 최적으로 사용할 수 있을 만큼 충분히 설정하는 것을 권장
  * 읽기 작업은 절대 지연될 수 없으나 데이터 쓰기 작업은 지연(버퍼링) 가능
    * `InnoDB`는 쓰기 작업을 버퍼링하여 일괄 처리
      * `CUD` 쿼리로 인한 데이터 변경은 완전히 저장될 때까지 기다리지 않아도 됨
    * `MyISAM`은 사용자 스레드가 쓰기 작업까지 처리

메모리 할당 및 사용 구조
* MySQL 서버가 시작되면서 OS로부터 할당
    * OS에 따라 요청 메모리 공간을 100% 다 할당하거나 또는 필요한 양만 조금씩 할당하기도 함
  * 서버가 사용하는 정확한 메모리 양을 측정하는 것은 어려움
    * 따라서 시스템 변수로 설정한만큼 모두 할당 받는다고 생각해도 문제 없음
* 글로벌 메모리 영역
  * 일반적으로 클라이언트 스레드 수와 무관하게 하나의 메모리 공간만 할당
    * 상황에 따라 필요 이상의 공간을 할당 받을 수 있지만, 클라이언트 스레드 수와 무관
    * 여러 영역이 생성되도 모든 스레드가 공유함
  * 대표적 글로벌 메모리 영역
    * 테이블 캐시
    * `InnoDB` 버퍼 풀
    * `InnoDB` 어댑티브 해시 인덱스
    * `InnoDB` 리두 로그 버퍼
    * `MyISAM` 키 캐시
    * 바이너리 로그 버퍼
* 로컬 메모리 영역
  * 세션 메모리 영역이라고도 함
    * 클라리언트(요청) 스레드가 사용하는 메모리 공간이라 클라이언트 메모리 영역이라고도 함
  * MySQL 서버상에 존재하는 클라이언트 스레드가 쿼리를 처리할 때 사용되는 영역
  * 각 클라이언트 스레드 별로 할당되기에 절대 공유되지 않음
    * 일반적으로 크게 주의(설정)해서 사용하지 않지만 최악의 경우 서버가 멈출 수도 있음
      * 적절히 메모리 공간을 할당(설정)할 것
    * 쿼리 용도로 필요할 때만 할당, 필요 없다면 할당자체를 하지 않을 수 있음
      * 소트 버퍼, 조인 버퍼 등이 대표적 예시
  * 커넥션이 열려 있는 동안 계속 할당된 상태로 남아 있는 공간도 있지만, 쿼리 실행 시에만 할당되고 금방 해제하는 공간도 있음
    * 소트 버퍼, 조인 버퍼 등이 대표적 예시
  * 대표적인 로컬 메모리 영역
    * 정렬 버퍼
    * 조인 버퍼
    * 바이너리 로그 캐시
    * 네트워크 버퍼
    * 리드 버퍼

플러그인 스토리지 엔진 모델
* MySQL은 플러그인 모델 형태로 구성되어 있음
  * 기본적으로 제공되는 기능 이외에 직접 개발하여 사용 가능
* MySQL의 대부분 작업이 엔진에서 처리, 마지막 데이터 읽기/쓰기만 스토리지 엔진에서 처리
  * 엔진을 직접 개발하더라도 전체가 아닌 일부분만 수행하는 엔진을 작성하게 되는 것
  * `데이터 읽기/쓰기` 작업은 대부분 1건의 레코드 단위로 처리됨
    * 특정 인덱스 레코드 1건 또는 마지막 읽은 레코드의 전이나 후 레코드 읽기 등
* 핸들러
  * 어떤 기능을 호출하기 위해 사용하는 객체 (MySQL 서버의 소스코드로부터 넘어온 표현)
    * 프로그래밍 언어에서는 어떤 기능을 호출하기 위해 사용하는 객체를 핸들러라고 표현
  * MySQL 엔진이 스토리지 엔진을 조정하기 위해 핸들러를 사용 (반드시 거침)
  * `Handler_`로 시작하는 상태 변수
    * 이는 MySQL 엔진이 각 스토리지 엔진에게 보낸 명령 횟수를 의미
* 서로 다른 스토리지 엔진을 사용하는 테이블에 쿼리를 실행하면 MySQL 처리 내용은 대부분 동일
  * 데이터 읽기/쓰기 영역만 다름
    * 다만 이 부분 작업 처리 방식이 달라질 수 있음
  * 실질적인 `group by`, `order by` 등 복잡한 처리는 `쿼리 실행기`에서 처리
  * 따라서 쿼리에 포함된 하위작업들이 어디서 처리되는지 알아야 함
* 지원되는 스토리지 엔진 확인
  * `SHOW ENGINES;`
    ~~~
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    | Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    | ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
    | BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
    | MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
    | FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
    | MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
    | PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
    | InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
    | MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
    | CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    9 rows in set (0.02 sec)
    ~~~
    * `Support` 컬럼에 표시되는 값
      * `YES`
        * 해당 서버에 포함되어 있고 사용 가능
      * `NO`
        * 해당 서버에 포함되지 않음
      * `DEFAULT`
        * `YES`와 같으나 필수 스토리지 엔진을 의미
        * 이 스토리지 엔진이 없다면 서버가 동작하지 않을 수도 있음
      * `DISABLED`
        * 해당 서버에 포함되어 있지만 파라미터에 의해 비활성화된 상태
  * 기존에 없는 스토리지 엔진을 포함하려면 서버 재빌드 필요
  * 플러그인 형태로 빌드된 스토리지 엔진을 다운로드, 삽입하면 쉽게 사용 가능
    * 플러그인 형태는 업그레이드도 용이
* 적용한 모든 플러그인 확인
  * 다양한 플러그인 확인 가능
    * 스토리지 엔진
    * 인증 및 전문 검색용 파서
    * 쿼리 재작성
    * 비밀번호 검증
    * 커넥션 제어
    * 기타
  * `SHOW PLUGINS;`
    ~~~
    +---------------------------------+----------+--------------------+---------+---------+
    | Name                            | Status   | Type               | Library | License |
    +---------------------------------+----------+--------------------+---------+---------+
    | binlog                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | mysql_native_password           | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
    | sha256_password                 | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
    | caching_sha2_password           | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
    | sha2_cache_cleaner              | ACTIVE   | AUDIT              | NULL    | GPL     |
    | daemon_keyring_proxy_plugin     | ACTIVE   | DAEMON             | NULL    | GPL     |
    | CSV                             | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | MEMORY                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | InnoDB                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | INNODB_TRX                      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP                      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_RESET                | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMPMEM                   | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMPMEM_RESET             | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_PER_INDEX            | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_PER_INDEX_RESET      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_PAGE              | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_PAGE_LRU          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_POOL_STATS        | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_TEMP_TABLE_INFO          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_METRICS                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_DEFAULT_STOPWORD      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_DELETED               | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_BEING_DELETED         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_CONFIG                | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_INDEX_CACHE           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_INDEX_TABLE           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_TABLES                   | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_TABLESTATS               | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_INDEXES                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_TABLESPACES              | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_COLUMNS                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_VIRTUAL                  | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CACHED_INDEXES           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SESSION_TEMP_TABLESPACES | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | MyISAM                          | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | MRG_MYISAM                      | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | PERFORMANCE_SCHEMA              | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | TempTable                       | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | ARCHIVE                         | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | BLACKHOLE                       | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | FEDERATED                       | DISABLED | STORAGE ENGINE     | NULL    | GPL     |
    | ngram                           | ACTIVE   | FTPARSER           | NULL    | GPL     |
    | mysqlx_cache_cleaner            | ACTIVE   | AUDIT              | NULL    | GPL     |
    | mysqlx                          | ACTIVE   | DAEMON             | NULL    | GPL     |
    +---------------------------------+----------+--------------------+---------+---------+
    45 rows in set (0.01 sec)
    ~~~

컴포넌트
8버전부터 기존의 지원되는 플러그인 아키텍처를 대체하기 위한 아키텍처 (기존 플러그인 단점 보완)
* 기존 플러그인 단점
  * 오직 MySQL 서버와 인터페이스할 수 있음 (플러그인끼리 통신 불가능)
  * MySQL 서버의 변수, 함수를 직접 호출해서 안전하지 않음 (캡슐화 불가능)
  * 상호 의존관계를 설정할 수 없어 초기화 어려움
* 대표적으로 비밀번호 검증 기능이 8버전부터 컴포넌트로 개선
* `validate_password` 컴포넌트 설치 예시
  * `INSTALL COMPONENT 'file://component_validate_password';`
  * 성공 시 `Query OK, 0 rows affected (0.01 sec)` 출력
  * 하지만 기존에 해당 명령으로 컴포넌트 추가가 이미 되어있다면 에러 발생
    * `ERROR 3529 (HY000): Cannot load component from specified URN: 'file://component_validate_password'.`
    * 삭제 후 다시 설치 가능
      * `UNINSTALL COMPONENT 'file://component_validate_password';`

쿼리 실행 구조
* 쿼리 파서
  * 사용자 요청으로 들어온 쿼리 문장을 토큰으로 분리, 트리 형태로 구성
    * 토큰은 MySQL이 인식할 수 있는 최소 단위의 어휘, 기호
  * 쿼리 문법 오류를 잡아내는 단계
* 전처리기
  * 파서에 의한 파싱 후 생성된 트리의 구조적인 문제를 파악
  * 테이블명, 컬럼명, 내장 함수 등 객체 존재 여부, 접근 권한 등을 확인하는 단계
* 옵티마이저
  * 요청된 문장을 가장 저렴한 비용으로 빠르게 처리하는 방법을 확인
* 실행 엔진
  * 옵티마이저가 생성한 계획대로 각 핸들러에게 요청, 받은 결과를 다시 다른 핸들러 요청으로 연결
  * 핸들러에게 명령을 내려 조합, 최종 결과를 사용자 또는 다른 모듈로 넘김
* 핸들러(스토리지 엔진)
  * 가장 밑단에서 데이터를 디스크로 저장하거나 읽어오는 역할
  * 각 스토리지엔진에(`InnoDB`, `MyISAM` 등) 맞는 핸들러가 설정됨

복제
* 16장에서 설명

쿼리 캐시
* SQL 실행 결과를 메모리에 캐시, 동일 쿼리 요청 시 캐싱된 결과를 즉시 반환하는 기능
* 8버전부터 해당 기능 및 관련된 변수 모두 삭제
  * 테이블의 데이터가 변경되면 관련된 캐싱된 결과는 모두 삭제 처리해 심각한 동시 처리 성능 저하를 유발
  * 실제 쿼리 캐시가 도움된 경우는 거의 없었음
  * 오히려 수많은 버그의 원인

스레드 풀
* 스레드 풀 기능은 엔터프라이즈 에디션 버전에서 제공됨
* 내부적으로 사용자 요청을 처리하는 스레드 수를 줄여 서버 리소스를 줄이는 것이 목적
  * 동시 처리되는 요청이 많아도 서버의 CPU가 제한된 수의 스레드 처리만 집중할 수 있게 제한
* 커뮤니티 버전에서는 `Percona Server`에서 제공하는 스레드 풀 사용
  * 플러그인 라이브러리 형태, 설치하여 사용
  * 단순히 적용한다고 성능이 개선되지 않음
    * 실제로 눈에 띄는 성능 향상을 보여주는 경우는 드묾
    * 스케줄링 과정에서 CPU 시간을 제대로 확보하지 못한자면 쿼리 처리가 더 느려지기도 함
  * 다만 제한된 수의 스레드만으로 CPU가 처리하도록 유도하면
    CPU의 프로세서 친화도를 높이고 OS의 불필요한 컨텍스트 스위칭을 줄일 수 있음
* `Percona Server`의 스레드 풀
  * 기본적으로 CPU 코어 수만큼 스레드 그룹 생성 (`thread_pool_size` 시스템 변수로 변경, 조정 가능)
    * 일반적으로 CPU 코어 수와 맞추는 것이 CPU 프로세서 친화도를 높이는 데 좋음
  * 이미 스레드 풀이 처리 중인 작업이 있는 경우
    * `thread_pool_oversubscribe` 시스템 변수에 설정된 수만큼 추가로 받아들여 처리
    * 이 값이 너무 크면 스케줄링 할 스레드가 많아져 스레드 풀이 비효율적으로 작동할 수도 있음
  * 스레드 그룹의 모든 스레드가 일을 처리하고 있는 경우
    * 스레드 풀은 해당 스레드 그룹에 새로운 작업 스레드를 추가 또는 기존 작업 스레드 처리 완료를 기다릴지 판단
    * 스레드 풀의 타이머 스레드는 주기적으로 스레드 그룹의 상태 체크
      * 기존 작업 스레드가 `thread_pool_stall_limit` 시스템 변수의 밀리초내에 끝나지 않으면 새 스레드 생성, 추가
        * 이때 스레드 풀의 스레드 수는 `thread_pool_max_threads` 변수 설정 값을 넘어갈 수 없음
      * 즉 새로운 쿼리 요청이 있어도 `thread_pool_stall_limit` 시간 만큼 대기
        * 따라서 응답시간이 예민한 서비스라면 적절히 낮춰 설정할 필요가 있음
        * 반대로 0에 가까운 값을 설정하면 의미가 없음 (설정하지 않는 편이 나음)
  * 선순위 큐, 후순위 큐를 이용해 특정 트랜잭션이나 쿼리를 우선적으로 처리하는 기능 제공
    * 트랜잭션 내에 SQL을 빨리 처리하면 잠금해제가 빠르게 됨 (잠금 경합 낮춤)

트랜잭션 지원 메타데이터
* 메타데이터(데이터 딕셔너리)
  * 테이블 구조 정보와 스토어드 프로그램 등의 정보를 표현
* 5.7 이전버전
  * `FRM` 파일에 테이블 구조 및 일부 스토어드 프로그램을 저장
  * 생성, 변경 작업이 실행 중 서버가 비정상적 종료되면 테이블 등이 깨짐
    * 생성, 변경 작업이 트랜잭션을 지원하지 않아 파일이 일관되지 않은 상태로 종료 됨
* 8버전 이후
  * 테이블 구조 정보, 스토어드 프로그램, 시스템 테이블 등 정보를 `InnoDB` 테이블에 저장
    * 시스템 테이블, 데이터 딕셔너리 모두 모아 `mysql` DB에 저장
      * `mysql` DB는 통재로 `mysql.ibd` 테이블스페이스의 저장됨
      * `*.ibd`는 모두 주의할 것
  * 스키마 변경 작업 중 실패 시 완전 성공 또는 완전 실패로 귀결
* 데이터 딕셔너리를 저장하는 테이블은 조회 되지 않음
  * `mysql` DB의 존재, 뷰를 통해서 확인 가능
    * 예를 들어 `information_schema` DB의 `TABLES`, `COLUMNS` 등
  * 따라서 조회 시 `테이블 없음`이 아니라 `접근 거절` 에러 발생
    * `SELECT * FROM mysql.tables LIMIT 1;`
      * `[HY000][3554] Access to data dictionary table 'mysql.tables' is rejected.`
* `MyISAM`, `CSV` 등 메타 정보는 여전히 저장 공간이 필요
  * `InnoDB` 외에 스토리지 엔진은 `SDI(Serialized Dictionary Information)` 파일을 사용하여 저장
    * 기존의 `*.FRM`과 동일한 역할
  * `InnoDB` 테이블 구조 또한 `SDI`로 변환(직렬화) 가능
  * `ibd2sdi` 유틸리티 이용하면 `InnoDB` 테이블스페이스에서 스키자 정보 추출 가능

## `InnoDB` 스토리지 엔진 아키텍처
* MySQL에서 지원하는 스토리지 엔진 중 거의 유일하게 레코드 기반 잠금 제공
* 높은 동시성 처리 가능, 안정적이며 성능이 뛰어남

프라이머리 키에 의한 클러스터링
* `InnoDB` 테이블은 기본적으로 프라이머리 키를 기준으로 클러스터링 되어 저장됨
  * 프라이머리 키 값 순서대로 디스크에 저장
  * 모든 세컨더리 인덱스는 레코드의 주소 대신 프라이머리 키 값을 논리적인 주소로 사용
  * 프라이머리 키를 이용한 레인지 스캔은 성능적으로 빠름
    * 프라이머리 키가 클러스터링 인덱스이기 때문
  * 일반적으로 프라이머리 키는 다른 보조 인덱스보다 비중이 높게 설정
    * 실행 계획에서 다른 인덱스보다 프라이머리 키가 선택될 확률이 높음
* 오라클의 `IOT(Index organized table)` 구조가 `InnoDB`에서는 일반적인 테이블 구조
* `MyISAM`에서는 클러스터링 키를 지원하지 않음
  * 따라서 프라이머리 키, 세컨더리 인덱스의 구조적 차이가 없음
    * 프라이머리 키는 유니크 제약을 가진 세컨더리 인덱스
  * 프라이머리 키 + 모든 인덱스는 물리적인 레코드의 주소 값(`ROWID`)을 가짐

외래 키 지원
* 외래 키는 `InnoDB`에서만 사용 가능
* 적용되어 있다면 데이터 처리 중 부모 및 자식 테이블에 데이터 확인 > 테이블 락이 전파됨
  * 이로 인한 데드락 주의
* 외래키가 복잡하게 얽혀 있는 경우 `SET foreign_key_checks = OFF or ON` 옵션 사용 가능
  * 복잡한 외래키 관계 유효성 체크를 피할 수 있어 속도가 빠름
  * 다만 해당 옵션 사용 시 데이터 정합성이 깨지면 그대로 반영되기 때문에 주의할 것

MVCC(`Multi Version Concurrency Control`)
* 일반적으로 레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능
* 잠금을 사용하지 않는 일관된 읽기를 제공함이 목적
* `InnoDB`는 `Undo log`를 통해 이 기능을 구현
  * `Update` 등의 명령 시 기존 데이터는 언두 로그에 보관되며 버퍼 풀에는 변경된 데이터 저장
    * 트랜잭션 격리 수준에 따라 언두 로그 또는 버퍼 풀 데이터를 읽어 다른 트랜잭션에서 별도 잠금 없이 조회 가능
    * 예를 들어 격리 수준이 `READ_UNCOMMITED` 버퍼 풀이나 데이터 파일로부터 변경되지 않는 데이터 조회(반환)
    * 격리 수준이 `READ_COMMITED` 또는 그 이상이면 언두 로그에 데이터 조회(반환)
  * 디스크 데이터 파일은 체크 포인트나 `Write` 스레드에 의해 값이 수정되기 때문에 변경 시점 정확하지 않음
    * 다만 `ACID` 특성을 보장하기 때문에 일반적으로 버퍼 풀과 디스크 데이터 파일이 동일한 상태로 간주해도 무방
  * 언두 로그는 트랜잭션이 길어지거나 하는 상황에 따라 사용되는 공간이 많이 늘어날 수 있음
    * 트랜잭션이 길어질 수록 예전 데에터가 오랫동안 삭제되지 못하고 관리되어야 함
    * 동시 요청 건수에 따라 관리 대상이 되는 예전 버전 데이터는 무한히 많아질 수 있음
* 커밋 시 수정 사항을 영구적인 데이터로 확정
  * 롤백하면 언두 영역에 데이터로 버퍼 풀 영역을 복구
  * 언두 영역은 필요로 하는 트랜잭션이 없어진 후 소멸

잠금 없는 일관된 읽기(`Non-Locking Consistent Read`)
* `InnoDB`는 `MVCC` 기술을 통해 잠금을 기다리지 않고 읽기 가능 (바로 실행)
  * 위에서 언급했듯 트랜잭션 격리 수준에 따라 버퍼 풀 / 언두 로그를 선택해 읽음
* 오랜 시간 트랜잭션이 활성 상태라면 서버의 성능 문제를 야기할 수 있음
  * 일관된 읽기를 위해 언두 로그를 삭제하지 못하고 유지해야 하기 때문
  * 따라서 트랜잭션은 가급적 빨리 커밋 또는 롤백하는 것을 권장

자동 데드락 감지
* `InnoDB`는 데드락 감지를 위해 잠금 대기 목록(`Wait-for-List`)을 그래프 형태로 관리
  * 데드락 감지 스레드가 이를 주기적으로 검사, 데드락인 트랜잭션 중 하나를 강제 종료
  * 이때 일반적으로 언두 로그 양이 적은 것을 기준으로 강제 종료(롤백)
    * 이는 처리할 내용이 적다는 것을 의미, 서버의 부하를 조금이라도 줄일 수 있음
* `InnoDB` 엔진은 `MySQL` 엔진에서 관리되는 테이블 락을 볼 수 없어 감지가 불확실
  * 이때 `innodb_table_locks` 시스템 변수를 통해 레코드 레벨 잠금뿐 아니라 테이블 레벨 잠금도 감지 가능
    * 특별한 이유가 없다면 활성화 해둘 것을 권장
  * 동시 처리 스레드가 많다면 데드락 감지 작업이 느려질 수 밖에 없음
    * 잠금 목록 검사 시 잠금 상태가 변경되지 않도록 잠금 대기 목록의 테이블에 새로운 잠금을 걸고 데드락 스레드를 찾음
    * 이를 위해 `innodb_deadlock_detect` 시스템 변수를 `OFF` 설정하여 데드락 감지 스레드 비활성화
      * 하지만 위 설정으로 데드락 상황에 무한 대기에 빠질 수 있음
      * `innodb_lock_wait_timeout` 시스템 변수를 활성화 해 잠금 제한 시간 설정
        * 데드락 발생 시 일정 시간이 지나면 해당 요청 실패 처리
  * 구글에서 데드락 감지 스레드의 성능 저하 이슈 확인

자동화된 장애 복구

`InnoDB` 버퍼 풀

Double Writer Buffer

언두 로그

## `MyISAM` 스토리지 엔진 아키텍처
키 캐시

운영체제의 캐시 및 버퍼

데이터 파일과 프라이머리 키(인덱스) 구조

## MySQL 로그 파일
에러 로그 파일

제네럴 쿼리 로그 파일(제너럴 로그 파일, General log)

슬로우 쿼리 로그
