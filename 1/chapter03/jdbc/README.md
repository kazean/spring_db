# [실습] 3. 트랜잭션 이해
## 1. 트랜잭션 - 개념이해
- 데이터를 저장할 때, 데이터베이스에 저장하는 이유는 뭘까?
- 트랜잭션 ACID
> 원자성(Automicity), 일관성(Consistency), 격리성(Isolation), 지속성(Durability)
- 트랜잭션 격리 수준 - Isolation level
> READ UNCOMMITED  
READ COMMITTED  
REPEATABLE READ  
SERIALIZABLE

## 2. 데이터베이스 연결구조와 DB세션
- 데이터베이스 서버 연결 구조와 DB 세션
- 데이터베이스 연결 구조1
> 데이터베이스 서버는 내부에 세션이라는 것을 만든다. 해당 커넥션을 통한 모든 요청은 이 세션을 통해서 실행하게 된다.  
세션은 트랜잭션을 시작하고, 커밋 또는 롤백을 통해 트랜잭션을 종료한다.
- 데이터베이스 연결 구조2

## 3. 트랜잭션 - DB예제1 - 개념이해
- 트랜잭션 사용법
> 커밋을 호출하기 전까지는 임시로 데이터를 저장  
기본데이터  
세션1 데이터 입력 (아직 커밋 안한 상태), 세션2 조회할 수 없다.
- 커밋하지 않은 데이터를 다른 곳에서 조회할 수 있으면 어떤 문제가 발생할까?
> !데이터 정합성에 큰 문제
- 세션1 신규 데이터 추가 후 commit
- 세션1 신규 데이터 추가 후 rollback

## 4. 트랜잭션 - DB예제2 - 자동 커밋, 수동 커밋
- 자동 커밋
> 자동 커밋으로 설정하면 각각의 쿼리 실행 직후에 자동으로 커밋을 호출한다.
- 자동 커밋 설정
> set autocommit true;
- 수동 커밋 설정
> set autocommit false;
>> 수동 커밋모드로 설정하는 것을 트랜잭션을 시작한다고 표현할 수 있다

## 5. 트랜잭션 - DB예제3 - 트랜잭션 실습
1. 기본 데이터 입력
- h2 session 2개
- 데이터 초기화 SQL
```
set autocommit true;
delete from member;
insert into member(member_id, money) values ('oldId', 10000);
```
2. 신규 데이터 추가 - 커밋 전
- 세션1
```
set autocommit false;
insert into member(member_id, money) values('newId1', 10000);
insert into member(member_id, money) values('newId2', 20000);
```
- 세션1,2 조회
3. 커밋 - commit
- 롤백 - rollback

## 6. 트랜잭션 - DB예제4 - 계좌이체
- 계좌이체 정상
```
set autocommit true;
delete from member;
insert into member(member_id, money) values ('memberA', 10000);
insert into member(member_id, money) values ('memberB', 10000);
```
- 계좌이체 실행 SQL-성공
```
set autocommit false;
update member set money=10000-2000 where member_id = 'memberA';
update member set money=10000+2000 where member_id = 'memberB';
```
- 계좌이체 문제 상황 - 커밋
```
set autocommit false;
update member set money=10000-2000 where member_id = 'memberA';
update member set money=10000+2000 where member_iddd = 'memberB';
```
> !데이터 정합성 문제
- 계좌이체 문제 상황 - 롤백
```
set autocommit false;
update member set money=10000-2000 where member_id = 'memberA';
update member set money=10000+2000 where member_iddd = 'memberB';
```
> 문제시 데이터 복구
- 정리
> 원자성: 트랜잭션 내에서 실행한 작업들은 마치 하나의 작업인 것처럼 모두 성공하거나 모두 실패
> 오토커밋: 계좌이체 중간에 실패하면 심각한 문제 발생
> 트랜잭션 시작: 이런 종류의 작업은 꼭 수동 커밋 모드를 사용해서 수동으로 커밋, 롤백할 수 있도록 해야한다.

