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
> 데이터베이스 서버는 내부에 세션이라는 것을 만든다. 해당 커넷션을 통한 모든 요청은 이 세션을 통해서 실행하게 된다.  
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