---
title:  "스프링 @Transactional 멀티스레드 환경에서 사용하기"
excerpt: "@Transactional 을 멀티스레드 환경에서 사용할 때 주의점과 스레드 안전한 방법을 알아보겠습니다."

categories:
  - Spring
tags:
  - 스프링
  - 멀티스레드
  - DB lock
last_modified_at: 2021-08-07T08:06:00-05:00
---

# Intro
---
인턴을 하면서 처음으로 받은 이슈를 진행하면서 생긴 궁금증을 정리하고자 합니다.

관련 코드는 [여기](https://github.com/ydh6226/blog-code/tree/master/spring-transactionl-multi-thread) 에 있습니다. 

해당 이슈는 파일을 다운로드 할 때 마다 해당 파일의 다운로드 카운트를 증가시켜 통계용으로 조회할 수 있도록
기능을 추가 시키는 것이였습니다. 단순히 파일의 id를 이용해서 count값을 증가시키고 업데이트하면 되는 것이였지만, 
이때 제가 생각이 든건 다음과 같습니다.

## 쿼리에서 로직 빼기 
(참고: [최범균님 유튜브](https://www.youtube.com/watch?v=fnH_SR3n9Ew))

저는 입사전 jpa를 주로 이용했기 때문에 거의 99퍼센트로 '데이터를 조회하고 코드에서 값 변경 후 업데이트' 하는
방법으로 코드를 작성했습니다. 하지만, 사내에서는 mybatis를 사용하고 있어서 쿼리에서 로직이 들어가있는 경우도 있었고, 
자바 코드에 로직이 들어가있는 경우도 있었습니다.  

제가 생각하기에는 쿼리에서 로직을 빼는것이 가독성, 재사용성 면에서 좋다고 생각했고, 사내에 어떤 상황에서는 어떻게 처리하라는
컨벤션이 없었기 때문에 코드상에 로직을 넣는방법으로 진행했습니다.


# 구현 코드

---
개발 환경: springboot, mybatis, h2 db

구현코드는 다음과 같습니다.
~~~java
@Data
public class File {
    int id;
    int count;
    
    public void increaseCount() {
        count++;
    }
}


@Service
public class NumberService {
    @Transactional
    public void increaseCount(long idx) {
        File file = numberRepository.findById(idx);
        file.increaseCount();
        numberRepository.update(file);
    }
}

@Repository
public interface FileRepository {
    int update(File file);
}

<mapper namespace="FileRepository">
    <update id="update">
        update file
        set count = #{count}
        where id = #{id};
    </update>
</mapper>
~~~
단순히 select하고 카운트 증가시키고 update 하는것으로 평소 같았으면 그대로 넘어갔겠지만 
실제 배포되는 서비스이기 때문에 '이게 과연 스레드 안전할까?' 라고 생각하며 테스트를 해봤습니다.

테스트를 위해 테스트 코드를 추가하고 NumberService :: increaseCount 메서드를 다음과 같이 수정했습니다.
~~~java
@SpringBootTest
class FileServiceTest {

    @Autowired
    NumberService numberService;

    @Autowired
    FileRepository fileRepository;
    
    private ExecutorService service = Executors.newFixedThreadPool(NUMBER_OF_THREADS);
    private CountDownLatch latch = null;

    private static final long IDX = 0;
    private static final int NUMBER_OF_THREADS = 100;
    private static final int REPETITION_COUNT = 100;
    
    @BeforeEach
    public void initDb() {
        service = Executors.newFixedThreadPool(NUMBER_OF_THREADS);
        latch = new CountDownLatch(NUMBER_OF_THREADS);
    }

    
    @Test
    public void increaseCountInQuery() throws Exception {
        executorServiceIteration(NUMBER_OF_THREADS, () -> {
            numberService.increaseCount(IDX, REPETITION_COUNT);
            latch.countDown();
        });
        latch.await();
        assertThat(fileRepository.findById(IDX).getCount())
                .isEqualTo(NUMBER_OF_THREADS * REPETITION_COUNT);
    }
}

@Slf4j
@Service
public class NumberService {
    @Transactional
    public void increaseCount(long idx, int repetitionCount) {
        log.info("service start");
        for (int i = 0; i < repetitionCount; i++) {
            File file = fileRepository.findById(idx);
            log.info("read [count : {}]", file.getCount());
            file.increaseCount();
            fileRepository.update(file);
            log.info("update [count : {}]", file.getCount());
        }
        log.info("service end");
    }
}
~~~
스레드 100개를 생성하고 각각의 스레드에서 로직을 100번씩 돌리는 테스트입니다.
하나의 로직에서 카운트롤 1 증가시키므로 카운트 결과값은 10000 이여야 합니다.

결과는 처참합니다. (~~서비스 장애날뻔 했다...~~)
~~~java
org.opentest4j.AssertionFailedError: 
expected: 10000
but was : 100
~~~

<details>
<summary>로그보기</summary>
<div markdown="1">

~~~java
2021-08-07 15:40:18.267  INFO 2936 --- [           main] com.zaxxer.hikari.HikariDataSource  : HikariPool-1 - Starting...
2021-08-07 15:40:18.421  INFO 2936 --- [           main] com.zaxxer.hikari.HikariDataSource  : HikariPool-1 - Start completed.
2021-08-07 15:40:18.469  INFO 2936 --- [ool-2-thread-30] com.study.service.NumberService     : service start
2021-08-07 15:40:18.469  INFO 2936 --- [ool-2-thread-36] com.study.service.NumberService     : service start
2021-08-07 15:40:18.469  INFO 2936 --- [ool-2-thread-33] com.study.service.NumberService     : service start
2021-08-07 15:40:18.469  INFO 2936 --- [ool-2-thread-20] com.study.service.NumberService     : service start
2021-08-07 15:40:18.469  INFO 2936 --- [pool-2-thread-9] com.study.service.NumberService     : service start
2021-08-07 15:40:18.469  INFO 2936 --- [ool-2-thread-24] com.study.service.NumberService     : service start
2021-08-07 15:40:18.472  INFO 2936 --- [ool-2-thread-16] com.study.service.NumberService     : service start
2021-08-07 15:40:18.475  INFO 2936 --- [ool-2-thread-27] com.study.service.NumberService     : service start
2021-08-07 15:40:18.478  INFO 2936 --- [ool-2-thread-10] com.study.service.NumberService     : service start
2021-08-07 15:40:18.481  INFO 2936 --- [ool-2-thread-21] com.study.service.NumberService     : service start
2021-08-07 15:40:18.486  INFO 2936 --- [ool-2-thread-13] com.study.service.NumberService     : service start
2021-08-07 15:40:18.491  INFO 2936 --- [ool-2-thread-28] com.study.service.NumberService     : service start
2021-08-07 15:40:18.496  INFO 2936 --- [ool-2-thread-25] com.study.service.NumberService     : service start
2021-08-07 15:40:18.499  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : service start
2021-08-07 15:40:18.502  INFO 2936 --- [pool-2-thread-5] com.study.service.NumberService     : service start
2021-08-07 15:40:18.505  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : service start
2021-08-07 15:40:18.507  INFO 2936 --- [ool-2-thread-29] com.study.service.NumberService     : service start
2021-08-07 15:40:18.510  INFO 2936 --- [ool-2-thread-12] com.study.service.NumberService     : service start
2021-08-07 15:40:18.513  INFO 2936 --- [ool-2-thread-35] com.study.service.NumberService     : service start
2021-08-07 15:40:18.524  INFO 2936 --- [pool-2-thread-2] com.study.service.NumberService     : service start
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-25] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-20] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.535  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-13] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.535  INFO 2936 --- [pool-2-thread-9] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-10] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-36] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-28] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-29] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-21] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-30] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-16] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-12] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-24] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-27] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.534  INFO 2936 --- [ool-2-thread-33] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.535  INFO 2936 --- [pool-2-thread-2] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.537  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 1]
2021-08-07 15:40:18.535  INFO 2936 --- [ool-2-thread-35] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.535  INFO 2936 --- [pool-2-thread-5] com.study.service.NumberService     : read [count : 0]
2021-08-07 15:40:18.538  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 1]
2021-08-07 15:40:18.539  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 2]
2021-08-07 15:40:18.540  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 2]
2021-08-07 15:40:18.541  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 3]
2021-08-07 15:40:18.542  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 3]
2021-08-07 15:40:18.542  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 4]
2021-08-07 15:40:18.543  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 4]
2021-08-07 15:40:18.544  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 5]

