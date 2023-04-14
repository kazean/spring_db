# [이론] 6. 스프링과 문제 해결 - 예외 처리, 반복
## 1. 체크 예외와 인터페이스
서비스 계층은 가급적 특정 구현 기술에 의존하지 않고, 순수하게 유지하는 것이 좋다.
> SQLException에 대한 의존을 제거하려면 어떻게 해야할까?
- 인터페이스 도입
> MemberService > MemberRepository > JdbcMemberRepo, JpaMemberRepo
- !체크 예외와 인터페이스
> SQLEx 체크 예외, 체크 예외를 사용하려면 인터페이스에도 해당 체크 예외가 선언 되어 있어야 한다.
- !특정 기술에 종속되는 인터페이스
- 런타임 예외와 인터페이스

## 2. 런타임 예외 적용
- <I> MemberRepository
- MyDbException extends RuntimeException
- MemberRepositoryV4_1
> [예외변환] 핵심은 SQLException을 MyDbException이라는 런타임 예외로 변환해서 던지는 부분이다.
- 주의!: 예외를 변환할 때 기존 예외를 꼭! 포함하자.
- MemberServiceV4
> MemberRepository<I> 의존, throws SQLException 제거

- 정리
> 체크 예외를 런타임 예외로 변환하면서 인터페이스와 서비스 계층의 순수성 유지  
덕분에 향후 JDBC에서 다른 구현 기술로 변경하더라도 서비스 계층 코드 변경 X
- 남은 문제
> !리포지토리에서 넘어오는 특정 예외의 경우 복구 시도, 특정 상황에는 예외를 잡아서 복구하고 싶으면?

## 3. 데이터 접근 예외 직접 만들기
회원 가입시 DB에 이미 같은 ID가 있으면 뒤에 숫자를 붙여 복구시도
- `SQLException에는 errorCode라는 것이 들어있다.`
> 같은 오류여도 각각의 데이터베이스 마다 정의된 오류 코드가 다르다.  
서비스 계층에서는 예외 복구를 위해 키 중복 오류를 확인할 수 있어야 한다.  
리포지토리는 SQLEx 예외를 던지고 서비스에서 에러코드를 확인하여 복구 시도?
>> ! JDBC기술 의존, 서비스 계층의 순수성이 무너진다.  
SQLEx -> MyDuplicateKeyException
- MyDuplicateKeyException extend MyDbException
> 이 예외는 우리가 직접 만든 것이기 때문에 특정 기술에 의존적이지 않다.
- ExTranslatorV1Test
> 리포지토리 계층이 예외를 변환해준 덕분에 서비스 계층은 특정 기술에 의존하지 않는 MyDuplicateKeyException을 사용해서 문제를 복구하고, 서비스 계층의 순수성도 유지

- 남은 문제
> SQL ErrorCode는 각각의 데이터베이스 마다 다르다, 수백가지 오류 코드가 있다

## 4. 스프링 예외 추상화 이해
데이터 접근과 관련된 예외를 추상화 해서 제공한다
- 스프링 데이터 접근 예외 계층
> `DataAccessException`, NonTransientDataAccessException, TransientDataAccessException  
스프링은 데이터 접근 계층에 대한 수집 가지 예외를 정리해서 일관된 예외 계층을 제공한다.  
특정 기술에 종속적이지 않게 설계  
JDBC나 JPA를 사용할 때 발생하는 예외를 스프링이 제공하는 예외로 변환해주는 역활도 스프링이 제공  
예외의 최고 상위는 `org.springframework.dao.DataAccessException`  
DataAccessException은 크게 2가지 구분,  
NonTransient: 일시적이지 않은 오류 (SQL 문법 오류, 데이터베이스 제약조건 위배)  
Transient: 일시적 오류

- 스프링이 제공하는 예외 변환기
- SpringExceptionTranslatorTest - 추가 exceptionTranslator
> SQLExceptionTranslator exTranslator = new SQLErrorCodeSQLExceptionTranslator(dataSource);  
DataAccessException resultEx = exTranslator.translate("select", sql, e);  
>> exTranslator.translate("select", sql, e): 파라미터 설명, sql, SQLException  
예제에서는 SQL 문법이 잘못되었으므로 BadGrammerException을 반환  
각각의 DB마다 SQL ErrorCode는 다르다
> `org.springframework.jdbc.support.sql-error-codes.xml` 10개 이상의 RDBMS 벤더

- 정리
> 스프링은 데이터 접근 계층에 대한 일관된 예외 추상화를 제공  
스프링은 예외 변환기를 통해서 SQLException의 ErrorCode에 맞는 적절한 스프링 데이터 접근 예외로 변환해준다.  
만약 서비스, 컨트롤러 계층에서 예외 처리가 필요하면, 스프링이 제공하는 데이터 접근 예외를 사용하면 된다  
스프링 예외 추상화 덕분에 특정 기술에 종속적이지 않게 되었다.  
!스프링에 대한 기술 종속성은 발생한다.

## 5. 스프링 예외 추상화 적용
- MemberRepositortV4_2
> SQLExceptionTranslator exTranslator;
- 정리
> 예외에 대한 부분을 깔끔하게 정리, DI 활용, 런타임 예외로 복구

## 6. JDBC 반복 문제 해결 - JdbcTemplate
!이번에는 리포지토리에서 JDBC를 사용하기 때문에 발생하는 반복 문제를 해결해보자.
- JDBC 반복 문제
> 커넥션 조회, 커넥션 동기화  
PrepareStatement 생성 및 파라미터 바인딩  
쿼리 실행, 결과 바인딩, 리소스 종료  
예외 발생시 스프링 예외 변환기 실행
>> 템플릿 콜백 패턴, `JdbcTemplate`
- MemberRepositoryV5
> JdbcTemplate template;  
template.update(sql, param1, ...)  
template.queryForObject(sql, rowMapper(), param1, ...)
> JdbcTempalte은 JDBC 반복 문제 해결, 트랜잭션을 위한 커넥션 동기화, 스프링 예외 변환기 자동 실행

## 7. 정리
- 서비스 계층의 순수성
> 트랜잭션 추상화 + 트랜잭션 AOP  
스프링이 제공하는 예외 추상화와 예외 변환기  
서피스 계층이 리포지토리 인터페이스에 의존
- 리포지토리에서 JDBC를 사용하면서 반복 코드가 JdbcTemplate으로 대부분 제거