# [실습] 4. 스프링과 문제 해결 - 트랜잭션
## 1. 문제점들
- 애플리케이션 구조
> 가장 단순하면서 많이 사용하는 방법은 역할에 따라 3가지 계층으로 나누는 것
- 프레젠테이션 계층: UI, 웹 요청과 응답, 사용자 요청 검증
- 서비스 계층: 비지니스 로직을 담당
- 데이터 접근 계층: 실제 데이터베이스에 접근하는 코드
- 순수한 서비스 계층
> 프레젠테이션, 데이터 접근 계층은 변하지만, `비지니스 로직은최대한 변경없이 유지`  
특정 기술에 종속적이지 않게 개발
>> 서비스 계층이 특정 기술에 종속되지 않기 때문에 비지니스 로직으 유지보수하기 쉽고, 테스트하기 편하다

- 문제점들
1. hello.jdbc.service.MemberService1
> 특정 기술 종속적이지 않고, 순수 비지니스 로직  
! SQLException이라는 JDBC 기술에 의존한다는점
2. hello.jdbc.service.MemberService2
> dataSource.getConnection(), con.setAutoCommit(false), ...
>> 트랜잭션을 비지니스 로직이 있는 서비스 계층에서 시작하는 것이 좋다.  
! 트랜잭션을 사용하기 위해서 DataSource, Connection, SQLException과 같은 JDBC 기술에 의존  
! JPA로 사용하게 되면 서비스 코드 모두 변경해야한다, 유지보수가 어렵다.

- 문제점 정리
> 1)트랜잭션 문제  
2)예외 누수 문제  
3)JDBC 반복 문제
- JDBC 구현 기술이 서비스 계층에 누수되는 문제: 특정 기술 종속
- 트랜잭션 동기화 문제: 커넥션 파라미터
- 트랜잭션 적용 반복 문제: try, catch, finally
- 예외 누수: SQLException
- JDBC 반복 문제: MemberRepository 유사 코드 반복

## 2. 트랜잭션 추상화
현재 서비스 계층은 트랜잭션을 사용하기 위해 JDBC 기술 종속
- 구현 기술에 따른 트랜잭션 사용법
> JDBC: con.setAutoCommit(false)  
JPA: transaction.begin()

- `트랜잭션 추상화`
> 이 문제를 해결하려면 트랜잭션 기능을 추상화하면 된다, 다음과 같은 인터페이스를 만들어서 사용하면 된다.
```
public interface TxManager {
	begin();
	commit();
	rollback();
}
```
> 구현체: JdbcTxManager, JpaTxManager

- `스프링의 트랜잭션 추상화, 트랜잭션 매니저`
> PlatformTransactionManager > DataSourceTransactionManager, JpaTransactionManager, HibernateTransactionManager, EtcTxManager..
- cf, 스프링 5.3부터는 DataSourceTransactionManager를 상속받아 기능을 확장한 JdbcTransactionManager를 제공한다.
- PlatformTransactionManager <I>
```
package org.springframework.transaction;
public interface PlatformTransactionManager extends TransactionManager {
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
	throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}
```
> getTransaction(): 트랜잭션을 시작한다.

## 3. 트랜잭션 동기화
트랜잭션 매니저는 크게 두가지 역할을 한다.
- 트랜잭션 추상화
- 리소스 동기화
> 트랜잭션을 유지하려면 데이터베이스 같은 커넥션 유지
- 트랜잭션 매니저와 트랜잭션 동기화 매니저
> 스프링은 `트랜잭션 동기화 매니저`를 제공, 쓰레드 로컬 사용
>> 멀티 쓰레드 상황에 안전하게 커넥션 동기화 할 수 있다. 이전처럼 파라미터를 전달하지 않아도 된다.
- 트랜잭션 동기화 매니저
> org.springframework.transaction.support.TransactionSynchronizationManager
>> cf, 쓰레드 로컬은 각각의 쓰레드 마다 별도의 저장소 부여

