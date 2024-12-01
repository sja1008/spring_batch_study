## SpringBatch 08: CompositeItemProcessor 으로 여러단계에 걸쳐 데이터 Transform하기

### CompositeItemProcessor: ItemProcessor 인터페이스 구현체
> 여러 개의 ItemProcessor를 하나의 Processor로 연결해 여러 단계의 처리 수행

- Delegates: 처리를 수행할 ItemProcessor 목록
- TransactionAttribute: 트랜잭션 속성 설정

<details>
<summary>🔽 TransactionAttribute </summary>

### ChatGPT에게 물어본 내용
TransactionAttribute는 Spring Batch에서 **CompositeItemProcessor**와 같은 프로세서 또는 단위 작업(step) 내에서 트랜잭션의 동작 방식을 제어하기 위해 사용됩니다.

TransactionAttribute는 Spring의 트랜잭션 관리에서 제공하는 인터페이스로, 트랜잭션의 속성(전파 수준, 격리 수준, 타임아웃 등)을 설정하는 역할을 합니다.

기본적으로 CompositeItemProcessor는 트랜잭션 경계 내에서 작동합니다. 이는 프로세서에서 발생하는 작업이 현재의 트랜잭션에 포함된다는 뜻이며,
TransactionAttribute를 사용하면 CompositeItemProcessor 또는 개별 프로세서의 트랜잭션 동작을 세부적으로 제어할 수 있습니다.

#### 주요 TransactionAttribute 설정
TransactionAttribute를 사용하면 다음 속성을 설정할 수 있습니다:
1.	전파 수준(Propagation): 트랜잭션이 이미 존재하는 경우 새로운 트랜잭션을 생성할지, 기존 트랜잭션을 사용할지 결정
   
       예: REQUIRED, REQUIRES_NEW, MANDATORY 등
  	
2.	격리 수준(Isolation Level): 트랜잭션이 다른 트랜잭션과 데이터를 공유하는 방식을 설정

  	예: READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE 등
  	
3.	읽기 전용(Read-only): 트랜잭션이 읽기 전용으로 작동할지 설정 (쓰기 작업을 방지할 수 있음)
   
4.	타임아웃(Timeout): 트랜잭션이 특정 시간 초과 후 롤백되도록 설정

5.	롤백 규칙(Rollback Rules): 특정 예외 발생 시 트랜잭션을 롤백하도록 설정

</details>

장점
1. 단계별 처리: 여러 단계로 나누어 처리를 수행하므로 코드를 명확하고 이해하기 쉽게 만들어줌
2. 재사용: 각 단계별 Processor를 재사용하여 다른 Job에서도 가져다 사용 가능
3. 유연성: 다양한 ItemProcessor를 조합해 원하는 처리 과정 구현

단점
1. 설정 복잡성: 여러 개의 Processor를 설정하고 관리해야 하므로 설정이 복잡해질 수 있음
2. 성능 저하: 여러 단계의 처리 과정을 거치므로 성능 저하 가능성

### 실습 주요 내용
``` java
/* 이름과 성별을 소문자로 변경하는 item processor */
public class LowCaseItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) {
        item.setName(item.getName().toLowerCase());
        item.setGender(item.getGender().toLowerCase());
        return item;
     }
}
/* 현재 나이에 20을 더하는 item processor */
public class After20YearsItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) {
        item.setAge(item.getAge() + 20);
        return item;
    }
}
```
#### CompositeItemProcessor
빌더를 이용하여 delegates()로 위에 작성한 item processor를 전달
``` java
 @Bean
    public CompositeItemProcessor<Customer, Customer> compositeItemProcessor() {
        return new CompositeItemProcessorBuilder<Customer, Customer>()
                .delegates(List.of(
                        new LowerCaseItemProcessor(),
                        new After20YearsItemProcessor()
                ))
                .build();
}
```
#### output.csv
이름과 성별이 소문자로 변경되고 나이는 + 20살이 된 결과가 저장됨

CompositeItemProcessor의 리스트에 저장된 item processor 순으로 적용하여 데이터를 변경할 할 수 있음
