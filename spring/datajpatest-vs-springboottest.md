### @DataJpaTest vs @SpringBootTest

---

### @SpringBootTest

 - 전체 어플리케이션의 컨텍스트를 로드하여 테스트 하는데 사용한다.
 - 주로 통합테스트에서 사용하며 실제 어플리케이션과 유사한 테스트 환경을 제공한다.

```
@SpringBootTest
class MemberRepositoryTest {

    @Autowired
    private MemberRepository memberRepository;
    
    @DisplayName("멤버 가입")
    @Test
    public void register() {
        // given
        Member member = new Member();

        //when
        Long id = memberRepository.save(member);

        // then
        assertThat(id).isEqualTo(1L);
    }
```

### @DataJpaTest

- JPA 관련 설정만을 로드하여 테스트를 수행할 수 있다.
- in-memory DB를 사용한다.

```
@DataJpaTest
class MemberRepositoryTest {

    @Autowired
    private MemberRepository memberRepository;
    
    @DisplayName("멤버 가입")
    @Test
    public void register() {
        // given
        Member member = new Member();

        //when
        Long id = memberRepository.save(member);

        // then
        assertThat(id).isEqualTo(1L);
    }
```

### 비교 및 성능

---

위 코드를 실행해보면 두 코드의 실행 속도 차이는 아래와 같이 난다.

 - @SpringBootTest : 417ms
 - @DataJpaTest : 132ms

단순한 JPA에 관한 테스트만 해볼때는 @DataJpaTest가 더 빠르다.  

하지만 @DataJpaTest 사용시 QueryDSL, 사용자 설정 @Bean등을 사용하게 되면 해당 되는 설정 빈을 적용시켜야 하고  
이미 @SpringBootTest를 사용하고 있는데 @DataJpaTest 테스트 생성시 스프링 컨테이너를 추가로 한번 더 실행 시켜주어야 한다.  
위와 같은 부분을 고려하여 더 적절한 테스트를 사용하면 된다.

