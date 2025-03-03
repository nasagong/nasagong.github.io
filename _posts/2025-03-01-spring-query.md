---
title: Memory of SpringBoot project
description: old memory about my first spring project
author: nasagong
date: 2025-03-01 11:10:00 +0800
categories: [Misc]
tags: [Web]
render_with_liquid: false
---

> 해당 포스팅은 1년 이상의 시간 차를 두고 있는 과거의 프로젝트 경험을 다룹니다.<br/>
> 스프링에 관한 또 다른 포스팅에 흥미가 있으시다면 [이 글](https://nasagong.github.io/posts/spring-versus-nest/)을 확인해주세요.

## 0.Intro

서버 개발에 처음 흥미를 가졌을 적에는 스프링부트를 참 많이 사용했지만 근래 간단한 서버를 구현할 일이 있을 땐 아무래도 최근 grpc에 흥미가 붙기도 했고 간단한 코드 정도는 플랫하게 처리하기 용이한 Go가 더 편하다고 생각되어 Go를 쓰고 있습니다.

 다만 모종의 이유로 스프링 사용 경험을 되새김질 할 필요가 생겨 가장 최근 스프링으로 작성했던 프로젝트 경험, 그 중에서도 데이터와 관련된 부분을 상기해보고자 합니다. 
## 1. SolveThis

바야흐로 군 전역 이후, 재정적으로 여유롭지 못했던 저는 영어과외에 손을 대고 말았고, 생각보다 많은 업무량에 당황했지만 적잖은 금액을 받는 입장에서 대충대충 하자는 마인드로 임하기도 곤란했기에 학생들의 학습 진척도를 최대한 쉽게 관리할 도구의 필요성을 느끼게 되었습니다. 
사족이지만 중/고등학생에게 영어를 가르칠 때는 사실 어휘가 80%라고 생각하는 주의입니다. 

따라서 자연스럽게 어휘 관리에 집중을 하게 되었고, 
매번 시험지를 만들고 인쇄하며 오답 여부를 학생별로 엑셀 파일에 저장하는 작업은 너무나도 귀찮게 느껴졌습니다. 
그렇게 복학을 앞두고 약 2주 가량을 할애해 SolveThis를 만들게 됩니다. 


제발 좀 풀라는 의미의 네이밍이었습니다.<br/> 지금에서야 과외도 그만뒀고 코드가 너무 부끄러운 나머지 레포도 지워버렸지만, 
나름대로 끙끙 앓으며 1인 개발을 마치고 프리티어에 배포파일을 던져 무려 유저 3명이 원활하게 이용할 수 있도록 구현해낸 애정어린 프로젝트이기도 합니다.


![alt text](/assets/img/image.png)
![alt text](/assets/img/image-1.png)

실 운영 코드는 거의 유실됐지만 사진으로나마 흔적이 조금 남아 있네요.

## 2. 개발

바로 본론을 적어보도록 하겠습니다. 코드 작성 과정이 유의미한 개발의 일부가 되기 위해 있어서 가장 중요한 것은 고민의 과정입니다. 누구나 할 수 있는 입 바른 소리 같지만 단순히 동작하는 코드를 작성하기만 하는 건 엔지니어링이라는 행위보다는 레고 블록 조립과 더 유사도가 높기 때문입니다.
저는 당시 작은 서비스를 개발하긴 했지만 나름의 고민의 과정이 있었습니다. 


### 2-1. 유저 테이블 정규화 문제

SolveThis에는 두 가지 유형의 유저가 존재합니다. 선생과 학생인데요, 당연히 선생은 저 하나기에 굳이 Teacher 필드를 추가할 필요 없이 개발자인 제가 직접 DB를 모니터링 하거나 로그를 확인하면 되는 문제긴 했습니다. 

다만 실용적인 목적 뿐만 아니라 학습의 목적도 있었기에 어느정도 확장성을 고려하여 관리자 페이지 구현을 염두에 둔 쉐도우 복싱 아닌 쉐도우 복싱을 하게 되었습니다. 실제로는 학교생활이 바빠 결국 직접 DB를 확인하는 방향으로 운영하긴 했지만요.

맨 처음 구상했던 테이블 구조는 아래와 같습니다. 가독성을 위해 최소한의 필드만 기재했습니다.

```sql
-- 유저 공통 정보 
CREATE TABLE users (
    user_id VARCHAR(20) PRIMARY KEY,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE teachers (
    user_id VARCHAR(20) PRIMARY KEY,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE TABLE students (
    user_id VARCHAR(20) PRIMARY KEY,
    teacher_id VARCHAR(20),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (teacher_id) REFERENCES teachers(user_id) ON DELETE SET NULL
);

```

깊은 생각을 하고 설계한 구조는 아니었습니다. 단순히 정규화 하는 편이 보기 좋다는 생각 하에 구상한 테이블이었고, 아무튼 역할을 분리하고 응집성을 높이는 게 좋은 패턴이라는 이상한 믿음이 마음 한 켠에 자리하던 시기였기 때문이기도 했습니다. 당연히 JOIN 비용이 발생하게 되겠지만 선생-학생 1:N 매핑에 있어 N이 그렇게까지 커질 가능성이 낮기도 했기에 크게 고려하진 않았습니다.

무언가 마음에 안 드는 부분은 다른 곳에서 발견됐습니다. 실제 repository 코드를 작성할 때였습니다.

```java
// 간소화한 예시 코드
public interface UserRepository extends JpaRepository<User, Long> {
    User findByUserId(String userId);
}

public interface TeacherRepository extends UserRepository {
    List<User> findAllByUserRole(UserRole userRole);
}

public interface StudentRepository extends UserRepository {
    List<User> findAllByUserRole(UserRole userRole);
    List<User> findAllByTeacher_UserId(String teacherId);
}
```

작성 직후엔 참 괜찮은 구조로 보였습니다만, 상속을 쓰는 게 영 맘에 걸렸습니다. 문제점이라고 하면 당연히 첫 번째로는 코드 중복이 발생했습니다. ```UserRole```로 유저를 긁어오는 메서드는 불필요한 상속으로 인해 두 레포지토리에 동시에 존재하게 됐죠.

타입 관점으로 볼 때 상당히 나사빠진 구조이기도 했습니다. ```UserRepositroy```는 당연히 User 엔티티를 반환하게 되겠지만 자식 레포지토리는 Teacher나 Student를 다뤄야 하기 때문입니다.
매퍼를 사용해서 어떻게 운영하는 건 가능하겠지만 엔티티 수정이 빈번한 프로젝트였던지라 코드 작성 측면에서 부담이 상당했습니다. 

당시에는 머리를 비우고 코드를 적던 중 JPA에서 타입 에러를 뱉어준 덕분에 다행히도 무언가 잘못됐음을 깨닫게 된 기억이 있습니다. 

#### 개선

당시에 도달한 결론은 그냥 DB레벨에서 테이블을 정규화 해버리는 게 최선이라는 것이었습니다.
프로젝트 규모상 사실 그렇게 높은 수준의 정규화가 필요하지도 않았을 뿐더러 어플리케이션 레벨에서 복잡도만 증가시키고 이점도 거의 없었기 때문입니다.

최종적으로는 아래와 같은 테이블을 구성했습니다.

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(20) NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    user_role VARCHAR(10) NOT NULL,
    ...
    teacher_user_id VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
);
```
```will_take_exam``` 이나 ```test_mode``` 같은 학생에게만 사용되는 필드도 존재했지만 nullable하게 선언해두고 실제 프론트에선 DTO를 통해 유저 역할을 확인하고
그에 따른 페이지를 보여주도록 설계 했습니다. 실제 서버 단계에서도 그냥 각각의 서비스 레이어에서 UserRepo를 사용하면 되는 일이라 개발하는 입장에서도 더 편했네요.

너무 길게 적은 것 같지만 그냥 단순하게 별 거 아니면 비정규화 하는 게 좋다는 정도의 깨달음을 얻은 경험이었습니다.


### 2-2. 시험 결과 전송 문제

제 서비스에서 학생들은 단어 시험을 보게 되면 아래와 같은 정보를 DB에 업데이트 하거나 추가해야 합니다.

```Text
1. 특정 단어를 틀렸는지 여부 업데이트
2. 시험 결과를 전송
```

전반적인 학생<->단어<->시험지 관계를 먼저 설명드리는 편이 좋을 것 같습니다. 
조금 길지만 원활한 이해와 코드 노출 비율을 높이기 위해 당시의 제 Entity와 DTO 코드를 아래에 첨부 후 문제 상황과 해결 과정을 적어보겠습니다.

이번엔 DDL 대신 Entity를 보며 확인해보겠습니다.

```java
public class Word {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @ManyToOne
    @JoinColumn(name = "student_user_id", referencedColumnName = "user_id", nullable = false)
    private User student;

