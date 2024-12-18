# 09 CustomItem Reader/Writer
* Querydsl을 이용하여 QuerydslPagingItemReader 구현
* 타서비스(pseudo code)를 호출하는 CustomItemWriter 구현
## QuerydslPagingItemReader 개요
* Querydsl은 Spring Batch의 공식 ItemReader가 아님
* AbstractPagingItemReader를 이용하여 Querydsl을 활용하는 ItemReader 구현
  - Querydsl 기능 활용: 강력하고 유연한 쿼리 기능 사용
  - JPA 엔티티 추상화: 코드 유지 관리성을 높일수 있음
  - 동적 쿼리 지원: 조건에 따라 동적 쿼리 생성
## QuerydslPagingReader 생성하기
- 생성자 주요 파라미터
  - Function<JPAQueryFactory, JPAQuery<T>> querySuppler
- `doReadPage()` 구현
  - 구현해야될 추상 메소드
  - `AbstractPagingItemReader`의 `doRead()`에서 호출됨
  - [소스](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/AbstractPagingItemReader.java#L138)
``` 
public class QuerydslPagingItemReader<T> extends AbstractPagingItemReader<T> {
    private EntityManager em;
    private final Function<JPAQueryFactory, JPAQuery<T>> querySupplier;
    private final Boolean alwaysReadFromZero;
    
    public QuerydslPagingItemReader(
        EntityManagerFactory entityManagerFactory, 
        Function<JPAQueryFactory, JPAQuery<T>> querySuppler,
        int chunkSize,
        Boolean alwaysReadFromZero
    ) {
        super.setPageSize(chunkSize);
        setName(name);
        this.querySupplier = querySupplier;
        this.em = entityManagerFactory.createEntityManager();
        this.alwaysReadFromZero = alwaysReadFromZero;
    }
    
    @Override
    protected void doClose() throws Exception {
        if(em != null)
            em.close();
        super.doClose();
    }
    
    @Override
    protected void doReadPage() {
        initQueryResult();
        
        JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(em);
        long offset = 0;
        if(!alwaysReadFromZero) {
            offset = (long)getPage() * getPageSize();
        }
        JPAQuery<T> query = querySupplier.apply(jpaQueryFactory).offset(offset).limit(getPageSize());
        
        List<T> queryResult = query.fetch();
        for(T entity: queryResult) {
            em.detach(entity);
            results.add(entity);
        }
    }
    
    private void initQueryResult() {
        if(CollectionUtils.isEmpty(results)) {
            results = new CopyOnWriteArrayList<>();
        } else {
            results.clear();
        }
    }
}
```
### QuerydslPagingItemReaderBuilder 구현
- builder 패턴을 이용해서 QuerydslPagingItemReader를 생성하는 헬퍼 클래스
```
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
### Job, Step 구성
``` 
public class QueryDSLPagingReaderJobConfig {

    /**
     * CHUNK 크기를 지정한다.
     */
    public static final int CHUNK_SIZE = 2;
    public static final String ENCODING = "UTF-8";
    public static final String QUERYDSL_PAGING_CHUNK_JOB = "QUERYDSL_PAGING_CHUNK_JOB";

    @Autowired
    DataSource dataSource;

    @Autowired
    EntityManagerFactory entityManagerFactory;

    @Bean
    public QuerydslPagingItemReader<Customer> customerQuerydslPagingItemReader() {
        return new QuerydslPagingItemReaderBuilder<Customer>()
                .name("customerQuerydslPagingItemReader")
                .entityManagerFactory(entityManagerFactory)
                .chunkSize(2)
                .querySupplier(
                        jpaQueryFactory -> 
                        jpaQueryFactory
                            .select(QCustomer.customer)
                            .from(QCustomer.customer)
                            .where(QCustomer.customer.age.gt(20))
                )
                .build();
    }

    @Bean
    public FlatFileItemWriter<Customer> customerQuerydslFlatFileItemWriter() {

        return new FlatFileItemWriterBuilder<Customer>()
                .name("customerQuerydslFlatFileItemWriter")
                .resource(new FileSystemResource("./output/customer_new_v2.csv"))
                .encoding(ENCODING)
                .delimited().delimiter("\t")
                .names("Name", "Age", "Gender")
                .build();
    }


    @Bean
    public Step customerQuerydslPagingStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) throws Exception {
        log.info("------------------ Init customerQuerydslPagingStep -----------------");

        return new StepBuilder("customerJpaPagingStep", jobRepository)
                .<Customer, Customer>chunk(CHUNK_SIZE, transactionManager)
                .reader(customerQuerydslPagingItemReader())
                .processor(new CustomerItemProcessor())
                .writer(customerQuerydslFlatFileItemWriter())
                .build();
    }

    @Bean
    public Job customerJpaPagingJob(Step customerJdbcPagingStep, JobRepository jobRepository) {
        log.info("------------------ Init customerJpaPagingJob -----------------");
        return new JobBuilder(QUERYDSL_PAGING_CHUNK_JOB, jobRepository)
                .incrementer(new RunIdIncrementer())
                .start(customerJdbcPagingStep)
                .build();
    }
}

```
## CustomItemWriter 개요
- Spring Batch에사 제공하는 기본 ItemWriter 인터페이스를 구현하여 구현
### 구성 요소
- ItemWriter 인터페이슥 ㅜ현
- 데이터 처리 로직 구현
### 장점
- 유연성
- 확장성
- 제어 가능성
### 단점
- 개발 복잡성
- 테스트 어려움
- 디버깅 어려움
### 구현
- 서비스 구현(pseudo code)
``` 
public class CustomService {
    public Map<String, String> processToOtherService(Customer item) {
        log.info("Call API to OtherService....");
        return Map.of("code", "200", "message", "OK");
    }
}
```
- CustomItemWriter 작성
```  
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
- Job, Step 구성
``` 
public class MybatisItemWriterJobConfig {
   public static final int CHUNK_SIZE = 100;
    public static final String ENCODING = "UTF-8";
    public static final String MY_BATIS_ITEM_WRITER = "MY_BATIS_ITEM_WRITER";

    @Autowired
    DataSource dataSource;

    @Autowired
    SqlSessionFactory sqlSessionFactory;

    @Autowired
    CustomItemWriter customItemWriter;

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
                .writer(customItemWriter)
                .build();
    }

    @Bean
    public Job flatFileJob(Step flatFileStep, JobRepository jobRepository) {
        log.info("------------------ Init flatFileJob -----------------");
        return new JobBuilder(MY_BATIS_ITEM_WRITER, jobRepository)
                .incrementer(new RunIdIncrementer())
                .start(flatFileStep)
                .build();
    }
}
```
