# 9주차 SPRING BATCH STUDY (written by @amazon7737)

KIDO 님의 SpringBatch 연재 시리즈를 보면서 스터디하는 과정을 진행한다.

9주차 : https://devocean.sk.com/blog/techBoardDetail.do?ID=167030

## 정리

## Custom ItemReader/ItemWriter 구현방법
Querydsl은 스프링배치에서 지원해주고 있지 않아 직접 인터페이스를 상속받아 구현해야합니다.

### QuerydslPagingItemReader

- Querydsl 을 활용할 수 있도록 직접 ItemReader 의 구현체로 작성
- AbstractPagingItemReader를 이용
- JPA 엔티티에 의존하지 않고 추상화된 쿼리 작성 가능
- 런타임 시 조건에 따라 동적으로 쿼리를 생성 가능

```java
public class QuerydslPagingItemReader<T> extends AbstractPagingItemReader<T> {

    private EntityManager em;
    private final Function<JPAQueryFactory, JPAQuery<T>> querySupplier;

    private final Boolean alwaysReadFromZero;

    public QuerydslPagingItemReader(EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize) {
        this(ClassUtils.getShortName(QuerydslPagingItemReader.class), entityManagerFactory, querySupplier, chunkSize, false);
    }

    public QuerydslPagingItemReader(String name, EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize, Boolean alwaysReadFromZero) {
        super.setPageSize(chunkSize);
        setName(name);
        this.querySupplier = querySupplier;
        this.em = entityManagerFactory.createEntityManager();
        this.alwaysReadFromZero = alwaysReadFromZero;

    }

    @Override
    protected void doClose() throws Exception {
        if (em != null)
            em.close();
        super.doClose();
    }

    @Override
    protected void doReadPage() {
        initQueryResult();

        JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(em);
        long offset = 0;
        if (!alwaysReadFromZero) {
            offset = (long) getPage() * getPageSize();
        }

        JPAQuery<T> query = querySupplier.apply(jpaQueryFactory).offset(offset).limit(getPageSize());

        List<T> queryResult = query.fetch();
        for (T entity: queryResult) {
            em.detach(entity);
            results.add(entity);
        }
    }

    private void initQueryResult() {
        if (CollectionUtils.isEmpty(results)) {
            results = new CopyOnWriteArrayList<>();
        } else {
            results.clear();
        }
    }
}
```

- AbstractPagingItemReader를 상속받아, doReadPage 메소드를 구현하면 됨.
- AbstractPagingItemReader는 어댑터 패턴

#### 생성자
```java
public QuerydslPagingItemReader(String name, EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize) {
    super.setPageSize(chunkSize);
    setName(name);
    this.querySupplier = querySupplier;
    this.em = entityManagerFactory.createEntityManager();

}
```

- name : ItemReader 구분하기 위한 이름
- EntityManagerFactory : JPA를 이용하기 위해 EntityManagerFactory 전달
- Function<JPAQueryFactory, JPAQuery> : JPAQuery 를 생성하기 위한 Functional Interface.
  - JPAQueryFactory를 파라미터로 전달 받음
  - 반환값은 JPAQuery 형태의 queryDSL 쿼리
- chunkSize : 한번에 페이징 처리할 페이지 크기
- alwaysReadFromZero : 항상 0부터 페이징을 읽을지 여부 지정. 만약 paging된 데이터 자체를 수정한다면 배치처리 누락이 발생할 수 있음. 그때 사용


- doClose()
  - AbstractPagingItemreader 자체 구현되어 있지만 EntityManager 자원을 해제하기 위해서 em.close() 수행

- doReadPage()
  - 구현해야하는 추상 메소드

```java
@Override
protected void doReadPage() {
    initQueryResult();

    JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(em);
    long offset = 0;
    if (!alwaysReadFromZero) {
        offset = (long) getPage() * getPageSize();
    }

    JPAQuery<T> query = querySupplier.apply(jpaQueryFactory).offset(offset).limit(getPageSize());

    List<T> queryResult = query.fetch();
    for (T entity: queryResult) {
        em.detach(entity);
        results.add(entity);
    }
}
```
- JPAQueryFactory 통해 queryDSL에 적용할 QueryFactory.
- 만약 alwaysReadFromZero 가 false 라면 offset , limit 을 계속 이동하면서 조회하도록 offset 연산
- querySupplier.apply
  - querySupplier 에 JPAQueryFactory를 적용 -> JPAQuery 생성
  - 페이징을 위해 offset, limit 을 계산된 offset과 pageSize (청크크기) 를 지정하여 페이징 처리


