### 스위치/AP 페이징 조회 API (기본 정렬 기준 수정)
- 기본 정렬 기준은 timestamp 1순위 + PK (그 중에서도 ip 2순위) 임 
- 그러나 현재 장비의 인증 상태(status) 를 나타내는 컬럼은 db 값 자체로 정렬을 하기 어려웠음. 따라서 별도의 기준을 만들어 authOrder 적용함
- 특히 ap 는 페이징 조회 당시 status 가 db 값에 들어가 있지 않고, 페이징 조회 이후 status 를 db에 넣는 특이한 구조였음.
- 따라서 AP의 상태 정렬 기준은 또 별도로 두었음

```java
// 기본 정렬 기준
query.orderBy(
      authOrder.asc(),
      qSwitch.timestamp.desc(),
      qSwitch.mgmtIp.asc()
);

// 특별히 상태 정렬 기준: authOrder
// 이 때 AP 의 상태 기준은 별도로 두어 4개의 authentication 컬럼 중 하나라도 그 상태라면 그 상태로 판정하는 로직도 추가함
private void applyDefaultSorting(JPAQuery<SwConfTotalSwitch> query, boolean isApRole) {
    BooleanExpression doneCond = isApRole
            ? qSwitch.authentication.eq(SwitchAuthState.DONE.name())
                .or(qSwitch.restAuthentication.eq(SwitchAuthState.DONE.name()))
                .or(qSwitch.httpAuthentication.eq(SwitchAuthState.DONE.name()))
                .or(qSwitch.snmpAuthentication.eq(SwitchAuthState.DONE.name()))
            : qSwitch.authentication.eq(SwitchAuthState.DONE.name());

    BooleanExpression waitingCond = isApRole
            ? qSwitch.authentication.eq(SwitchAuthState.WAITING.name())
                .or(qSwitch.restAuthentication.eq(SwitchAuthState.WAITING.name()))
                .or(qSwitch.httpAuthentication.eq(SwitchAuthState.WAITING.name()))
                .or(qSwitch.snmpAuthentication.eq(SwitchAuthState.WAITING.name()))
            : qSwitch.authentication.eq(SwitchAuthState.WAITING.name());

    BooleanExpression failCond = isApRole
            ? qSwitch.authentication.eq(SwitchAuthState.FAIL.name())
                .or(qSwitch.restAuthentication.eq(SwitchAuthState.FAIL.name()))
                .or(qSwitch.httpAuthentication.eq(SwitchAuthState.FAIL.name()))
                .or(qSwitch.snmpAuthentication.eq(SwitchAuthState.FAIL.name()))
            : qSwitch.authentication.eq(SwitchAuthState.FAIL.name());

    BooleanExpression noneCond = isApRole
            ? qSwitch.authentication.eq(SwitchAuthState.NONE.name())
                .or(qSwitch.restAuthentication.eq(SwitchAuthState.NONE.name()))
                .or(qSwitch.httpAuthentication.eq(SwitchAuthState.NONE.name()))
                .or(qSwitch.snmpAuthentication.eq(SwitchAuthState.NONE.name()))
            : qSwitch.authentication.eq(SwitchAuthState.NONE.name());

    NumberExpression<Integer> authOrder =
            new CaseBuilder()
                .when(doneCond).then(1)
                .when(waitingCond).then(2)
                .when(failCond).then(3)
                .when(noneCond).then(4)
                .otherwise(5);
}
```