생략

2021-08-07 15:40:18.663  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 98]
2021-08-07 15:40:18.663  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 98]
2021-08-07 15:40:18.664  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 99]
2021-08-07 15:40:18.665  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : read [count : 99]
2021-08-07 15:40:18.665  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : update [count : 100]
2021-08-07 15:40:18.665  INFO 2936 --- [ool-2-thread-37] com.study.service.NumberService     : service end
2021-08-07 15:40:18.667  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : update [count : 1]
2021-08-07 15:40:18.667  INFO 2936 --- [pool-2-thread-7] com.study.service.NumberService     : service start
2021-08-07 15:40:18.667  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : read [count : 1]
2021-08-07 15:40:18.668  INFO 2936 --- [pool-2-thread-7] com.study.service.NumberService     : read [count : 100]
2021-08-07 15:40:18.668  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : update [count : 2]
2021-08-07 15:40:18.669  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : read [count : 2]
2021-08-07 15:40:18.669  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : update [count : 3]
2021-08-07 15:40:18.670  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : read [count : 3]
2021-08-07 15:40:18.670  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : update [count : 4]
2021-08-07 15:40:18.671  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : read [count : 4]
2021-08-07 15:40:18.671  INFO 2936 --- [ool-2-thread-22] com.study.service.NumberService     : update [count : 5]

