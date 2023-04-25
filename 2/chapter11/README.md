# [이론] 11.스프링 트랜잭션 전파2 - 활용
## 1.트랜잭션 전파 활용1 - 예제 프로젝트 시작
- 비지니스 요구 사항
> 회원을 등록하고 조회한다  
회원에 대한 변경 이력을 추적할 수 있도록 변경 이력을 DB LOG 테이블(등록 시)
- hello.springtx.propgation.Member
- MemberRepository
> save(), find()
- Log
- LogRepository
> save(): message.contains("로그예외") > RuntimeEx, find()
- MemberService
> joinV1(String username), ~V2() //Memeber,LogRepo

- MemberServiceTest
> outerTxOff_success()
- cf
> JPA구현체 auto-create, 메모리DB, JPA(등록, 수정, 삭제) > Transaction

## 2.트랜잭션 전파 활용2 - 커밋, 롤백
- 서비스 계층에 트랜잭션 없을 때 - 커밋
> 각자 다른 트랜잭션
- @Transactional과 REQUIRED
> `@Transaction(propagation = Propagation.REQUIRED) = @Transactional`
- 서비스 계층에 트랜잭션이 없을 때 - 롤백
- outerTxOff_fail()
> Member: commit, Log: rollback
>> !데이터 정합성 문제

## 3.트랜잭션 전파 활용3 - 단일 트랜잭션
- 트랜잭션 하나만 사용하기
- MemberServiceTest: singleTx
> MemberService: Transaction  
MemberRepository: //Transaction  
LogRepository: //Transaction
>> 정합성 문제 해결
- 각각 트랜잭션이 필요한 상황
> 트랜잭션 전파 없이 해결하려면 트랜잭션이 있는 메서드와 트랜잭션이 없는 메서드 각각 만들어야 할 것이다.

## 4.트랜잭션 전파 활용4 - 전파 커밋
스프링은 @Transactional이 적용되어 있으면 기본적으로 REQUIRED라는 전파 옵션을 사용
- outerTxOn_success()
> MemberService, MemberRepository, LogRepository: @Transaction: on
>> 정합성 문제 해결, 트랜잭션 참여

## 5.트랜잭션 전파 활용5 - 전파 롤백
- outerTxOn_fail()
> Member.isPresnet, Log.isEmpty

## 6.트랜잭션 전파 활용6 - 복구 REQUIRED
회원 가입을 시도한 로그를 남기는데 실패하더라도 회원 가입은 유지되어야 한다.
> 예외 잡아서 정상로직 변환?
>> !rollback-Only 표시후 예외 발생, 커밋 안된다
- recoverException_fail()
> 예외 잡아서 처리 하여도 전체 rollback, UnexpectedRollbackException > rollback-Only 체크

- 정리
> 논리 트랜잭션중 하나라도 롤백되면 전체 트랜잭션은 롤백된다.  
내부 트랜잭션이 롤백 되었는데, 외부 트랜잭션이 커밋되면 UnexpectedRollbackException 예외  
rollbackOnly상황에서 커밋이 발생하면 UnexpectedRollbackException 예외

## 7.트랜잭션 전파 활용7 - 복구 REQUIRES_NEW
- recoverException_success
> LogRepository: @Transaction(propagation = Propagation.REQUIRES_NEW)
>> Log 롤백되어도 Member는 커밋
- 정리
> 논리 트랜잭션은 하나라도 롤백되면 관련된 물리 트랜잭션은 롤백되어 버린다.  
이 문제를 해결하려면 REQUIRES_NEW를 사용해서 트랜잭션을 분리해야 한다.
- 주의
> REQUIRES_NEW를 사용하면 하나의 HTTP 요청에 동시에 2개의 데이터베이스 커넥션을 사용하게 된다.  
따라서 성능이 중요한 곳에서는 이런 부분을 주의해서 사용해야 한다.