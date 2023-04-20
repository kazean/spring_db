# [실습] 7.데이터 접근 기술 - Querydsl
## 1.Querydsl 소개1 - 기존 방식의 문제점
## 2.Querydsl 소개2 - 해결
## 3.Querydsl 설정
- build.gradle
```
//Querydsl 추가
//	== 스프링 부트 3.0 이상 ==
	implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
	annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
//	== 스프링 부트 3.0 미만 ==
//	implementation 'com.querydsl:querydsl-jpa'
//	annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"

annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
	delete file('src/main/generated')
}

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
//clean {
//	delete file('src/main/generated')
//}

// === ⭐ QueryDsl 빌드 옵션 (선택) ===
def querydslDir = "$buildDir/generated/querydsl"

sourceSets {
	main.java.srcDirs += [ querydslDir ]
}

tasks.withType(JavaCompile) {
//	options.annotationProcessorGeneratedSourcesDirectory = file(querydslDir)
	options.generatedSourceOutputDirectory = file(querydslDir)
}

clean.doLast {
	file(querydslDir).deleteDir()
}
```
> querydsl-jpa, querydsl-apt, jakarta.annotation-api, jakarta.persistence-api
- 검증 - Q타입 생성 확인 방법
1. Gradle
- Gradle > Tasks > build > clean
- Gradle > Tasks > other > complieJava
> build > generated > sources > annotationProcessor > java/main 하위에 QItem이 생성되어야 한다
2. Intellij IDEA
- Build > Build Project 또는 build > Rebuild
- cf, Q타입은 Git에서 제외
- Q타입 삭제
> Intellij IDEA 옵션을 선택하려면 src/main/generated에 파일이 생성되고, 필요한 경우 Q파일을 직접 삭제해야 한다.  
gradle에 해당 스크립트 추가하면 gradle clean 명령어를 실행할 때 /src/main/generated의 파일도 함께 삭제해준다.
```
//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
	delete file('src/main/generated')
}
```

## 4.Querydsl 적용
- JpaItemRepositoryV3 
```
@Repository
@Transactional
private fianl EntityManager em;
private final JpaQueryFactory query;

JpaItemRepositoryV3(EntityManager em) {
	this.em = em;
	this.query = new JPAQueryFactory(em);
}

public List<Item> findAllOld(ItemSearchCond itemSearch) {
	QItem item = QItem.item;
	BooleanBuilder builder = new BooleanBuilder();
	if(StringUtils.hasText(itemName)) {
		build.and(item.itemName.like("%" + itemName + "%"));
	}
	...

	return query.select(item)
		.from(item)
		.where(builder)
		.fetch();
}

public List<Item> findAll(ItemSearchCond itemSearch) {
	...
	return query.select(item)
		.from(item)
		.where(likeItemName(itemName), maxPrice(maxPrice))
		.fetch();
}

private BooleanExpression likeItemName(String itemName) {
	if(StringUtils.hasText(itemName)) {
		return item.itemName.like("%" + itemName + "%");
	}
	return null;
}

...
```
> Querydsl을 사용하려면 JPAQueryFactory가 필요하다. JPAQueryFactory는 JPA 쿼리는 JPQL을 만들기 때문에 EntityManager가 필요하다.
- save(), update(), findById()
- findAllOld(): Booleanbuilder 사용
- findAll(): where(a, b) 다양한 조건, AND 조건으로 처리 된다. 참고로 where()에 null을 입력하면 해당 조건은 무시된다.
- QuerydslConfig, ItemServiceApplication
- 예외 변환
> Querydsl은 별도의 스프링 예외 추상화를 지원하지 안흔ㄴ다, @Repository에서 스프링 예외 추상화

## 5.정리
- Querydsl 장점
> 동적 쿼리, 컴파일 시점 오류, 메서드 추출을 통해서 코드를 재사용할 수 있다.