생략

2021-08-07 15:41:37.854  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : read [count : 96]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : update [count : 97]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : read [count : 97]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : update [count : 98]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : read [count : 98]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : update [count : 99]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : read [count : 99]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : update [count : 100]
2021-08-07 15:41:37.855  INFO 2948 --- [ool-2-thread-24] com.study.service.NumberService     : service end

org.opentest4j.AssertionFailedError: 
expected: 10000
but was : 100
Expected :10000
Actual   :100
~~~
</div>
</details>
<br>

# 문제파악

---
우선 로그를 보면서 상황을 정리해보겠습니다.

1. 다수의 스레드에서 increaseCount 메서드에 들어가고 read 를 한다.
2. 37번 스레드가 다수의 스레드 중 처음으로 update 를 한다.
3. 37번스레드가 루프를 백번돌고 increaseCount 메서드를 빠져 나오면서 commit 을 할때까지 
   다른 스레드가 read, update 를 못한다.
   (로그 상으로는 37번 스레드의 update 이후에 35, 5번 스레드가 read 를 하지만 로그 시간을 보면
   37번 스레드의 update 이전에 read 를 했다. 로그 출력이되면서 순서가 바뀐듯 하다.)
   >  왜 increaseCount 메서드에 synchronized 안걸었는데 다른 스레드가 접근을 못하지?

   
4. 37번 스레드가 commit 을 한 후 22번 스레드가 update 를 하고 commit 을 할 때까지 다른 스레드가 접근을
못한다.
   
5. 1 ~ 4 반복

## 업데이트락 (Update lock)
우선 '왜 increaseCount 메서드에 synchronized 안걸었는데 다른 스레드가 접근을 못하지?' 에 대한 이유를 알아보겠습니다.

결론부터 말하면, 처음 update 를 한 트랜잭션에서 해당 레코드에 대해 업데이트락을 걸기 때문입니다.
업데이트락이 걸렸기 때문에 다른 트랜잭션에서는 해당 레코드에 update 를 하지 못합니다. 
**하지만 값을 수정하고 커밋하기전에 이미 다른 트랜잭션에서 해당 값을 read 했으므로 데이터 무결성이 깨집니다.**

<center>
<img src="https://user-images.githubusercontent.com/53700256/128600335-088d183a-0ff6-43d1-96c5-30623df43828.png"><br>
<i>Lock Compatibility Matrix</i>
</center>

update가 안되는건 알겠는데 read는 왜 못했을까요? 
바로 db connection pool 개수 때문이였습니다.
상황을 재연하면 다음과 같습니다. (스레드 100개, pool 개수 20개로 가정)

