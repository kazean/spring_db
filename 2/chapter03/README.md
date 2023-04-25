# [이론] 3. 데이터 접근 기술 - 테스트 
## 1. 테스트 - 데이터베이스 연동
- test - application.yml
> datasource, logging.level.org.springframework.jdbc= debug
- `@SpringBootTest`
> @SpringBootApplication 를 찾아서 설정으로 사용한다.
> JdbcTempalte을 통해 실제 데이터베이스를 호출하게 된다.

- 테스트 실행
> findItems() > 실패
- 실패 원인
> 과거에 서버를 실행하면서 저장했던 데이터가 보관되어 있기 때문이다.

## 2. 테스트 - 데이터베이스 분리
테스트를 다른 환경과 철저하게 분리해야 한다.
> 가장 간단한 방법은 테스트 전용 데이터베이스 별도 운영  
jdbc:h2:tcp://localhost/~/testcase, 테이블 생성
- 접속 정보 변경 test - application.yml
- 테스트 실행
> 처음만 성공

- 테스트에서 매우 중요한 원칙
> 1. 테스트는 다른 테스트와 격리해야 한다.  
2. 테스트는 반복해서 실행할 수 있어야 한다.

## 3. 테스트 - 데이터 롤백
- 트랜잭션과 롤백 전략
- @BeforeEach, @AfterEach
- ItemRepositoryTest
> 트랜잭션 관리자는 PlatformTransactionManager 를 주입받아서 사용하면 된다.  
@BeforeEach > transactionManager.getTransaction(new DefaultTransactionDefinition()), 트린잭션을 시작한다  
@AfterEach > transactionManager.rollback(stats), 트랜잭션을 롤백 한다

## 4. 테스트 - @Transactionnal
스프링을 테스트 데이터를 초기화를 위해 트랜잭션을 적용하고 롤백하는 방식을 `@Transactional` 애노테이션 하나로 깔끔하게 해결해준다.
- ItemRepositoryTest
> 기존 트랜잭셔널 관련 코드 주석, @Transactional (class level)  
>> 테스트
- `@Transactional 원리`
> 로직이 성공하면 커밋, 테스트에서 아주 특별하게 동작  
@Transactional이 테스트에 있으면 스프링은 테스트를 트랜잭션 안에서 실행하고, 테스트가 끝나면 트랜잭션을 자동으로 롤백시켜 버린다.  
트랜잭션은 기본적으로 전파
- cf
> 테스트 케이스의 메서드나 클래스에 @Transactional을 직접 붙여서 사용할 때 만 이렇게 동작한다.  
그리고 트랜잭션을 테스트에서 시작하기 때문에 서비스, 리포지토리에 있는 @Transactional도 테스트에서 시작한 트랜잭션에 참여한다.

- 정리
> 테스트가 끝난 후 개발자가 직접 데이터를 삭제하지 않아도 되는 편리함을 제공한다.  
테스트 실행 도중 강제로 종료되어도 걱정이 없다(자동 롤백)  
트랜잭션 범위 안에서 테스트 진행하기 때문에 다른 테스트가 서로 영향을 주지 않는다.  
@Transactional 덕분에 아주 편리하게 다음 원칙을 지킬수 있게 되었다.
>> 테스트는 다른 테스트와 격리해야 한다.  
테스트는 반복해서 실행할 수 있어야 한다.
- 강제로 커밋하기 - `@Commit`
> @Rollback(value = false)

## 5. 테스트 - 임베디드 모드 DB
- 임베디드 모드
>  H2 데이터베이스는 자바로 개발, JVM 안에서 메모리 모드로 동작하는 특별한 기능 제공
- ItemServiceApplicaiton - 추가
> @Profile("test")  
dataSource()
>> jdbc:h2:mem:db, DB_CLOSE_DELAY=-1: 임베디드 모드에서는 데이터베이스 커넥션 연결이 모두 끊어지면 데이터베이스 종료하는데 그것을 방지하는 설정
- 실행
> !Table "ITEM" not found  
테스트를 실행하기 전에 테이블을 먼저 생성해주어야 한다. 스프링부트는 이 문제를 해결할 아주 편리한 기능 제공

- `스프링 부트 - 기본 SQL 스크립트를 사용해서 데이터베이스를 초기화하는 기능`
> `src/test/resources/schema.sql`
- cf, SQL 스크립트를 사용해서 데이터베이스를 초기화 하는 자세한 방법
> https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.datainitialization.using-basic-sql-scripts
- 실행

## 6. 테스트 - 스프링 부트와 임베디드 모드
임베디드 데이터베이스에 대한 설정도 기본으로 제공한다.
- ItemServiceApplication: DataSource 주석
- `test - application.yml: DataSource 주석`
- 실행
> `jdbc:h2:mem: 뒤에 임이의 데이터베이스 이름`  
spring.datasource.generate-unique-name=false > jdbc:h2:mem:testdb