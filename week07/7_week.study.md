# Spring Batch Study 7

참고: [DEVOCEAN KIDO님 SpringBatch 연재 7](https://devocean.sk.com/blog/techBoardDetail.do?ID=166932)

---

Spring Batch의 `MyBatisPagingItemReader`와 `MyBatisItemWriter`로 DB 데이터를 읽고 쓰는 방법을 알아보자.

## 1. MyBatisPagingItemReader/MyBatisItemWriter 개요

### MyBatisPagingItemReader 개요

- Spring Batch에서 제공하는 `ItemReader` 인터페이스를 구현하는 클래스이다.
- 장점
    - **간편한 설정**: MyBatis **쿼리 매퍼**를 직접 활용하여 데이터를 읽을 수 있고 설정이 간편하다.
    - **쿼리 최적화**: MyBatis의 다양한 기능을 활용하여 최적화된 쿼리를 작성할 수 있다.
    - **동적 쿼리 지원**: 런타임 시 조건에 따라 **동적**으로 쿼리를 생성할 수 있다.
- 단점
    - **MyBatis 의존성**: MyBatis 라이브러리에 의존해야 한다.
    - **커스터마이징 복잡**: Chunk-oriented Processing 방식과 비교했을 때 커스터마이징이 더 복잡할 수 있다.

### MyBatisPagingItemReader 주요 구성 요소

- `SqlSessionFactory`
    - MyBatis **설정 정보** 및 SQL 쿼리 **매퍼 정보**를 담고 있는 객체이다.
    - 설정은 **@Bean, Spring Batch XML, Java** 코드로 직접 설정 가능하다.
- `QueryId`
    - 데이터를 읽을 MyBatis **쿼리 ID**이다.
    - MyBatisPagingItemReader setQueryId() 메소드를 통해 데이터를 읽을 MyBatis 쿼리 ID를 설정한다.
    - 쿼리 ID는 com.example.mapper.CustomerMapper.selectCustomers 와 같은 형식으로 지정된다.
- `PageSize`
    - 페이징 쿼리를 위한 **페이지 크기**를 지정한다.
- `SkippableItemReader`
    - 오류 발생 시 해당 Item을 건너뛸 수 있도록 한다.
- `ReadListener`
    - 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리할 수 있도록 한다.
- `SaveStateCallback`
    - 잡의 중단 시 현재 상태를 저장하여 재시작 시 이어서 처리할 수 있도록 한다.

### MyBatisItemWriter 개요

- Spring Batch에서 제공하는 `ItemWriter` 인터페이스를 구현하는 클래스이다.
- 데이터를 MyBatis를 통해 데이터베이스에 저장하는 데 사용된다.
- 장점
    - **ORM 연동**: MyBatis를 통해 다양한 데이터베이스에 저장할 수 있다.
    - **SQL 쿼리 분리**: SQL 쿼리를 Java 코드로부터 분리하여 관리 및 유지 보수가 용이하다.
    - **유연성**: 다양한 설정을 통해 원하는 방식으로 데이터를 저장할 수 있다.
- 단점
    - **설정 복잡성**: MyBatis 설정 및 SQL 매퍼 작성이 복잡할 수 있다.
    - **데이터베이스 종속**: 특정 데이터베이스에 종속적이다.
    - **오류 가능성**: 설정 오류 시 데이터 손상 가능성이 있다.

### MyBatisItemWriter 주요 구성 요소

- `SqlSessionTemplate`
    - MyBatis SqlSession 생성 및 관리를 위한 템플릿 객체이다.
- `SqlSessionFactory`
    - SqlSessionTemplate 생성을 위한 팩토리 객체이다.
- `StatementId`
    - 실행할 MyBatis SQL 맵퍼의 statement ID이다.
- `ItemToParameterConverter`
    - 객체를 ParameterMap으로 변경할수 있다.

<br>

---

## 2. MyBatisPagingItemReader 구현

`MyBatisPagingItemReader`를 활용하여 db의 **customer** 테이블로 부터 데이터를 읽어들이고 **flatfile**(csv)로 저장하는 로직을 구현해보자.

구현에 앞서 먼저 mybatis-spring 관련 **라이브러리 의존성**을 추가한다

```gradle
    // SpringBoot 3.2.2 기준 myBatis 관련 의존성 추가
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
    implementation 'org.mybatis:mybatis-spring:3.0.4'
    implementation 'org.mybatis:mybatis:3.5.16'
```

다음으로 myBatis에서 **쿼리 매퍼**로 사용할 **xml** 파일을 추가해준다.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.schooldevops.springbatch.batchstudy.jobs">

    <resultMap id="customerResult" type="com.schooldevops.springbatch.batchstudy.models.Customer">
        <result property="id" column="id"/>
        <result property="name" column="name"/>
        <result property="age" column="age"/>
        <result property="gender" column="gender"/>
    </resultMap>

    <select id="selectCustomers" resultMap="customerResult">
        SELECT id, name, age, gender
        FROM customer
        LIMIT #{_skiprows}, #{_pagesize}
    </select>
</mapper>
```

- `namespace`: 쿼리들을 그룹화해서 모아놓은 이름 공간이다.
- `resultMap`: 결과로 반환할 결과맵을 지정한다. 이는 db 칼럼과, java 필드 이름을 매핑한다.
- `select`: 쿼리를 지정한다.
- `#{_skiprows}`: 오프셋을 이야기하며, 쿼리 결과에서 얼마나 스킵할지 지정한다. pageSize를 지정했다면 자동으로 계산된다.
- `#{_pagesize}`: 한번에 가져올 페이지를 지정한다.

application.yml 파일에도 mapper 파일의 위치를 지정해준다.

```yaml
mybatis:
  mapper-locations: classpath:/mappers/**/*.xml
```

`MyBatisPagingItemReader` 빈을 구현해보자.

```java

@Slf4j
@Configuration
public class MyBatisReaderJobConfig {

	/**
	 * CHUNK 크기를 지정한다.
	 */
	public static final int CHUNK_SIZE = 2;
	public static final String ENCODING = "UTF-8";
	public static final String MYBATIS_CHUNK_JOB = "MYBATIS_CHUNK_JOB";

	@Autowired
	SqlSessionFactory sqlSessionFactory;

	@Bean
	public MyBatisPagingItemReader<Customer> myBatisItemReader() throws Exception {

		return new MyBatisPagingItemReaderBuilder<Customer>()
			.sqlSessionFactory(sqlSessionFactory)
			.pageSize(CHUNK_SIZE)
			.queryId("com.schooldevops.springbatch.batchstudy.jobs.selectCustomers")
			.build();
	}

...
```

- `sqlSessionFactory`: SqlSession을 생성하는 팩토리 객체를 지정한다.
- `pageSize`: 페이징 단위를 지정한다.
- `queryId`: 쿼리 매퍼 파일에서 사용할 쿼리 id를 지정한다.

나머지 Job, Step도 마저 구현한다.

```java
...

@Bean
public FlatFileItemWriter<Customer> CustomerCursorFlatFileItemWriter() {
	return new FlatFileItemWriterBuilder<Customer>()
		.name("CustomerCursorFlatFileItemWriter")
		.resource(new FileSystemResource("./output/Customer_new_v4.csv"))
		.encoding(ENCODING)
		.delimited().delimiter("\t")
		.names("Name", "Age", "Gender")
		.build();
}

@Bean
public Step CustomerJdbcCursorStep(JobRepository jobRepository,
	PlatformTransactionManager transactionManager) throws Exception {
	log.info("------------------ Init CustomerJdbcCursorStep -----------------");

	return new StepBuilder("CustomerJdbcCursorStep", jobRepository)
		.<Customer, Customer>chunk(CHUNK_SIZE, transactionManager)
		.reader(myBatisItemReader())
		.processor(new CustomerItemProcessor())
		.writer(CustomerCursorFlatFileItemWriter())
		.build();
}

@Bean
public Job CustomerJdbcCursorPagingJob(Step CustomerJdbcCursorStep, JobRepository jobRepository) {
	log.info("------------------ Init CustomerJdbcCursorPagingJob -----------------");
	return new JobBuilder(MYBATIS_CHUNK_JOB, jobRepository)
		.incrementer(new RunIdIncrementer())
		.start(CustomerJdbcCursorStep)
		.build();
}
```

### 실행 결과

먼저 customer 테이블에서 읽어들일 데이터를 저장해둔다.

![img.png](img.png)

스프링배치를 실행하여 FlatFileItemWriter 빈에서 출력 파일로 지정했던 `Customer_new_v4.csv` 파일이 생성되고, 데이터가 잘 읽어왔는지 확인해보자.

![img_1.png](img_1.png)

정상적으로 csv 파일이 생성되고 데이터를 읽어온 것을 확인할 수 있다.

<br>

---

## 3. MyBatisItemWriter 구현

`MyBatisItemWriter`를 활용하여 **flatfile**에서 데이터를 읽어들인 후 **customer2** 테이블에 저장하는 로직을 구현해보자.

먼저 **customer.csv** 파일에 불러올 데이터를 저장해두자.

![img_2.png](img_2.png)

다음에는 쿼리 매퍼 파일에 customer 테이블에 insert를 하기 위한 쿼리문을 추가한다.

```xml
...

<insert id="insertCustomers" parameterType="com.schooldevops.springbatch.batchstudy.models.Customer">
    INSERT INTO customer2(name, age, gender) VALUES (#{name}, #{age}, #{gender});
</insert>
        </mapper>
```

`MyBatisItemWriter` 빈을 구현해보자.

```java

@Slf4j
@Configuration
public class MybatisItemJobConfig {

	/**
	 * CHUNK 크기를 지정한다.
	 */
	public static final int CHUNK_SIZE = 100;
	public static final String ENCODING = "UTF-8";
	public static final String MYBATIS_ITEM_WRITER_JOB = "MYBATIS_ITEM_WRITER_JOB";

	@Autowired
	SqlSessionFactory sqlSessionFactory;

	@Bean
	public MyBatisBatchItemWriter<Customer> mybatisItemWriter() {
		return new MyBatisBatchItemWriterBuilder<Customer>()
			.sqlSessionFactory(sqlSessionFactory)
			.statementId("com.schooldevops.springbatch.batchstudy.jobs.insertCustomers")
			.build();
	}

...
```

- `sqlSessionFactory`: SqlSession을 생성하는 팩토리 객체를 지정한다.
- `statementId`: 쿼리 매퍼 파일에서 사용할 insert 쿼리문의 id를 지정한다.

나머지 Job, Step도 마저 구현한다.

```java
...

@Bean
public FlatFileItemReader<Customer> flatFileItemReader() {

	return new FlatFileItemReaderBuilder<Customer>()
		.name("FlatFileItemReader")
		.resource(new ClassPathResource("./customer.csv"))
		.encoding(ENCODING)
		.delimited().delimiter(",")
		.names("name", "age", "gender")
		.targetType(Customer.class)
		.build();
}

@Bean
public Step flatFileStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	log.info("------------------ Init flatFileStep -----------------");

	return new StepBuilder("flatFileStep", jobRepository)
		.<Customer, Customer>chunk(CHUNK_SIZE, transactionManager)
		.reader(flatFileItemReader())
		.writer(mybatisItemWriter())
		.build();
}

@Bean
public Job flatFileJob(Step flatFileStep, JobRepository jobRepository) {
	log.info("------------------ Init flatFileJob -----------------");
	return new JobBuilder(MYBATIS_ITEM_WRITER_JOB, jobRepository)
		.incrementer(new RunIdIncrementer())
		.start(flatFileStep)
		.build();
}
}
```

### 실행 결과

스프링 배치를 실행하여 customer.csv 파일의 데이터가 `customer2` 테이블에 잘 저장되었는지 확인해보자.

![img_3.png](img_3.png)

정상적으로 저장된 것을 확인할 수 있다.