1. 먼저 increaseCount 메서드 내부에서 read 를 한 20개의 스레드가 커넥션 풀에서 커넥션을 가져온다.
2. 커넥션을 가지고있는 스레드중 한 스레드가 update를 하면 업데이트락이 걸려서 나머지 19개의 update를 못하기 때문에 대기한다.
3. 커넥션풀에 사용가능한 커넥션이 없으므로 나머지 80개의 스레드는 커넥션을 얻지 못하고 커넥션이 없기 때문에 read 도 못한다.

커넥션 풀 개수를 충분히 늘린다면 첫번째 스레드에서 update를 하더라도 다른 스레드가 read를 하는 로그를 확인할 수 있습니다.

처음엔 첫번째 update 이후에 read를 못하길래 베타락(Exclusive lock)이 걸린줄 알았는데, 결국은 pool size 때문에 마치
베타락이 걸린것처럼 보인 것 이였습니다. (~~이게 커넥션 풀까지 생각해야되는 문제 였다니...~~)

> 한 트랜잭션에서 select 이후에 update 를 한다면, select 할 때는 락이 걸리지 않고 update 할 때만 락이 걸립니다.
> update는 필터링(where 절) 을 할 때 필터링된 레코드에 대해 업데이트락이 걸리며 
> 필터링된 결과에 대해 실제로 update를 할 때 베타락으로 전환되고 update종류 후에는 다시 업데이트락으로 전환됩니다.
> 업데이트락은 트랜잭션종료 시점(commit or rollback)까지 지속됩니다.
> 
> 공유락은 트랜잭션 격리수준(isolation level)이 serializable일 때만 걸리며 이때 걸린 공유략은 트랜잭션이 종료 될 때까지
> 지속됩니다.

# 해결방법

---
## 1. 쿼리에 로직 넣기
이 상황에서는 쿼리에 로직을 넣는 방법이 가장 빠르고 간편합니다.
~~~java
<update id="increaseCountInQuery">
  update file set count = count + 1 where id = #{idx}
</update>
~~~
위에서 설명한 문제가 발생한 근본적인 문제가 read 하고 update를 하는 것이기 때문에 'read 를 하고 코드상에서 값을 변경' 을 
안하고 update 쿼리안에서 카운트를 증가시키게 되면 무결성을 지키면서 로직을 수행할 수 있습니다. 
또한 쿼리 하나로 카운트를 증가시킬 수 있기 때문에 속도도 빠릅니다.

## 2. select for update
select for update 문법은 값을 변경하려는 목적으로 조회를 하는 방법으로 조회할 때 부터 업데이트락을 거는 방법입니다.
~~~java
<select id="findByIdWithSelectForUpdate" resultType="com.study.domain.File">
    select * from file where id = #{idx} for update
</select>
~~~
다만, 아무래도 카운트를 증가시킬 떄마다 쿼리가 두번 나가기 때문에(select, update) 속도가 1번 방법 보다 다소 느립니다.
평균적으로 1번 방법은 2.5초, 2번방법은 5초가 걸렸습니다.

수행해야되는 로직이 간단하다면 1번방법을 사용하는것이 좋겠지만 로직이 조금만 복잡해지면 쿼리에 로직을 넣기에는 가독성도 떨어지고
쿼리로는 해결을 하지 못할수도 있기 떄문에 2번방법을 사용하는게 좋을것 같습니다.

<details>
<summary>해결방법인줄 알았으나 아니였던 것</summary>
<div markdown="1">

## ⚠ @Transactional 격리수준 serializable 로 설정하기 ⚠
격리수준을 serializable 로 높혀 read 할 떄 공유략이 걸리도록 하더라도 공유락 끼리는 호환 되기 때문에 이 방법은 불가능합니다.
첫 번째 트랜잭션이 update 를 해서 공유락이 업데이트락으로 변경되더라도 이미 다른 트랜잭션에서 값을 read 했기 때문에 무결성이 깨집니다. 

## 💩 increaseCount 메서드에 synchronized 사용하기 💩
사실 select for update 방법을 찾아내기 전에 이 방법부터 떠올랐습니다. 'increaseCount 메서드에 
스레드가 하나만 접근할 수 있게 하면 되는거 아닌가?' 라고 생각했기 때문이죠. (~~누구나 그럴싸한 계획을 갖고 있다. 쳐 맞기 전까지는~~)

