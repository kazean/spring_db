# [이론] 4. 데이터 접근 기술 - Mybatis
## 1.Mybatis 소개
JdbcTemplate 보다 더 많은 기능을 제공하는 SQL Mapper 이다.  
SQL을 XML에 편리하게 작성할 수 있고 동적 쿼리를 매우 편리하게 작성할 수 있다.
- SQL 여러줄
- 동적 쿼리
- 설정의 장단점
> Mybatis는 약간의 설정의 필요
- 정리
> 동적 쿼리와 복잡한 쿼리가 많다면 Mybatis  
하지만 Mybatis를 선택했다면 그것으로 충분할 것이다.

## 2.Mybatis 설정
- `mybatis-spring-boot-starter`
- build.gradle
> implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
>> 버전정보, 스프링 부트가 버전을 관리해주는 공식 라이브러리가 아니다!  
mybatis-spring-boot-starter: Mybatis를 스프링 부트에서 사용할 수 있게 시작하는 라이브러리  
mybatis-spring-boot-autoconfigure: Mybatis와 스프링 부트 설정 라이브러리  
mybatis-spring: Mybatis와 Spring 연동
mybatis: Mybatis 라이브러리
- 설정 - application.properties (main, test)
> mybatis.type-aliases-package: 마이바티스 타입정보 사용할 때는 패키지 이름 적어줘야 하는데, 여기에 명시하면 패키지 이름 생략 가능 ex, resultType  
mybatis.map-underscore-to-camel-case: BeanPropertyRowMapper 처럼 언더바를 카멜로 자동 변경
- 관례의 불일치

## 3.Mybatis 적용1 - 기본
- ItemMapper `@Mapper`
> 마이바티스 매핑 XML을 호출 해주는 매퍼 인터페이스  
@Mapper 애노테이션을 붙여 주어야한다. 구현체는 추후 설명  
이제 같은 위치에 실행할 SQL이 있는 XML 매핑 파일을 만들어 주면 된다
- `src/main/resources/hello/itemservice/repository/mybatis/ItemMapper.xml`
```
<? xml version="1.0" encoding="UTF-8">
<!DOCTYPE mapper ~>
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper>
	<insert id="save" useGeneratedKeys="true" keyProperty="id">
		insert into item(item_name, price, quantity)
		values (#{itemName}, #{price}, #{quantity})
	</insert>
```
> namespace: 앞서 만든 매퍼 인터페이스 지정
- cf - XML 파일 경로 수정하기 / application.properties
> mybatis.mapper-locations=classpath:mapper/**/*.xml

- insert - save
> <insert>, id="메서드 이름", 파라미터 #{} 문법  
useGeneratedKeys 데이터베이스가 키를 생성해 주는 IDENTITY 전략일 때 사용, keyProperty 생성되는 키 속성의 이름 지정 > insert가 끝나면 Item 객체의 id속성에 생성된 값이 입력

- update
```
void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);

<update id="update">
	update item
	set item_name=#{updateParam.itemName}, price=#{updateParam.price}, quantity=#{updateParam.quantity}
	where id = #{id}
</update>
```
> <update>, 파라미터가 2개 이상이면 @Param으로 이름을 지정해서 파라미터 구분해야 한다.

- select - findAll
```
List<Item> findAll(ItemSearchCond itemSearch);

<select id="findAll" resultType="Item">
	select id, item_name, price, quantity
	from item
	<where>
		<if test="itemName != null and itemName != ''">
			and item_name like concat('%',#{itemName},'%')
		</if>
		<if test="maxPrice != null">
			and price &lt;= #{maxPrice}
		</if>
	</where>
</select>
```
> <where>, <if> 같은 동적 쿼리 문법, <if> 가 하나라도 성공하면 처음 나타나는 and를 where로 변환해준다.
- XML 특수 문자
```
< : &lt;
> : &gt;
& : &amp;
```
> xml 영역에 <, > 같은 특수 문자를 사용할 수 없다
- XML CDATA 사용
```
<![CDATA[]]> 사용
```

## 4.Mybatis 적용2 - 설정과 실행
- MybatisRepository
> private ifanl ItemMapper itemMapper;
- MybatisConfig, ItemServiceApplication - 변경
- 테스트 

## 5.Mybatis 적용3 - 분석
ItemMapper 매퍼 인터페이스의 구현체가 없는데 어떻게 동작?
> Mybatis 스프링 연동 모듈에서 자동으로 처리
- 설정 원리
> 1. 애플리케이션 로딩 시점에 Mybatis 연동 모듈은 @Mapper가 붙어있는 인터페이스 조사  
2. 동적 프록시 기술을 사용해서 ItemMapper 인터페이스의 구현체를 만든다  
3. 생성된 구현체를 스프링 빈으로 등록한다.
- MybatisItemRepository - 로그 추가
> log.info("itemMapper class={}, itemMapper.getClass());
>> com.sun.proxy.$Proxy66
- 매퍼 구현체
> ItemMapper의 구현체 덕분에 인터페이스 만으로 XML의 데이터를 찾아서 호출  
매퍼 구현체는 예외 변환까지 처리

- 정리
> 매퍼 구현체 덕분에 마이바티스를 스프링에 편리하게 통합해서 사용  
스프링 예외 추상화  
마이바티스 스프링 연동 모듈이 많은 부분을 자동으로 설정(커넥션, 트랜잭션)
- cf - MybatisAutoConfiguration

## 6.Mybatis 기능 정리1 - 동적 쿼리
- 동적 SQL: if, choose, trim, foreach
- if
```
<if test="title != null">
	and title like #{title}
</if>
```
> 내부 문법은 `OGNL 사용`
- choose, when, otherwise
```
<choose>
	<when test="~"> and ~ </when>
	<when test="~"> and ~ </when>
	<otherwise> and ~ </otherwise>
</choose>
```
- trim, where, set
> <where> 사용
> trim
```
<trim prefix="WHERE" prefixOverrides="AND |OR"> ~ </trim>
```
- foreach
```
<foreach item="item" index="index" collectio="list"
	open="ID in (" seperator="," close=")" nullable="true>
	#{item}
</foreach>
```

## 7.Mybatis 기 정리2 - 기타 기능
- 애노테이션으로 SQL 작성
```
@Select("select id, item_name, price, quantity from item where id=#{id})
Optional<Item> findById(Long id);
```
> `@Select, @Insert, @Update, @Delete` 이 경우 XML에는 제거, 동적 SQL이 해결되지 않으므로 간단한 경우에만 사용
- 문자열 대체(String Substitution)
> ${}
>> !주의 ${}를 사용하면 SQL 읹ㄱ션 공격을 당할 수 있다. 가급적 사용X
- 재사용 가능한 SQL 조각
```
<sql id="userColumns> ${alias}.id, ${alias}.username </sql>

<include refid="userColumns>
	<property name="alias" value="t1">
</include>
```
- Result Maps
```
<resultMap id="userResultMap" type="User">
	<id property="id" column="user_id">
	<result property="username" column="username>
</resultMap>

<select id="~" resultMap="userResultMap">
```
- 복잡한 결과 매핑
> <association>, <collection>
>> JPA는 객체와 관계형 데이터베이스를 ORM 개념으로 매핑하기 때문에 이런 부분이 자연스럽지만, Mybatis에서는 공수가 많고 선응 최적화하기 어렵다.