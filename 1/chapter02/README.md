# [이론] 2. 커넥션풀과 데이터소스 이해
## 1. 커넥션풀 이해
데이터베이스 커넥션을 획득할 때 다음과 같은 복잡한 과정 반복
> 1. 애플리케이션 로직은 DB 드라이버를 통해 커넥션을 조회한다.  
2. DB 드라이버는 DB와 TCP/IP 커넥션을 연결(3 way handshake)  
3. DB 드라이버는 TCP/IP 커넥션이 연결되면, ID, PW와 기타 부가정보를 DB에 전달한다.  
4. DB는 ID, PW를 통해 내부 인증을 완료하고, 내부에 DB 세션을 생성한다.  
5. DB는 커넥션 생성이 완려되었다는 응답을 보낸다.
6. DB 드라이버는 커넥션 객체를 생성해서 클라이언트에 반환한다.
> !커넥션을 새로 만드는 것은 과정도 복잡하고 시간도 많이 소모, SQL을 실행하는 시간 뿐만 아니라 커넥션을 새로 만드는 시간 추가
- 커넥션 풀
> 이런 문제를 한번에 해결하는 아이디어  
애플리케이션을 시작하는 시점에 커넥션 풀을 필요한 만큼 커넥션을 미리 확보해서 풀에 보관, 즉시 사용가능
- 정리
> 적적한 커넥션풀 숫자 > 성능테스트, 서버당 최대 커넥션 수 제한, 커넥션풀 이점 실무 항상 기본사용, 사용도 편리하고 성능도 뛰어난 오픈소스  
`commons-dbcp2, tomcat-jdbc pool, HikariCP`, 최근에는 hikariCP 주로 사용

## 2. DataSource 이해
커넥션을 얻는 방법은 DriverManager를 직접 사용하거나, 커넥션 풀을 사용하는 방법
- DriverManager를 통해서 커넥션 획득
- DriverManager를 커넥션 획득하다가 커넥션 풀로 변경시 문제
> 애플리케이션 코드 변경 필요
- `커넥션을 획득하는 방법을 추상화`
> `javax.sql.DataSource 인터페이스 제공`
```
public interface DataSource {
	Connection getConnection() throws SQLException;
}
```
- 정리
> 대부분의 커넥션 풀은 DataSource 인퍼페이스를 이미 구현  
커넥션풀 구현 기술을 변경하고 싶으면 해당 구현체로 갈아끼우기만 하면 된다,  
DriverManager는 DataSource 인터페이스 구현 X > DriverManagerDataSource: 스프링 구현 클레스 제공  
자바는 DataSource를 통해 커넥션을 획득하는 방법을 추상화

## 3. DataSource 예제1 - DriverManager
- test/hello.jdbc.ConnectionTest - 드라이버 매니저
> Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);  
Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
- test/hello.jdbc.ConnectionTest - 데이터소스 드라이버 매니저 추가
> DriverManagerDatasource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);  
Connection con1 = dataSource.getConnection();  
Connection con2 = dataSource.getConnection();
- 파라미터 차이, 설정과 사용의 분리

## 4. DataSource 예제2 - 커넥션풀
- test/hello.jdbc.ConnectionTest - 데이터소스 커넥션 풀 추가
> HikariDataSource dataSource = new HikariDataSource();  
Connection con1 = dataSource.getConnection();    
Connection con2 = dataSource.getConnection();  
>> HikariCP 커넥션 풀을 사용한다. HikariDataSource는 DataSource 인터페이스를 구현
- HikariConfig
- MyPool connection adder
> 별도의 쓰레드 사용해서 커넥션 풀에 커넥션을 채우고 있는 것을 확인
- 커넥션 풀에서 커넥션 획득

## 5. DataSource 적용
- MemberRepositoryV1
```
private final Datasource dataSource;

private Connection getConnection() {
	dataSource.getConnection();
}

private void close(Connection con, Statement stmt, ResultSet rs) {
	JdbcUtils.closeResultSet(rs);
	JdbcUtils.closeStatement(stmt);
	JdbcUtils.closeConnection(con);
}
```
- `DataSource 의존관계 주입`
- `JdbcUtils` 편의 메서드
- MemberRepositoryV1Test
```
MemberRepositoryV1 repository;

@BeforeEach
void beforeEach() throws Exception {
	//기본 DriverManager - 항상 새로운 커넥션 획득
	//DriverManagerDataSource dataSource =
	// new DriverManagerDataSource(URL, USERNAME, PASSWORD);
	//커넥션 풀링: HikariProxyConnection -> JdbcConnection
	HikariDataSource dataSource = new HikariDataSource();
	dataSource.setJdbcUrl(URL);
	dataSource.setUsername(USERNAME);
	dataSource.setPassword(PASSWORD);
	repository = new MemberRepositoryV1(dataSource);
}
```
> 커넥션 풀 사용시 conn0 커넥션이 재사용 된 것을 확인할 수 있다.
- DI
> DriverManagerDataSource -> HikariDataSource로 변경해도 Repository 코드 변경 X
>> DI + OCP