코드는 아래와 같이 synchronized 키워드만 추가했습니다.

~~~java
@Transactional
public synchronized void increaseInCodeSync(long idx, int repetitionCount) {
    for (int i = 0; i < repetitionCount; i++) {
        File file = fileRepository.findById(idx);
        file.increaseCount();
        int row = fileRepository.update(file);
    }
}
~~~

결과는...

~~~java
org.opentest4j.AssertionFailedError: 
expected: 10000
but was : 9900
~~~
synchronized 를 사용하지 않았을 때보단 카운트가 많이 증가했지만 역시 무결성이 깨집니다.

이유는 [stack overflow](https://stackoverflow.com/questions/41767860/spring-transactional-with-synchronized-keyword-doesnt-work) 를
보면 알 수 있습니다. 정리를 하면 다음과 같습니다.
@Transactional 은 프록시 형태로 동작하기 때문에 프록시 객체가 increaseInCode 메서드를 감싸서 실행하게 됩니다.
아래와 같은 형태로 말이죠.
~~~java
public class FileServiceProxy {
  private final FileService file;
  private final TransactonManager tm = TransactionManager.getInstance();
  
  public FileServiceProxy(FileService fileService) {
    this.fileService = fileService;
  }
  
  public void increaseInCode(long idx, int repetitionCount) {
    try {
      tm.begin();
      fileService.increaseInCode(bookName);
      tm.commit();
    } catch (Exception e) {
      tm.rollback();
    }
  }
}
~~~

문제가 발생하는 상황은 아래와 같습니다.

------- FileServiceProxy: 트랜잭션 시작, 다른 스레드들도 트랜잭션은 시작됨  
------- increaseInCode 메서드 접근, synchronized이기 때문에 한 스레드에서만 접근 가능  
------- increaseInCode 메서드에 들어온 스레드가 select, update 수행  
------- increaseInCode 메서드 종료, synchronized 끝  
------- **이때 다른 트랜잭션에서 값을 읽어 간다면??**  
------- FileServiceProxy: commit, 트랜잭션 종료  

commit 은 synchronized 가 걸린 메서드 안에서 일어나는게 아니라 메서드 종료후에 발생합니다. 
따라서 매서드 종료 후 아직 commit 을 하기전에 다른 트랜잭션에서 값을 읽어 간다면 무결성이 깨집니다.  
@Transactional 을 사용하지 않으면 카운트가 정확하게 올라가지만 원자성(Atomicity)이 깨지기 때문에
오히려 오히려 더 큰 장애가 발생할 수도 있습니다.

</div>
</details>

<br>
<br>
# 결론

---
쿼리에서 로직을 빼기 위해 다른 방법을 많이 찾아보긴 했지만 결국 사용한 방법은 '쿼리에 로직 넣기' 입니다.
~~~java
<update id="increaseCountInQuery">
  update file set count = count + 1 where id = #{idx}
</update>
~~~
로직이 간단해서 mapper 파일을 보고 이해가 안되는 것도 아니고 성능도 가장 잘나오기 때문입니다.

하지만 로직이 복잡해져서 쿼리에 로직을 넣을 수 없을 때는 다른 방법을 사용해야겠습니다.
상황에 따라서 방법이 많이 달라지겠지만 간단하게 생각한 상황별 방법은 다음과 같습니다.

|로직복잡도|성능|트랜잭션 경합|방법|
|----|----|-----|-----|
|단순|중요|자주발생|쿼리에 로직 넣기|
|복잡|.|거의없음|쿼리에서 로직분리, 일반적인 select/update 사용|
|복잡|.|자주발생|쿼리에서 로적분리, select for update 사용|

<br>
<br>

# 참고
## lock  
https://www.mssqltips.com/sqlservertip/6290/sql-server-update-lock-and-updlock-table-hints/  
https://kuaaan.tistory.com/m/97

## 격리 수준  
https://luran.me/325

## hikari connection pool  
https://jaehun2841.github.io/2020/01/27/2020-01-27-hikaricp-maximum-pool-size-tuning/#hikari%EB%8B%98-connection-%EB%8B%A4-%EC%8D%BC%EC%96%B4%EC%9A%94