- initQueryResult
  - 매 페이징 결과를 반환할때 페이징 결과만 반환하기 위해 초기화.
  - 만약 결과 object가 초기화 되어 있지 않다면 CopyOnWriteArrayList 객체를 신규로 생성

#### Builder 생성
- 위 생성자가 복잡했기에, 빌더 패턴을 통해서 작성해보자.

```java
public class QuerydslPagingItemReaderBuilder<T> {

    private EntityManagerFactory entityManagerFactory;
    private Function<JPAQueryFactory, JPAQuery<T>> querySupplier;

    private int chunkSize = 10;

    private String name;

    private Boolean alwaysReadFromZero;

    public QuerydslPagingItemReaderBuilder<T> entityManagerFactory(EntityManagerFactory entityManagerFactory) {
        this.entityManagerFactory = entityManagerFactory;
        return this;
    }

    public QuerydslPagingItemReaderBuilder<T> querySupplier(Function<JPAQueryFactory, JPAQuery<T>> querySupplier) {
        this.querySupplier = querySupplier;
        return this;
    }

    public QuerydslPagingItemReaderBuilder<T> chunkSize(int chunkSize) {
        this.chunkSize = chunkSize;
        return this;
    }

    public QuerydslPagingItemReaderBuilder<T> name(String name) {
        this.name = name;
        return this;
    }

    public QuerydslPagingItemReaderBuilder<T> alwaysReadFromZero(Boolean alwaysReadFromZero) {
        this.alwaysReadFromZero = alwaysReadFromZero;
        return this;
    }

    public QuerydslPagingItemReader<T> build() {
        if (name == null) {
            this.name = ClassUtils.getShortName(QuerydslPagingItemReader.class);
        }
        if (this.entityManagerFactory == null) {
            throw new IllegalArgumentException("EntityManagerFactory can not be null.!");
        }
        if (this.querySupplier == null) {
            throw new IllegalArgumentException("Function<JPAQueryFactory, JPAQuery<T>> can not be null.!");
        }
        if (this.alwaysReadFromZero == null) {
            alwaysReadFromZero = false;
        }
        return new QuerydslPagingItemReader<>(this.name, entityManagerFactory, querySupplier, chunkSize, alwaysReadFromZero);
    }
}
```
- Builder 패턴을 이용하여 QuerydslPagingItemReader 객체를 생성하는 코드

### CustomItemWriter
- ItemWriter 인터페이스를 구현하여 직접 작성하는 ItemWriter 클래스
- 제공되지 않는 특정 기능을 구현할 때 사용
- write() 메소드를 구현하여 원하는 처리를 수행
- 사용할 라이브러리 및 객체 선언
- write() 메소드에서 데이터 처리 로직 구현


#### 장점
- ItemWriter 클래스에서 제공되지 않는 특정 기능을 구현 가능
- 다양한 방식으로 데이터 처리를 확장 가능
- 데이터 처리 과정을 완벽하게 제어

#### 단점
- ItemWriter 클래스 보다 개발 과정이 복잡함
- 테스트 작성이 어려울 수 있음
- 문제 발생 시 디버깅이 어려움

#### CustomService
```java
@Slf4j
@Service
public class CustomService {

    public Map<String, String> processToOtherService(Customer item) {

        log.info("Call API to OtherService....");

        return Map.of("code", "200", "message", "OK");
    }
}
```
- CustomSerivce 생성자 파라미터를 받고 , write 메소드를 구현
- write 메소드는 ItemWriter의 핵심 메소드
- Chunk 는 Customer 객체를 한 묶음으로 처리할 수 있도록 반복 수행 가능 (for문)
- processToOtherService 호출

#### CustomItemWriter
```java
@Slf4j
@Component
public class CustomItemWriter implements ItemWriter<Customer> {

    private final CustomService customService;

    public CustomItemWriter(CustomService customService) {
        this.customService = customService;
    }

    @Override
    public void write(Chunk<? extends Customer> chunk) throws Exception {
        for (Customer customer: chunk) {
            log.info("Call Porcess in CustomItemWriter...");
            customService.processToOtherService(customer);
        }
    }
}
```

