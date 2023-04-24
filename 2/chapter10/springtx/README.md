# [실습] 10.스프링 트랜잭션 전파1 - 기본
## 1.스프링 트랜잭션 전파1 - 커밋, 롤백
트랜잭션이 둘 이상 있을때 어떻게 동작하는지, 스프링이 제공하는 `트랜잭션 전파(Propagation)`라는 개념
- hello.springtx.propagation.BasicTxTest
```
@SpringBootTest{
	@Autowired PlatformTransactionManager txManager

	@TestConfiguration
	Config { ~ }

	@Test
	commit();
	rollback();
}
```
- application.yml
```
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
logging.level.org.springframework.jdbc.support.JdbcTransactionManager=DEBUG
#JPA log
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
logging.level.org.hibernate.resource.transaction=DEBUG
```
> commit(), rollback()

## 2.스프링 트랜잭션 전파2 - 트랜잭션 두 번 사용
- double_commit() - BasicTxTest 추가
> Acquired Connection HikariProxyConnection@1 conn0, @2 conn0
- 주의
> `로그를 보면 tx1과 tx2가 같은 conn0 커넥션 사용, 반납후 conn0 획득한 것, 객체의 주소가 다름으로 다른 커넥션 구분`  
트랜잭션이 각각 수행되면서 사용되는 DB 커넥션도 각각 다르다  
트랜잭션을 각자 관리하기에 전체 트랜잭션을 묶을수 없다
- double_commit_rollback() - BasicTxTest 추가
> 각자 다른 트랜잭션

## 3.스프릥 트랜잭션 전파3 - 전파 기본
트랜잭션을 각각 사용하는 것이 아니라, 트랜잭션이 이미 진행중인데 트랜잭션을 추가로 수행한다면?
- cf, 기본 옵션인 `REQUIRED` 기준 설명
- 외부 트랜잭션이 수행중인데, 내부 트랜잭션이 추가로 수행됨
> 외부 트랜잭션과 내부 트랜잭션을 묶어서 하나의 트랜잭션, 내부 트랜잭션이 외부 트랜잭션에 참여
- 물리 트랜잭션, 논리 트랜잭션
> 물리 트랜잭션: 우리가 이해하는 실제 데이터베이스에 적용되는 트랜잭션, 실제 커넥션을 통해 트랜잭션 시작, 커밋, 롤백하는 단위  
논리 트랜잭션: 트랜잭션 매니저를 통해 트랜잭션을 사용하는 단위
- `원칙`
> 모든 논리 트랜잭션이 커밋되어야 물리 트랜잭션이 커밋된다.  
하나의 논리 트랜잭션이라도 롤백되면 물리 트랜잭션은 롤백된다.

## 4.스프링 트랜잭션 전파4 - 전파 예제
- inner_commit() - BasicTxTest 추가
```
TransactoinStatus outer = txManager.getTransaction(new DefaultTrasactionAttribute());
log.info("outer.isNewTransaction()={}, outer.isNewTransaction());

TransactoinStatus inner = txManager.getTransaction(new DefaultTrasactionAttribute());
log.info("inner.isNewTransaction()={}, inner.isNewTransaction());
txManager.commit(inner);

txManager.commit(outer);
```
- 외부 트랜잭션 new(true), 내부 트랜잭션 new(false)  
- 트랜잭션 참여
> 내부 트랜잭션이 외부 트랜잭션을 그대로 이어 받아서 따른다는 뜻  
외부 트랜잭션과 내부 트랜잭션이 하나의 물리 트랜잭션으로 묶이는것
- `status.isNewTransaction()`
> tx1: manual commit, tx2: Participation in existing transaction  
내부 트랜잭션을 시작하거나 커밋할 때는 DB 커넥션을 통해 커밋하는 로그 X  
처음 트랜잭션을 시작한 외부 트랜잭션이 실제 물리 트랜잭션을 관리
- 핵심 정리
> 트랜잭션 매니저에 커밋을 호출한다고해서 항상 실제 커넥션에 물리 커밋이 발생하지 않는다는 점  
신규 트랜잭션인 경우에만 실제 커넥션을 사용해서 물리 커밋과 롤백을 수행한다.

## 5.스프링 트랜잭션 전파5 - 외부 롤백
- outer_rollback - BasicTxTest 추가
> 전체 내용 모두 롤백

## 6.스프링 트랜잭션 전파6- 내부 롤백
내부 트랜잭션이 롤백했지만, 내부 트랜잭션은 물리 트랜잭션에 영향을 주지 않는다, 이 문제 어떻게 해결?
- inner_rollback() - BasicTxTest 추가
```
assertThatThrowBy(()-> txManager.commit())
	.isInstanceOf(UnexpectedRollbackException.class);
```
> `UnexpectedRollbackException.class` 발생  
tx2: `Participating transaction failed - marking existing transaction as rollback-only`  
기존 트랜잭션을 롤백전용으로 표시  
tx1: `Global transaction is marked as rollback-only but transaction code requested commit`  
외부 트랜잭션 커밋, 전체 트랜잭션이 롤백 전용으로 표시, 따라서 물리 트랜잭션을 롤백한다.
- `정리`
> 논리 트랜잭션이 하나라도 롤백되면 물리 트랜잭션은 롤백된다.  
내부 논리 트랜잭션이 롤백되면 롤백 전용 마크를 표시한다.  
외부 트랜잭션을 커밋할 때 롤백 전용 마크를 확인한다. 롤백 전용 마크가 표시되어 있으면 물리 트랜잭션을 롤백하고, UnexpectedRollbackException 예외를 던진다

## 7.스프링 트랜잭션 전파7 - REQUIRES_NEW
외부 트랜잭션과 내부 트랜잭션을 분리해서 사용하는 방법
- REQUIRES_NEW
> 물리 트랜잭션을 분리하려면 내부 트랜잭션을 시작할 때 REQUIRES_NEW 옵션 사용
- inner_rollback_requires_new() - BasicTxTest 추가
```
DefaultTransactionAttribute definition = new DefaultTransactionAttribute();
definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
TransactionStatus inner = txManager.getTransaction(definition);
```
> 내부 트랜잭션을 시작할 때 전파 옵션인 propagationBehavior에 PROPAGATION_REQUIRES_NEW 옵션을 주었다, 새로운 물리 트랜잭션을 만들어서 사용
- 정리
> REQUIRES_NEW 옵션을 사용하면 물리 트랜잭션이 명확하게 분리된다.  
REQUIRES_NEW를 사용하면 데이터베이스 커넥션이 동시에 2개 사용된다는 점을 주의해야 한다.

## 8.스프링 트랜잭션 전파8 - 다양한 전파 옵션
REQUIRED 기본 사용, REQUIRES_NEW 가끔 사용, 나머지는 거의 사용하지 않는다
- `REQUIRED`
> (기존 트랜잭션 없음)생성, (있음) 참여
- `REQUIRES_NEW`
> 생성, 생성
- SUPPORT
> X, 참여
- NOT_SUPPORT
> X, X(기존 트랜잭션 보류)
- MANDATORY
> IllegalTransactionStateException, 참여
- NEVER
> X, IllegalTransactionStateException
- NESTED
> 생성, 중첩
>> 외부 트랜잭션의 영향을 받지만, 중첩 트랜잭션은 외부에 영향을 주지 않는다
- 트랜잭션 전파와 옵션
> `isolation, timeout, readOnly는 트랜잭션이 처음 시작될 때만 적용된다.`