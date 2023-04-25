# [이론] 9.스프링 트랜잭션 이해
## 1.스프링 트랜잭션 소개
- `스프링 트랜잭션 추상화`
> JDBC 기술과 JPA 기술은 트랜잭션을 사용한는 코드 자체가 다르다.
```
//JDBC
Connection con = dataSource.getConnection();
con.setAutoCommit(false);
con.commit()/rollback();

//JPA
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin()/commit()/rollback()
```
>> 스프링은 이런 문제 해결을 위해 트랜잭션 추상화를 제공한다. `PlatformTransactionManager`  
추상화뿐만 아니라, 트랜잭션 매니저의 구현체도 제공: DataSourceTransactionManager, JpaTransactionManager  
어떤 접근 기술을 사용하는지 자동 인식해서 적적한 트랜잭션 매니저를 선택해서 스프링 빈으로 등록

- 스프링 트랜잭션 사용 방식
> 선언적 트랜잭션 관리 vs 프로그래밍 방식 트랜잭션 관리
- 선언적 트랜잭션과 AOP
> `@Transactional`
>> con.setAutoCommit(), 같은 트랜잭션 유지, 동기화
- 스프링이 제공하는 트랜잭션 AOP
> org.sprigframework.transaction.annotation.Transactional

## 2.프로젝트 생성
- Project: gradle, spring 2.6.x > 3.x, Group: hello, Articat:springtx  
Dependencies: Spring Data JPA, H2, Lombok

## 3.트랜잭션 적용 확인
- TxApplyBasicTest
- proxyCheck()
> AopUtils.isAopProxy(): basicService > basicService$$EnhancerBySpringCGLIB  
특정 클래스나 메서드에 하나라도 @Transactional이 하나라도 있으면 트랜잭션 AOP는 프록시를 만들어 스프링 컨테이너에 등록
- txTest()
> `트랜잭션 로그` application.yml: logging.level.org.springframework.transaction.interceptor=TRACE  
basicService.tx(): 프록시 tx() > 트랜잭션 적용 대상 > 트랜잭션 시작 후 basicService.tx() 실행  
basicService.nonTx(): 프록시 nonTx() > 적용 대상 X > basicService.nonTx()  
`TransactionSynchronizationManager.isActualTransactionActive()`: 현재 쓰레드에 트랜잭션이 적용되어 있는지 확인

## 4.트랜잭션 적용 위치
더 구체적이고 자세한 것이 높은 우선순위를 가진다.
- TxLevelTest
> 클래스 타입보다 메서드가 우선

- `@Transaction 두 가지 규칙`
1. 우선순위 규칙
2. 클래스에 적용하면 메서드는 자동 적용
- 1. 우선순위
> 클래스보단 메서드가 더 구체적
- 2. 클래스에 적용하면 메서드는 자동 적용
> cf, readOnly=false가 기본 옵션
- TransactionSynchronizationManager.isCurrentTransationReadOnly(): readOnly 값 반환

- 인터페이스에 @Transactional 적용
1. 클래스의 메서드
2. 클래스의 타입
3. 인터페이스의 메서드
4. 인터페이스의 타입
> !인터페이스에 @Transactional 사용하는 것은 스프링 공식 메뉴얼에서 권장하지 않는 방법 > 적용되지 않는 경우
- cf, @Transactional + 스프링 5.0
> 구체 클래스 기반으로 프록시를 생성하는 CGLIB 방식을 사용하면 인터페이스에 있는 @Transactional을 인식하지 못했다, 5.0 인식
>> 가급적 가이드대로 구체 클래스에 @Transactional

## 5.트랜잭션 AOP 주의 사항 - 프록시 내부 호출1
프록시 안에서 내부 메서드 호출이 발생하면 프록시를 거치지 않고 대상 객체를 직접 호출하는 문제
- InterlCallV1Test
> @Transactional 하나라도 있으면 프록시로 등록  
internalCall(): 트랜잭션  
externalCall(): 트랜잭션X > this.internal() 적용 X
- 문제 원인
> `this.internal()`
- `프록시 AOP 한계`
> 프록시를 사용하면 메서드 내부 호출에 프록시를 적용할 수 없다.
>> 내부 호출을 피하기 위한 가장 단순한 방법은 inernal메서드를 별도 클래스로 분리