#### 실습 실행 결과
```text
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.4)

2024-12-10T15:47:46.582+09:00  INFO 32431 --- [           main] o.s.s.sample.SampleApplication           : Starting SampleApplication using Java 17.0.7 with PID 32431 (/Users/amazon/lunch/스프링배치스터디/mysql/ch09/build/classes/java/main started by amazon in /Users/amazon/lunch/스프링배치스터디/mysql/ch09)
2024-12-10T15:47:46.583+09:00  INFO 32431 --- [           main] o.s.s.sample.SampleApplication           : No active profile set, falling back to 1 default profile: "default"
2024-12-10T15:47:46.776+09:00  INFO 32431 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2024-12-10T15:47:46.788+09:00  INFO 32431 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 8 ms. Found 0 JPA repository interfaces.
2024-12-10T15:47:46.871+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Hikari' of type [org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Hikari] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.880+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.datasource-org.springframework.boot.autoconfigure.jdbc.DataSourceProperties' of type [org.springframework.boot.autoconfigure.jdbc.DataSourceProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.881+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration$PooledDataSourceConfiguration' of type [org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration$PooledDataSourceConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.881+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'jdbcConnectionDetails' of type [org.springframework.boot.autoconfigure.jdbc.PropertiesJdbcConnectionDetails] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.890+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'dataSource' of type [com.zaxxer.hikari.HikariDataSource] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.895+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.jpa-org.springframework.boot.autoconfigure.orm.jpa.JpaProperties' of type [org.springframework.boot.autoconfigure.orm.jpa.JpaProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.895+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.jpa.hibernate-org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties' of type [org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.896+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.jdbc.metadata.DataSourcePoolMetadataProvidersConfiguration$HikariPoolDataSourceMetadataProviderConfiguration' of type [org.springframework.boot.autoconfigure.jdbc.metadata.DataSourcePoolMetadataProvidersConfiguration$HikariPoolDataSourceMetadataProviderConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.896+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'hikariPoolDataSourceMetadataProvider' of type [org.springframework.boot.autoconfigure.jdbc.metadata.DataSourcePoolMetadataProvidersConfiguration$HikariPoolDataSourceMetadataProviderConfiguration$$Lambda$500/0x0000000800ea3c38] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.898+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaConfiguration' of type [org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.900+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.transaction.TransactionManagerCustomizationAutoConfiguration' of type [org.springframework.boot.autoconfigure.transaction.TransactionManagerCustomizationAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.901+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'transactionExecutionListeners' of type [org.springframework.boot.autoconfigure.transaction.ExecutionListenersTransactionManagerCustomizer] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.903+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.transaction-org.springframework.boot.autoconfigure.transaction.TransactionProperties' of type [org.springframework.boot.autoconfigure.transaction.TransactionProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.903+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'platformTransactionManagerCustomizers' of type [org.springframework.boot.autoconfigure.transaction.TransactionManagerCustomizers] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.907+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.sql.init.DataSourceInitializationConfiguration' of type [org.springframework.boot.autoconfigure.sql.init.DataSourceInitializationConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.908+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.sql.init-org.springframework.boot.autoconfigure.sql.init.SqlInitializationProperties' of type [org.springframework.boot.autoconfigure.sql.init.SqlInitializationProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.910+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'dataSourceScriptDatabaseInitializer' of type [org.springframework.boot.autoconfigure.sql.init.SqlDataSourceScriptDatabaseInitializer] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.910+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration$DataSourceInitializerConfiguration' of type [org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration$DataSourceInitializerConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.912+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.batch-org.springframework.boot.autoconfigure.batch.BatchProperties' of type [org.springframework.boot.autoconfigure.batch.BatchProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:46.915+09:00  INFO 32431 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2024-12-10T15:47:46.987+09:00  INFO 32431 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@761956ac
2024-12-10T15:47:46.988+09:00  INFO 32431 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2024-12-10T15:47:47.011+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'batchDataSourceInitializer' of type [org.springframework.boot.autoconfigure.batch.BatchDataSourceScriptDatabaseInitializer] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.020+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'jpaVendorAdapter' of type [org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.022+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'entityManagerFactoryBuilder' of type [org.springframework.boot.orm.jpa.EntityManagerFactoryBuilder] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.026+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'persistenceManagedTypes' of type [org.springframework.orm.jpa.persistenceunit.SimplePersistenceManagedTypes] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.041+09:00  INFO 32431 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2024-12-10T15:47:47.060+09:00  INFO 32431 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.5.3.Final
2024-12-10T15:47:47.072+09:00  INFO 32431 --- [           main] o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
2024-12-10T15:47:47.205+09:00  INFO 32431 --- [           main] o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
2024-12-10T15:47:47.599+09:00  INFO 32431 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
2024-12-10T15:47:47.600+09:00  INFO 32431 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-12-10T15:47:47.602+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'entityManagerFactory' of type [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.602+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'entityManagerFactory' of type [jdk.proxy2.$Proxy90] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.603+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'transactionManager' of type [org.springframework.orm.jpa.JpaTransactionManager] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). Is this bean getting eagerly injected into a currently created BeanPostProcessor [jobRegistryBeanPostProcessor]? Check the corresponding BeanPostProcessor declaration and its dependencies.
2024-12-10T15:47:47.606+09:00  WARN 32431 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration$SpringBootBatchConfiguration' of type [org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration$SpringBootBatchConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying). The currently created BeanPostProcessor [jobRegistryBeanPostProcessor] is declared through a non-static factory method on that class; consider declaring it as static instead.
2024-12-10T15:47:47.644+09:00  INFO 32431 --- [           main] o.s.s.s.c.QueryDSLPagingReaderJobConfig  : ------------ Init customerQuerydslPagingStep -------------
2024-12-10T15:47:47.702+09:00  INFO 32431 --- [           main] o.s.s.s.c.QueryDSLPagingReaderJobConfig  : ------------ Init customerJpaPagingJob -------------
2024-12-10T15:47:47.834+09:00  INFO 32431 --- [           main] o.s.s.sample.SampleApplication           : Started SampleApplication in 1.404 seconds (process running for 1.627)
2024-12-10T15:47:47.835+09:00  INFO 32431 --- [           main] o.s.b.a.b.JobLauncherApplicationRunner   : Running default command line with: []
2024-12-10T15:47:47.899+09:00  INFO 32431 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=QUERYDSL_PAGING_CHUNK_JOB]] launched with the following parameters: [{'run.id':'{value=3, type=class java.lang.Long, identifying=true}'}]
2024-12-10T15:47:47.916+09:00  INFO 32431 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [customerJpaPagingStep]
2024-12-10T15:47:48.093+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@18ab513d
2024-12-10T15:47:48.094+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@659e003e
2024-12-10T15:47:48.100+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@7568134c
2024-12-10T15:47:48.100+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@28c75c93
2024-12-10T15:47:48.105+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@3f1d4ecf
2024-12-10T15:47:48.105+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@6bbac73d
2024-12-10T15:47:48.109+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@7b55f0a6
2024-12-10T15:47:48.109+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@3fc7abf6
2024-12-10T15:47:48.114+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@144792d5
2024-12-10T15:47:48.114+09:00  INFO 32431 --- [           main] o.s.s.sample.jobs.CustomerItemProcessor  : Item Processor ------- org.schooldevops.springbatch.sample.domain.Customer@1da61a29
2024-12-10T15:47:48.122+09:00  INFO 32431 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [customerJpaPagingStep] executed in 205ms
2024-12-10T15:47:48.131+09:00  INFO 32431 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=QUERYDSL_PAGING_CHUNK_JOB]] completed with the following parameters: [{'run.id':'{value=3, type=class java.lang.Long, identifying=true}'}] and the following status: [COMPLETED] in 222ms
2024-12-10T15:47:48.133+09:00  INFO 32431 --- [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2024-12-10T15:47:48.134+09:00  INFO 32431 --- [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2024-12-10T15:47:48.137+09:00  INFO 32431 --- [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.

Process finished with exit code 0
```

--- 
#### 발표자 업로드 참고 소스
<https://github.com/amazon7737/Spring-Batch-Study/tree/main/ch09>

#### 스터디원 정리 블로그
<https://github.com/mardi2020/Spring-batch-study/blob/main/docs/9%EC%A3%BC%EC%B0%A8.md>

<https://github.com/won-js/spring-batch-study/tree/main/docs/week9>

<https://velog.io/@hanni/Custom-ItemReaderItemWriter>
  
<https://github.com/chanwoo040531/batch-study/tree/master/assignment09>

<https://github.com/youngkim90/spring-batch-study/blob/main/study/9_week/9_week.study.md>

<https://more-n.tistory.com/58>

<https://github.com/sajacaros/spring-batch-tutorial/blob/main/docs/09_Custom.md>