## 7. DB락 - 개념이해
세션1이 트랜잭션을 시작하고 데이터를 수정하는 동안 아직 커밋을 수행하지 않았는데, 세션2에서 동시에 같은 데이터를 수정하게 되면 여러가지 문제가 발생.
> 트랜잭션 원자성이 깨지는 것이다.
>> 이런 문제를 해결하기 위해 `락(Lock)`이라는 개념을 제공
- 락-1
> 세션1 memberA의 money 500 변경 > 세션2 memberA 변경시도 (대기) > 세션1 커밋 > 세션2 락 획득 후 변경 진행

## 8. DB락 - 변경
- 기본 데이터
```
set autocommit true;
delete from member;
insert into member(member_id, money) values ('memberA', 10000);
```
- 세션1
```
set autocommit false;
update member set money=500 where member_id = 'memberA'
```
- 세션2
```
SET LOCK_TIMEOUT 60000;
set autocommit false;
update member set money=1000 where member_id = 'memberA'
```
> 대기
- 세션2 락 획득: 세션1을 커밋하면 세션1이 커밋되면서 락을 반납한다. 이후 세션2 락 획득
- 세션2 락 타임아웃
> SET LOCK_TIMEOUT <milliseconds>: 락 타입아웃 시간을 설정한다.

## 9. DB락 - 조회
일반적인 조회는 락을 사용하지 않는다.
- 조회와 락
> 데이터를 조회할 때도 락을 획득하고 싶을 때, `select for update` 
- 기본 데이터
```
set autocommit true;
delete from member;
insert into member(member_id, money) values ('memberA', 10000);
```
- 세션1
```
set autocommit false;
select * from member where member_id = 'memberA' for update;
```
- 세션2
```
set autocommit false;
update member set money=1000 where member_id = 'memberA'
```
> 대기
- 세션2 락 획득: 세션1을 커밋하면 세션1이 커밋되면서 락을 반납한다. 이후 세션2 락 획득 후 변경
- 정리
> 트랜잭션과 락은 데이터베이스마다 실제 동작하는 방식이 조금씩 다르기 때문에 확인후 테스트.

## 10. 트랜잭션 - 적용1
- hello.jdbc.service.MemberServiceV1
> accountTransfer(fromId, toId, money)
- MemberServiceV1Test
> accountTransfer(), accountTransEx()
>> !예외 상황에서 데이터 정합성 문제 발생
- 테스트 데이터 제거
> @BeforeEach  
@AfterEach
>> 테스트에서 사용한 데이터를 제거하는 더 나은 방법으로는 트랜잭션을 활용하면 된다.

## 11. 트랜잭션 - 적용2
- DB 트랜잭션을 사용해서 앞서 발생한 문제점 해결해보자  
- 트랜잭션을 어떤 계층에 걸어야 할까?
- 트랜잭션은 비지니스 로직이 있는 서비스계층에서 시작해야 한다.
- *그런데 트랜잭션을 시작하려면 커넥션 필요, 애플리케이션에서 DB 트랜잭션을 사용하려면 트랜잭션을 사용하는 동안 같은 커넥션을 유지
> 가장 단순한 방법은 커넥션을 파라미터로 전달
- hello.jdbc.repository.MemberRepositoryV2
> findById(Connection con, String memberId)  
update(Connection con, String memberId, int money)  
1. 커넥션 유지가 필요한 두 메서드는 파라미터로 넘어온 con 사용, con = getConnection() 코드가 있으면 안된다.
2. 커넥션 유지가 필요한 두 메서드는 리포지토리에서 커넥션을 닫으면 안된다.
- hello.jdbc.service.MemberServiceV2
```
private final DataSource dataSource;

accountTransfer(fromId, toId, money)  
con = dataSource.getConnection;
try{
	con.setAutoCommit(false);
	bizLogic(con, fromId, toId, money);
	con.commit();
} catch (Exception e) {
	con.rollback();
} finally {
	realse(con)
}
```
> con.setAutoCommit(false); //트랜잭션 시작
- hello.jdbc.service.MemberServiceV2Test
> accountTransfer(), accountTransEx()
>> 트랜잭션 덕분에 계좌이체가 실패할 때 롤백을 수행해서 모든 데이터를 정상적으로 초기화 할 수 있게 되었다.
- 남은 문제
> 애플리케이션에서 DB 트랜잭션을 적용하려면 서비스 계층이 매우 지저분, 생각보다 매우 복잡한 코드를 요한다
>> 스프링을 사용해서 이런 문제들을 하나씩 해결