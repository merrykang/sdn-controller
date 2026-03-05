### 문제 상황
- 배경: 클라우드 서비스 업체에서 데이터 볼륨 파티션 분리를 위해 db 서버 vm 을 종료, 재부팅을 계속 해서 하는 상황
- db 공통 운영 장애 상황
- systemctl start mariadb 가 되지 않음
- `/var/log/mysql/error.log` 와 `journalctl -u mariadb -xe` 2개의 로그를 확인하였을 때 WSREP 관련 에러가 보임
- 갈레라 클러스터링 설정 문서를 확인하였을 때 모든 클러스터링 노드에서 seqno: -1  & safe_to_bootstrap: 0 설정 확인 가능함
- 따라서 해당 상황이 클러스터링이 깨진 split brain 상황이라고 판단하고, 설정 문서를 수정하여 새로운 갈레라 클러스터링을 생성해야겠다고 판단함

---

### 해결 방법
- **둘 중 아무 노드 1대를 골라 강제 Bootstrap**
  -  두 노드의 데이터 차이가 거의 없다고 판단할 수 있을 때
  - 가장 현실적인 방법
- 그러나 추가 문제 발생함. 투표 노드를 고려하지 않음. 또한 mysql 을 접속했을 때 클러스터링 상태가 ON 으로 되어 있지 않았음

```bash
# 모든 노드에서 MariaDB 완전히 중지
systemctl stop mariadb
ps -ef | grep mariadbd  
## 이 명령어로 확인했을 때 완전히 아무 것도 없어야함. 
## 또는 이렇게 나오면 됨 root     2982235  343100  0 14:29 pts/0    00:00:00 grep --color=auto mariadbd

# Bootstrap할 “정답 노드” 1대 선정 : 202 DB1
vi /var/lib/mysql/grastate.dat
safe_to_bootstrap: 1  ## 정답 노드에서만 이렇게 수정

# 위처럼 실행하고 정답 노드에서만 클러스터 생성 명령어 입력
galera_new_cluster

# 정답 노드가 아닌 노드들에서 
systemctl start mariadb
```