    @Column(name = "english", length = 50, nullable = false)
    private String english;

    @Column(name = "korean", length = 50, nullable = false)
    private String korean;

    @Column(name = "was_took")
    private boolean wasTook = false;

    @Column(name = "recently_wrong")
    private boolean recentlyWrong = false;

    @Column(name = "number_of_wrong", nullable = false)
    private int numberOfWrong = 0;

    @Column(name = "created_at", updatable = false)
    @CreationTimestamp
    private LocalDate createdAt;
}
```
우선 각 학생별로 여러개의 단어 목록을 갖게 됩니다. 이 단어는 관리자가 자신의 학생에게 직접 등록하게 되며, 교사는 학생의 특정 단어
누적 오답 횟수나 최근 오답 여부를 확인할 수 있게 됩니다. 이 정보는 앞서 ```User``` 테이블에 있었던 ```test_mode``` 에 따라 애플리케이션이 다른 로직으로 시험지를 서빙하도록 만듭니다.

지금 와서 이 때 작성한 코드를 보자자니 오히려 정규화는 엄한 User 테이블이 아닌 Word 테이블에 적용해야 했던 게 아닌가 싶네요.
지난일이니 우선 각설하고 이번엔 Test 엔티티를 보여드리겠습니다. 

```java
public class Test {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "student_user_id", referencedColumnName = "user_id", nullable = false)
    private User student;

    @Column(name = "created_at", updatable = false)
    @CreationTimestamp
    private LocalDate createdAt;

    public Test(User student) {
        this.student = student;
    }
}
```

```Test```는 크게 특별한 테이블은 아니었습니다. 학생이 '시험 응시' 버튼을 누르게 되면 해당 학생을 포인팅하는 Test 인스턴스가 생기게 됩니다.
후술할 ```TestResult``` 와 1:N 매핑되어 시험지 정보를 만드는 역할을 하게 됩니다.

```java 
public class TestResult {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "is_correct")
    private boolean correct;

    @ManyToOne
    @JoinColumn(name = "test_id", referencedColumnName = "id", nullable = false)
    private Test test;

    @ManyToOne
    @JoinColumn(name = "word_id", referencedColumnName = "id", nullable = false)
    private Word word;
}
```

Test result 까지 읽어보게 되시면 당시 제가 어떤 구조를 구현하려고 했는지 어렴풋이 눈치채실 수 있으실 것 같습니다.

![alt text](/assets/img/erd.png)

ERD로는 이렇게 표현할 수 있겠네요. Word 테이블의 many to one 설계가 조금 결함있어 보이지만 우선 동작 자체에는 큰 문제가 없어보입니다.
정리하자면 아래의 흐름이겠네요.

---
학생(```User```)이 시험(```Test```)을 보면
- 여러 단어(```Word```)가 출제되고
- 시험 결과(```TestResult```)에 각 단어를 맞췄는지 저장된다.

---

실제로 학생들은 본인들의 단어 DB를 가지게 됐고, user 테이블의 필드 중 하나인 test_mode를 통해 오답노트 모드, 신규 단어 모드 등 여러 모드로 시험을 응시할 수 있었습니다. 구현을 성공했는가의 관점으로만 본다면 문제 있는 구조는 아니었습니다. 다만 실 운영 중 한 가지 문제가 발생합니다. 우선 당시에 ```시험 결과를 등록하는``` 역할을 했던 Service를 레이어의 메서드를 하나 보여드리고자 합니다.

```java
public void registerResult(TestResultRequestDTO testResultRequestDTO, HttpSession session) {
    if (session == null) {
        throw new SessionNotFoundException("Session is null");
    }
    
    String userId = (String) session.getAttribute("userId");
    Long testId = (Long) session.getAttribute("testId");
    if (userId == null || testId == null) {
        throw new IllegalArgumentException("Session attributes missing");
    }
    
    User student = userRepository.findUserByUserId(userId)
        .orElseThrow(() -> new UserNotFoundException("No user matched this ID"));
    Word word = wordRepository.findByStudentAndEnglish(student, testResultRequestDTO.getEnglish())
        .orElseThrow(() -> new WordsNotFoundException("No word matched this info"));
    Test test = testRepository.findById(testId)
        .orElseThrow(() -> new TestNotFoundException("Test not found"));
    
    testResultRepository.save(new TestResult(testResultRequestDTO.isCorrect(), test, word));
    
    session.removeAttribute("testId");
}
```

사용된 DTO는 아래와 같습니다.

```java
public record TestResultResponseDTO(
    @NotNull String korean,
    @NotNull String english,
    boolean correct,
    @NotNull Long testId
) {
    public static TestResultResponseDTO of(String korean, String english, boolean correct, Long testId) {
        return new TestResultResponseDTO(korean, english, correct, testId);
    }
    
    public static TestResultResponseDTO from(com.nasagong.solvethis.entities.TestResult testResult) {
        return new TestResultResponseDTO(
            testResult.getWord().getKorean(), 
            testResult.getWord().getEnglish(),  
            testResult.isCorrect(),
            testResult.getTest().getId()
        );
    }
}
```

학습 목적으로 노션에 기록된 몇 안 되는 실 운영 코드 중 하나인데요, 기록 될 가치가 있을 만큼 조금 큰 하자가 존재했습니다.

예컨대 제 학생들은 시험지당 평균 4-50개의 단어가 포함 돼있었는데요. 말인 즉슨 마지막 정답을 입력한 시점에서 ```INSERT```를 매번 별개의 트랜잭션으로 따박따박 50개를 날려야 한다는 말입니다.

당시에는 제가 DB에 대한 감각도 영 없었고, 아무튼 빨리 배포해서 학생별로 매번 종이 시험지를 뽑는 귀찮은 작업을 간소화 하고 싶다는 욕망이 컸던지라 매우 무겁고 느린 기능이 될 거라 예측은 했지만 아무렴 어떻게든 굴러가겠지라는 마인드 하에 저 상태로 배포를 진행했습니다.

시험이라는 서비스의 특성 상 유저는 빠르게 결과를 확인하고 싶어합니다. 단어를 하나도 공부해오지 않은 학생들은 보기 싫어했지만 이건 별개의 이야기고요.

다만 저렇게 몇 십개의 트랜잭션을 와르르 쏟아내는 메서드는 생각보다 아주 느리게 동작했고, 시험을 마치고 학생에게 마이 페이지에 바로 들어가보라고 했지만 아직 시험 결과가 등록되지 않은 경우가 많았습니다. 실제로 사용하기엔 너무나도 느렸던 겁니다.

#### 개선

저는 문제를 어떻게 처리해야 했을까요? 가장 처음 떠오른 건 컨트롤러단에서 DTO를 리스트 형태로 전달 받아 INSERT 연산을 벌크로 수행하는 것이었습니다. 아래처럼요.

```java
public void registerResults(List<TestResultRequestDTO> testResults, HttpSession session) {
    if (session == null) {
        throw new SessionNotFoundException("Session is null");
    }

    String userId = (String) session.getAttribute("userId");
    Long testId = (Long) session.getAttribute("testId");
    if (userId == null || testId == null) {
        throw new IllegalArgumentException("Session attributes missing");
    }

    User student = userRepository.findUserByUserId(userId)
        .orElseThrow(() -> new UserNotFoundException("No user matched this ID"));
    Test test = testRepository.findById(testId)
        .orElseThrow(() -> new TestNotFoundException("Test not found"));

    List<TestResult> results = testResults.stream()
        .map(dto -> {
            Word word = wordRepository.findByStudentAndEnglish(student, dto.getEnglish())
                .orElseThrow(() -> new WordsNotFoundException("No word matched this info"));
            return new TestResult(dto.isCorrect(), test, word);
        })
        .toList();

    testResultRepository.saveAll(results);

    session.removeAttribute("testId");
}
```
JPA를 사용하면 벌크 연산을 정말 쉽게 사용할 수 있다니 참 놀라웠는데요, 이 때나 지금이나 저는 JPA를 잘 알지 못합니다. 공부에는 항상 Why가 있어야 한다는 주의고 저에겐 스프링이나 JPA의 내부를 공부할 동기가 없던 상태였기에 영속성 컨텍스트의 러프한 개념과 간단한 사용법 숙지 정도에 그친 상태입니다.

여하간, 저는 코드를 작성하며 한 가지 의문을 갖게 됩니다. ```'저 SaveAll로 추상화 된 DB연산이 정말 벌크 연산을 하고 있는 걸까?'``` 하는 의문이요. 그도 그럴 게 저는 JPA를 잘 알지 못하니까요. 그랬기에 이게 정말 개선안이 될 수 있을지 알아볼 필요가 있었습니다. 

결론적으로 매 INSERT마다 persist 가 발생하기에 기술적으로 벌크 연산이라고 보기엔 무리가 있었습니다. 지금 발견한 거지만 ```@Transactional``` 도 없어서 그냥 개별 쿼리 날리는 거랑 거의 같은 코드였네요. 

당시 내린 결정은 그냥 네이티브로 Batch Insert하는 쿼리를 작성하자는 거였습니다. JPA를 요리조리 조작해보면 성능이 나아질 수도 있었겠지만 그 때는 이게 가장 빠른 해결법이었고 지금도 다시 최적화 하라면 이렇게 할 것 같네요. 최종 수정 코드는 아래와 같습니다.

```java
public void registerResults(List<TestResultRequestDTO> testResults, HttpSession session) {
    if (session == null) {
        throw new SessionNotFoundException("Session is null");
    }

    String userId = (String) session.getAttribute("userId");
    Long testId = (Long) session.getAttribute("testId");
    if (userId == null || testId == null) {
        throw new IllegalArgumentException("Session attributes missing");
    }

    User student = userRepository.findUserByUserId(userId)
        .orElseThrow(() -> new UserNotFoundException("No user matched this ID"));
    Test test = testRepository.findById(testId)
        .orElseThrow(() -> new TestNotFoundException("Test not found"));

    List<Long> wordIds = testResults.stream()
        .map(dto -> wordRepository.findByStudentAndEnglish(student, dto.getEnglish())
            .orElseThrow(() -> new WordsNotFoundException("No word matched this info")))
        .map(Word::getId)
        .toList();

    List<Boolean> corrects = testResults.stream()
        .map(TestResultRequestDTO::isCorrect)
        .toList();

    testResultRepository.bulkInsertTestResults(corrects, testId, wordIds);

    session.removeAttribute("testId");
}
```
```java
@Modifying
@Query(value = "INSERT INTO test_result (is_correct, test_id, word_id) VALUES (:correct, :testId, :wordId)", nativeQuery = true)
void bulkInsertTestResults(@Param("correct") List<Boolean> corrects, 
                           @Param("testId") Long testId, 
                           @Param("wordId") List<Long> wordIds);