## 6.트랜잭션 AOP 주의 사항 - 프록시 내부 호출2
- InternalCallV2Test
> CallService, InternalService
- public 메서드만 트랜잭션 적용
> protected, private, package-visible에는 트랜잭션이 적용되지 않는다.
>> 클래스에 트랜잭션 적용시 의도치 않은 메서드까지 적용되는 것을 방지하기 위해

## 7.트랜잭션 AOP 주의 사항 - 초기화 시점
`스프링 초기화 시점에는 트랜잭션 AOP가 적용되지 않을 수 있다`
- InitTxTest
> @PostConstruct와 @Transactional 트랜잭션 적용 X  
초기화 코드가 먼저 호출되고 ,그 다음에 트랜잭션 AOP가 적용  
> `ApplicationReadyEvent` 사용

## 8.트랜잭션 옵션 소개
- `@Transactional Opt`
```
public @interface Transactional {
	String value();
	String transactionManager();

	Class<? extends Throwable>[] rollbackFor();
	Class<? extends Throwable>[] noRollbackFor();

	Propagation propagation() default Propagation.REQUIRED;
	Isolation isolation() default Isolation.DEFAULT;;
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	boolean readOnly() default false;
	String[] label();
}
```
- value, transactionManager
> 어떤 트랜잭션 매니저 사용할지, 생략시 기본 등록된 트랜잭션 매니저  
@Transactional("memberTxManager")
- rollbackFor
> 예외 발생시 트랜잭션 기본정책  
언체크 예외 RuntimeEx, Error: rollback  
체크 예외 Exception: commit  
`@Transactional(rollbackFor = Exception.class)`: Ex > rollback
- noRolbackFor: rollbackFor 반대
- `propagation`: 트랜잭션 전파
- isolation
- timeout
- label: 트랜잭션 애노테이선에 있는 값을 직접 읽어서 어떤 동작을 하고 싶을 때 사용, 일반적 사용 X
- `readOnly`
> readOnly=true 읽기전용 트랜잭션, 다양한 성능 최적화
- 프레임워크
> JdbcTempalte 변경시 예외  
JPA 플러쉬 X, 스냅샷 객체 생성 X
- JDBC 드라이버
> DB와 드라이버 버전에 따라 동작방식이 다르다, 변경 시 예외 / 읽기, 쓰기(마스터, 슬레이브) 대이터베이스를 구분해서 요청
- 데이터베이스
> 읽기 전용 트랜잭션의 경우 읽기만 하면 되므로, 내부적 성능 최적화

## 9.예외와 트랜잭션 커밋, 롤백 - 기본
예외 발생시 스프링 트랜잭션 AOP는 예외 종류에 따라 트랜잭셔늘 커밋하거나 롤백
> 언체크 예외: RuntimeEx, Error > rollback  
체크 예외: Ex > commit
- RollbackTest
- application.yml
```
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=
DEBUG
#JPA log
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
logging.level.org.hibernate.resource.transaction=DEBUG
```
> runtimeException() - rollback  
checkedException() - commit  
rollbackFor() - rollback

## 10.예외와 트랜잭션 커밋, 롤백 - 활용
체크 예외는 비지니스 의미가 있을 때 사용하고, 런타임 예외는 복구 불가능한 예외로 가정한다.  
꼭 이런 정책을 따를 필요는 없다, rollbackFor 사용
- 비지니스 요구사항
> 비지니스 예외: 주문시 결제 잔고가 부족하면 주문 데이털를 저장하고, 결제 상태를 대기로 처리한다 NotEnoughMoneyException 체크 예외
- NotEnoughMoneyException
- Order
> @Setter를 남발 X
- OrderRepository <I> extends JpaRepository<Order, Long>
- OrderService
> orer.getUsername(): 예외, 잔고부족, 정상
- OrderServiceTest
```
@Test
complete(), runtimeException(), bizException()
```
- 정리
> NotEnoughEx은 시스템에 문제가 발생한 것이 아니라, 비지니스 문제 상황을 예외를 통해 알려준다.  
따라서 커밋하는 것이 맞고, 주문자체가 사라지면 문제가 된다.  
체크 예외의 경우도 롤백하고 싶으면 rollbackFor 사용