## 4. 트랜잭션 문제 해결 - 트랜잭션 매니저1
- hello.jdbc.repository.MemberRepositoryV3
```
close() {
	DataSourceUtils.releaseConnection(con, dataSource);
}

getConnection() {
	Connection con = DataSourceUtils.getConnection(dataSource);
}
```
> DataSourceUtils.getConnection()
>> 트랜잭션 동기화 매니저가 관리하는 커넥션이 있으면 해당 커넥션 반환
> DataSourceUtils.releaseConnection()
>> 트랜잭션 사용하기 위해 동기화된 커넥션은 커넥션을 닫지 않고 그대로 유지해준다.
- hello.jdbc.servie.MemberServiceV3_1
```
private final PlatformTransactionManager transactionManager;

accountTransfer() {
	TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

	try {
		bizLogic();
		transactionManager.commit(stats);
	} catch (Exception e) {
		transactionManager.rollback(status);
		throw new IllegalStateException(e);
	}
}
```
> PlatformTransactionManager: 트랜잭션 매니저를 주입 받는다.  
transaction.getTransaction(): 트랜잭션을 시작한다, TransactionStatus를 반환한다. 현재 트랜잭션의 상태정보가 포함되어 있다.  
new DefaultTranscationDefinition: 트랜잭션과 관련된 옵션을 지정할 수 있다.  
transactionManager.commit()/rollback()
- MemberServiceV3_1Test
> PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);

## 5. 트랜잭션 문제 해결 - 트랜잭션 매니저2
트랜잭션 매니저의 전체 동작 흐름[그림]
1. 서비스 계층에서 transactionManager.getTransaction() 트랜잭션 시작
2. 트랜잭션 시작하려면 데이터베이스 커넥션 필요, 내부에서 데이터소스를 사용해 커넥션 생성
3. 커넥션을 수동 커밋 모드로 변경
4. 커넥션을 트랜잭션 동기화 매니저에 보관
5. 트랜잭션 동기화 매니저는 쓰레드 로컬에 커넥션을 보관
6. 서비스는 비지니스 로직 실행시 커넥션 파라미터 전달 X
7. 리포지토리 메서드는 커넥션 얻기 위해 DataSourceUtils.getConnection()을 이용해 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내 사용
8. 획득한 커넥션을 사용해 SQL을 DB에 전달
9. 비지니스 로직이 끝나고 커밋, 롤백으로 트랜잭션을 종료
10. 트랜잭션 종료시 동기화 매니저로 커넥션 획득
11. 커밋, 롤백
12. 전체 리소스 정리
- 정리
> 트랜잭션 추상화 덕분에 서비스 코드는 이제 JDBC 기술에 의존하지 않는다.  
!SQLException이 아직 남아있다

## 6. 트랜잭션 문제 해결 - 트랜잭션 템플릿
! 트랜잭션을 사용하는 로직의 패턴이 반복, try, catch, finally, 서비스에서 반복
> 템플릿 콜백 패턴을 활용하면 이런 반복 문제 해결

- cf 템플릿 콜백 패턴, TransactionTemplate 편리한 기능 제공
- `트랜잭션 템플릿, TransactionTemplate`
```
public class TransactionTemplate {
	private PlatformTransactionManager transactionManager;

	public <T> T execute(TransactionCallback<T> action){..}
	void executeWithoutResult(Consumer<TransactionStatus> action){..}
}
```
- hello.jdbc.service.MemberServiceV3_2
```
private fianl Transactiontemplate txTemplate

public MemberServiceV3_2(PlatformTransactionManager transactionManager) {
	txTemplate = new TransactionTemplate(transactionManager)
}

accuntTransfer() {
	txTempalte.executeWithoutResult((status) -> {
		try {
			bizLogic()
		} catch (SQLException e) {
			throw new IllegalStateException(e);
		}
	})
}
```
> TransactionTemplate을 사용하려면 TransactionManager가 필요하다, 생성자 이용  
트랜잭션 템플릿 덕분에 트랜잭션을 시작하고, 커밋, 롤백 부분 제거  
트랜잭션 기본 동작
>> 비지니스 로직이 정상 수행되면 커밋, `언체크 예외가 발생하면 롤백`(체크 예외의 경우 커밋)
> 코드에서 예외처리를 위해 try~catch, bizLogic()시 SQLException 체크 예외를 넘겨준다. 람다에서 체크 예외를 밖으로 던질 수 없기 때문에 언체크 예외로 바꾸어 던지도록 예외 전환.
- MemberServiceV3_2Test

