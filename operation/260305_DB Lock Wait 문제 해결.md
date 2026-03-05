### 문제점 및 오류 메시지
- 웹 접속 안 됨
- 스프링 로그에서 발견한 오류 메시지
`Caused by: java.sql.SQLTransientConnectionException: (conn=2366832) Lock wait timeout exceeded; try restarting transaction`

---

### 문제 원인 발견 과정 및 해결 방법
- gpt : Lock wait timeout exceeded 오류는 **다른 트랜잭션이 잡고 있는 InnoDB row/table lock을 오래 기다리다가 timeout이 발생** 
- 그래서 먼저 db 의 프로세스를 확인함. 오래 걸리고 규모 큰 프로세스 있으면 제거하려고 했음. 실제로 저번에도 그렇게 해서 db 정상 기동되지 않는 문제를 해결함
  -  mysql 접속하여 `SHOW FULL PROCESSLIST;`
  -  그러나 프로세스가 너무 많아서 어떤 것을 삭제 해야 하는지 모르는 상태
- 그래서 db 관련 리눅스 프로세스 자체를 확인하려고 `ps -ef | grep mariadb` 명령어 실행함
  - 아래의 프로세스 중 `mysql    1703152       1 99 01:30 ?        16:26:24 /usr/sbin/mariadbd --wsrep_start_position=7516482c-0e1d-11f1-b001-476e71e20205:4508134846` 을 제거하고 웹서버 재시작했더니 정상화 됨
![프로세스 사진](/images/process_list.jpg)
- 해결된 이유
  - 프로세스 사진 기반으로 해석해보면 `MariaDB 서버 프로세스가 CPU 99% 상태로 내부에서 lock / trx 처리로 busy`하다는 뜻
  - db 가 강제 종료되면서 
    1️⃣ 모든 transaction rollback
    2️⃣ 모든 row lock 해제
    3️⃣ InnoDB recovery 수행
    4️⃣ DB 정상 상태로 재시작

---

### 방지 방법
- 장비 수집할 때 update 와 set 같은게 동시 발생하여 데이터 변경하는데 인덱스가 없는 경우 찾아서 인덱스 걸어주기
- 따라서 이 문제를 추적하고 방지하려면 `SELECT * FROM information_schema.innodb_lock_waits;` 실행해서 락 상태 확인하고 그걸 해결해줘야함

---

### 관련 학습 사항
- InnoDB row/table lock
  - MySQL/MariaDB의 **InnoDB 스토리지 엔진이 데이터의 동시 접근을 제어하기 위해 거는 잠금(lock)**
  - 여러 트랜잭션이 동시에 같은 데이터를 변경할 때 데이터 충돌을 막기 위한 보호 장치
  - row lock(행 잠금), table lock(테이블 잠금) 이 존재함
  - row lock 도 인덱스가 없으면 table lock 처럼 동작함
  - 예시: update 하고 set 하는 경우 
- `SHOW FULL PROCESSLIST;` vs `ps -ef | grep mariadb`
  - 전자: MariaDB/MySQL 내부에서 실행 중인 세션과 쿼리를 보여주는 명령어
  - 후자: OS에서 실행 중인 프로세스를 확인하는 명령어