```

정확한 벤치마크를 남기진 않았지만 적어도 부랴부랴 수정하고 재배포 한 이후에는 결과가 바로 뜨지 않아
'쌤이 만든거잖아요' 같은 표정을 짓는 학생 앞에서 난처한 표정을 지을 일은 없었던 걸로 기억하네요.


## 3. 마치며

스프링 사용 경험이 마냥 풍부한 사람은 아닌지라 유의미한 스프링 사용 경험은 이 정도로 정리할 수 있을 것 같습니다. 
아무래도 자바 관련 포스팅 계획은 없던 블로그였는데, 동아리용 포트폴리오라는 이유로 적게 되다보니 그 부분을 의식하면서 적게 된 면이 없잖아 있는 것 같네요. 
아래는 그냥 자기 변호의 성격이 짙은 여담입니다.

---

저는 예나 지금이나 스프링은 강력하고 좋은 도구라고 생각합니다. 다만 "N학년까지 스프링을 공부하지 않으면 좋은 직장을 가질 수 없다" 같은 말은 굉장히 싫어합니다. 
빠르고 늦음의 문제가 아니라 도구에 종속된 공부를 하는 게 과연 옳은가 하는 다소 회의적인 저의 관점 때문입니다. 차라리 "객체지향 패러다임에 익숙해지는 편이 좋을 거다." 같은 말이었으면 고개가 끄덕여지기라도 했을 겁니다. 

인프런 누구의 스프링 강의를 듣고, 동아리에 들어가 웹 서비스 프로젝트를 하고, 백준 몇 문제를 풀고 Solved.ac 플레티넘 티어를 달성하면 취업을 할 수 있다는 식의 도시전설 같은 팁들이 구전되는 한국 IT대학가의 현실에도 염증을 느낍니다.

본문에 언급한 것처럼 저는 자신이 하는 모든 일의 'Why'를 알아야 한다는 주의입니다. 지금의 저는 스프링보다는 Go를 사용하는 편입니다. 하지만 이는 제가 스프링을 싫어한다거나 스프링 학습자에 대한 무의미한 선민의식을 갖고 있어서는 아닙니다. 저는 지금 Go가 재밌고, Go를 통해 작성된 프로젝트를 읽어보는 것도 재밌고, 실제 SWE 필드에서 Go가 이점을 갖는 부분이 무엇인지 직접 사용해 보며 익히는 것에 큰 재미를 느끼고 있습니다. 
무엇보다 Go는 개인적 흥미 외에도 혼자 간단하게 작업해야 하는 일이 많은 제 상황에서 보일러플레이트를 최소화하여 플랫하게 작성하기 좋고 빠르게 빌드할 수 있다는 Why를 충족합니다.

그렇다면 제가 지금 앞 뒤 안 가리고 스프링을 공부해야 할 Why는 무엇일까요? 취업을 위해서? 제가 지금 많은 이들이 으레 그러는 것처럼 스프링 강의를 듣고 수많은 어노테이션들의 기능을 공부하고, JPA에 딥다이브 한다면 그게 정말 유의미한 공부가 될 수 있을까요? 토이프로젝트라도 만들며 그런 기능들을 덕지덕지 붙여보는 것도 도구 사용자로서 숙련도의 측면에선 유의미한 학습이 될 수 있을 것 같지만, 정말 엔지니어로서의 내공을 쌓을 수 있을 것 같다고 생각하진 않습니다. 

저는 영어영문학과 2학년을 마치고 다시 새로운 학교 소프트웨어 학과에 입학하여, 1학년을 마친 후 소프트웨어 엔지니어링에 대해 일자무식한 상태에서 SW개발 관련 부대에 입대했습니다. 이 고마운 기회 덕에 지금까지도 사회에서 쉽게 만나볼 수 없는 급의 개발자들을 다수 만날 수 있었고, 많은 조언을 듣게 됐습니다. 이러한 인물들의 가장 인상깊었던 특징은 휘황찬란한 기술 스택 목록도, 묘기에 가까운 코딩 실력도 아니었습니다. 항상 기술 근본에 대한 흥미를 갖고 있었으며 철저히 흥미에 기반한 코어 지식 학습을 이어 간다는 것이었습니다. 

현재 흔히들 말하는 한국 N대 IT 서비스 기업에 취업해 스프링으로 업무를 보고 있는 분들도 다수 계시는데, 이들 중 스프링을 메인으로 공부하신 분도, 저에게 스프링을 강조하신 분도 없었습니다. 

그리고 저는 그들의 성과는 그들이 JPA가 아닌 ORM과 데이터베이스를 공부했고, 스프링이 아닌 서버 엔지니어링을 공부했기 때문이라고 생각합니다. 저 또한 이러한 부분에 영향을 받아 학습 방향을 세우고 있습니다. 이러한 방향으로 학습을 이어간다면 제 스프링 숙련도는 조금 낮을지라도 조금의 적응기만 주어진다면 어떤 도구를 사용해도 무의미한 학습을 이어 온 이들보다는 엔지니어링이라는 측면에서 유의미한 코드를 작성할 수 있을 거라는 신념을 갖고 있습니다.

이러한 고집에 가까운 신념을 가진 제게도 한 가지 갈증을 느끼는 부분이 있습니다. 바로 실제 팀 단위의 협업 경험입니다. 기술적 성장도 중요하지만 결국 현대 소프트웨어 개발은 팀워크의 산물이기 때문입니다.<br/>

특히 실제 서비스를 운영하는 환경에서는 코드 리뷰, 아키텍처 설계 논의 등 혼자서는 경험하기 어려운 다양한 협업 상황이 발생합니다. 이러한 실전 경험을 통해 제가 쌓아온 기술적 지식이 실제 비즈니스 가치 창출에 어떻게 기여할 수 있는지 올해가 가기 전에 충실한 학생의 자세로 배워보고 싶습니다.<br/> 

또한 가능하다면 제가 지금까지 쌓아 왔고, 또 쉽게 접해보기 힘든 이들에게서 얻은 인사이트를 다른 이들과 나눠볼 수 있는 경험도 가져보고 싶습니다. 제가 그리 잘난 사람은 아니지만, 학습에 어려움을 겪던 시기에 시니어 학습자들에게 수혜를 받은 것처럼 다른 이들에게도 나눠주어야 한다는 생각이 있기 때문입니다.

긴 글 읽어주셔서 감사합니다.