- 정리
> 트랜잭션 템플릿 덕분에 트랜잭션을 사용할 때 반복하는 코드를 제거  
!하지만 서비스 로직인데 비지니스 로직 뿐만 아니라 트랜잭션을 처리하는 기술 로직이 함께 포함되어 있다.  
애플리케이션 구성하는 로직을 핵심 기능과 부가 기능으로 구분, 트랜잭션은 부가 기능이다.  
! 두 관심사를 하나의 클래스에서 처리, 유지보수가 어려워 진다.

## 7.트랜잭션 문제 해결 - 트랜잭션 AOP 이해
트랜잭션 추상화, 트랜잭션 템플릿
> ! 서비스 계층에 순수한 비지니스 로직을 남긴다는 목표 달성 X
>> 스프링 AOP를 통해 프록시를 도입하면 문제를 깔끔하게 해결

- cf, `@Transactional`을 사용하면 스프링이 AOP를 사용해서 트랜잭션을 편리하게 처리해준다.
- 프록시를 통한 문제 해결
> 프록시를 사용하면 트랜잭션을 처리하는 객체와 비지니스 로직을 처리하는 서비스 객체를 명확히 분리 할 수 있다.

- 스프링이 제공하는 트랜잭션 AOP
> @Aspect, @Advice, @Pointcut, @Transactional
- `@Transactional`
> org.springframework.transaction.annotation.Transactional
- cf, 스프링은 트랜잭션 AOP 처리를 위해 다음 클래스 제공, 스프링 부트를 사용하면 해당 빈들은 스프링 컨테이너에 자동으로 등록된다.
> 어드바이저: BeanFactoryTransactionAttributeSourceAdvisor  
포인트컷: TransactionAttributeSourcePointcut  
어드바이스: TransactionInterceptor

## 8. 트랜잭션 문제 해결 - 트랜잭션 AOP 적용
- MemberServiceV3_3
> @Transactional, 순수한 비지니스 로직만 남기고, 트랜잭션 관련 코드는 모두 제거  
@Transactional 애노테이션은 메서드에 붙여도 되고, 클래스에 붙여도 된다. 클레스에 붙이면 외부에서 호출 가능한 public 메서드가 AOP 적용 대상이 된다.
- MemberServiceV3_3Test
> @SpringbootTest, @TestConfiguration, AopCheck(): AopUtils.isAopProxy(object)

## 9. 트랜잭션 문제 해결 - 트랜잭션 AOP 정리
- 트랜잭션 AOP 적용 전체 흐름[그림]
- 선언적 트랜잭션 관리 vs 프로그래밍 방식 트랜잭션 관리
> 선언적 트랜잭션 관리: @Transactional, 간편하고 실용적 실무에서 많이 사용  
프로그래밍 방식 트랜잭션 관리: 스프링 컨테이너나 스프링 AOP 기술 없이 간단히 사용할 수 있지만, 실무에서 대부분 스프링 컨테이너와 스프링 AOP를 사용하기 때문에 거의 사용되지 않는다.
- 정리
> 선언적 트랜잭션 관리 덕분에 드디어 트랜잭션 관련 코드를 순수한 비지니스 로직에서 제거할 수 있었다.  
개발자는 @Transactional

## 10. 스프링 부트의 자동 리소스 등록
데이터소스나 트랜잭션 매니저를 직접 등록하지 않는다
- 데이터소스 - 자동득록
> 스프링 부트틑 데이터소스를 스프링 빈에 자동 등록한다  
application.properties에 있는 속성을 사용해서 DataSource 생성  
스프링 부트가 기본으로 생성하는 데이터소스는 커넥션풀을 제공하는 HikariDataSource이다.
- 트랜잭션 매니저 - 자동 등록
> 스프링 부트는 적절한 트랜잭션 매니저(PlatformTransactionManager)를 자동으로 스프링빈에 등록, 어떤 트랜잭션 매니저를 생성하는지는 현재 등록된 라이브러리를 보고 판단
- 데이터 소스와 트랜잭션 매니저 자동 등록
> application.properteis
> MemberServiceV3_4Test
>> 데이터소스와 트랜잭션 매니저를 스프링 빈으로 등록하는 코드가 생략
- 정리
> 데이터소스와 트랜잭션 매니저는 스프링 부트가 제공하는 자동 빈 등록 기능을 사용하는 것이 편리하다.  
추가로 application.properies를 통해 설정도 편리하게 할 수 있다.