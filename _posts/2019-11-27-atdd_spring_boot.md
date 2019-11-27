---
layout: post
title: "ATDD with Spring Boot"
author: "Boorownie"
tags: ["ATDD", "Spring", "Spring Boot"]
---

<!-- ![](https://images.squarespace-cdn.com/content/v1/5bcdd76a81551217e12a7a2b/1541601663984-RX2K7WRMXZTYY4PGU9AE/ke17ZwdGBToddI8pDm48kOyctPanBqSdf7WQMpY1FsRZw-zPPgdn4jUwVcJE1ZvWQUxwkmyExglNqGp0IvTJZUJFbgE-7XRK3dMEBRBhUpyD4IQ_uEhoqbBUjTJFcqKvko9JlUzuVmtjr1UPhOA5qkTLSJODyitRxw8OQt1oetw/Hindsight_Outside+in+Development.png){: width="60%" height="60%"} -->

TDD를 배우면서 도메인에 먼저 집중하여 도메인 기능의 테스트 코드를 구현한 후 기능을 구현하는 방법으로 학습합니다.
한 번에 한 가지를 집중하고 각 도메인들은 애플리케이션이 구성되기 전에 작성됩니다.
하지만 ATDD에서 인수 테스트는 큰 그림을 그려 구조를 형성한 다음 구체적인 부분을 구현합니다.
이러한 접근 방법을 전자는 `Inside out`이라 하고 후자는 `Outside in`이라고 합니다.(TDD가 Inside out이고 ATDD가 Outside in이라는 뜻은 아닙니다.)
TDD를 이미 접한 후 ATDD를 배우는 사람은 각 접근 방식의 차이점을 인지하지 못하여 학습에 어려움을 겪기도 합니다.
Inside out 접근 방식은 도메인 지식이 있거나 요구 사항이 단순한 경우 적합하고
Outside in 접근 방식은 도메인 지식이 없거나 요구 사항 복잡도가 높은 경우 적합하다 할 수 있습니다.
각 방식의 장단점은 [TDD - From the Inside Out or the Outside In?](https://8thlight.com/blog/georgina-mcfadyen/2016/06/27/inside-out-tdd-vs-outside-in.html) 에서 확인할 수 있습니다.
이번 포스팅에서는 간단한 예제를 통해 Outside in방식의 ATDD 개발 방법에 대해서 알아보겠습니다.

--
# ATDD 개발 프로세스
![](https://user-images.githubusercontent.com/4353846/67550967-c108ea80-f742-11e9-9f06-1b39fe4ba0f8.png){: width="60%" height="60%"}
1. 인수 조건을 바탕으로 인수 테스트 작성한다.
2. 인수 테스트를 바탕으로 문서화를 하여 인터페이스를 공유한다.
3. 인수 테스트를 동작시키는 기능을 하나씩 구현한다.

--
# [예제] 서핑 용품 대여 신청 기능 개발
> 상황에 따라 테스트 방법이 달라질 수 있고 아래의 예시는 그 방법 중 하나입니다.

서핑 용품 대여 신청 기능을 구현하는 과정을 ATDD 개발 프로세스로 단계별로 알아보겠습니다.
예제에서는 WebTestClient로 Happy case에 대한 인수 테스트를 만들고
예외적인 상황이나 에러 처리 테스트는 Controller Test의 MockMvc객체를 이용할 예정입니다.
(추후 Github 공유 예정)

## 인수 조건
> 대여를 희망하는 용품을 희망하는 날짜에 신청 할 수 있어야 한다.

## 인수 테스트 작성
이전 포스팅에서 언급되었던 것 처럼, 비즈니스 규칙(Business Rule)은 기술의 구현이나 작업흐름만큼 변경이 많지 않습니다.
인수 테스트를 작성할 때 비즈니스 규칙을 고려하여 테스트 시나리오를 만들 경우 변경사항에 비교적으로 유연하게 대처할 수 있습니다.
그런 관점에서 볼 때, UI 인수 테스트를 작성하는 것 보다 API 인수 테스트를 하는게 시나리오의 품질관리에 유리합니다.
따라서 UI 테스트 보다는 API 테스트를 통해 인수 테스트를 작성하겠습니다.

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
public class RentalAcceptanceTest {
    public static final String LOCATION = "Location";

    @Autowired
    private WebTestClient webTestClient;

    @Test
    public void createRental() {
        String inputJson = "{\"dateTime\":\"" + "2019120112" + "\", " +
                "\"itemId\":\"" + "110920" + "\", " +
                "\"itemType\":\"" + "14" + "\"}";

        this.webTestClient.post().uri("/rentals")
                .contentType(MediaType.APPLICATION_JSON) // 요청으로 보내는 데이터 유형 명시
                .accept(MediaType.APPLICATION_JSON) // 응답으로 받고 싶은 데이터 유형 명시
                .body(Mono.just(inputJson), String.class)
                .exchange()
                .expectStatus().isCreated()
                .expectHeader().contentType(MediaType.APPLICATION_JSON)
                .expectHeader().valueMatches(LOCATION, "\\/rentals\\/\\d")
                .expectBody()
                .jsonPath("$").isNotEmpty()
                .jsonPath("$.id").isNotEmpty()
                .jsonPath("$.status").isEqualTo("READY");
    }
}
```

--
## 문서화
Mock 서버를 먼저 개발하여 Mock 데이터를 제공하면 백엔드와 프론트엔드 개발이 병렬적으로 진행이 가능합니다.
이 때, 인수 테스트를 기반으로 문서화까지 해서 제공한다면 더 효과적인 커뮤니케이션이 가능합니다.
따라서 [Spring Rest Docs](https://spring.io/projects/spring-restdocs)를 이용하여 문서를 만들 예정입니다.
Rest Docs를 이용하여 문서를 만들기 위해서는 테스트가 성공되어야 합니다.
따라서 인수 테스트를 성공시키기 위한 Mock 서버를 구현하고 Request / Response DTO를 선언합니다.
이 부분은 ATDD 프로세스에 위배될 수 있지만 업무 효율을 고려하여 유리하다고 판단하여 이렇게 결정하였습니다.

#### Mock 서버 구현
```java
@RestController
public class RentalController {
    @PostMapping("/rentals")
    public ResponseEntity createArticles(@RequestBody RentalRequestDto requestDto) {
        return ResponseEntity.created(URI.create("/rentals/" + 1)).body(new RentalResponseDto(1, "READY"));
    }
}
```

#### Rest Docs 문서 설정
Rest Docs를 통해 문서화를 할 경우 다음과 같은 객체를 사용하는 것을 권장합니다, MockMvc / WebTestClient / RestAssured
각 테스트 객체의 특징을 고려하여 선택한 후 문서화에 사용하면 좋을 것 같습니다.
상황에 따라서 여러 종류의 테스트 객체를 사용해야 할 수 도 있습니다.

참고 문서
- [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/2.0.4.RELEASE/reference/html5/)
- [Spring Rest Docs 적용](http://woowabros.github.io/experience/2018/12/28/spring-rest-docs.html)

--
## TDD 기능 구현
#### Controller TDD
Controller의 메서드에서 구현할 내용에 대해서 given을 통해 응답을 정의합니다.
작업 순서는 Controller Test를 작성하면서 Service 클래스를 생성하여 빈 껍데기 메서드만 만든 뒤 Controller 프로덕션 코드를 작성합니다.
Controller 테스트에서 MockMvc 객체를 사용한 이유는 Interceptor나 ArgumentResolver와 같이 Controller 이전에 동작하는 기능 등 전체 로직을 통합 테스트하기 위해서 입니다.
만약 Controller나 Interceptor, ArgumentResolver 와 같은 객체들을 단위테스트 할 경우, 해당 객체를 생성하여 메서드를 바로 호출하는 테스트를 작성해도 무관합니다.
RentalService의 createRental 메서드에서는 Item조회를 하고 Item이 유효한지 확인한 후 Rental을 만들어 저장하는 로직을 구현할 예정입니다.

아래 코드의 개발 순서는

```java
@WebMvcTest(controllers = RentalController.class)
@AutoConfigureMockMvc
public class RentalControllerTest {
    @Autowired
    private MockMvc mockMvc;

    // 3 - RentalService 객체를 만들고 createRental메서드 생성
    @MockBean
    private RentalService rentalService;

    @DisplayName("대여 신청을 한다.")
    @Test
    public void createRental() throws Exception {

        // 5 - ItemTest와 RentalTest를 생성하고 생성하는 로직을 구현
        Item item = new Item(110920, "READY");
        Rental rental = new Rental(1, "20191127", item, "READY");

        // 2 - item 존재 여부확인 + rental 저장 로직을 service 객체로 분리하기로 결정
        given(rentalService.createRental(any())).willReturn(rental);

        // 1
        String inputJson = "{\"date\":\"" + "20191127" + "\", " +
                "\"itemId\":\"" + "110920" + "\"}";
        mockMvc.perform(post("/rentals")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON)
                .content(inputJson))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.id").isNotEmpty())
                .andExpect(jsonPath("$.date").value("20191127"))
                .andExpect(jsonPath("$.itemId").value("110920"))
                .andExpect(jsonPath("$.status").value("READY"))
                .andDo(print());
    }
}

---

// 4
@Service
public class RentalService {
    public Rental createRental(RentalRequestDto requestDto) {
        return null;
    }
}

---

// 6
@RestController
public class RentalController {
    private RentalService rentalService;

    public RentalController(RentalService rentalService) {
        this.rentalService = rentalService;
    }

    @PostMapping("/rentals")
    public ResponseEntity createArticles(@RequestBody RentalRequestDto requestDto) {
        Rental persistRental = rentalService.createRental(requestDto);
        return ResponseEntity.created(URI.create("/rentals/" + persistRental.getId())).body(persistRental);
    }
}

```

#### Service TDD
Controller Test에서 given으로 정의한 Service 메서드를 구현합니다. Controller와 마찬가지로 테스트를 먼저 작성한 뒤 프로덕션 코드를 작성합니다.
Service 레이어를 테스트 할 때 반드시 Repository를 MockBean으로 간주하여 단위테스트를 해야하는 건 아니며 트랙젝션 등 처리로직 확인을 위해서는 통합테스트로 진행해도 무관합니다.

```java
@SpringBootTest(classes = RentalService.class)
public class RentalServiceTest {
    private RentalService rentalService;

    // 4
    @MockBean
    private RentalRepository rentalRepository;
    @MockBean
    private ItemRepository itemRepository;

    @BeforeEach
    void setUp() {
        rentalService = new RentalService(rentalRepository, itemRepository);
    }

    @Test
    void createRental() {
        Item item = new Item(1, "READY");

        // 3
        given(itemRepository.findById(anyInt())).willReturn(item);

        RentalRequestDto rentalRequestDto = new RentalRequestDto("20191127", 1123);

        // 2
        given(rentalRepository.save(any())).willReturn(new Rental(1, rentalRequestDto.getDate(), item, "READY"));

        // 1
        Rental persistRental = rentalService.createRental(rentalRequestDto);

        assertThat(persistRental).isNotNull();
    }
}

---

// 5
@Service
public class RentalService {
    private RentalRepository rentalRepository;
    private ItemRepository itemRepository;

    public RentalService(RentalRepository rentalRepository, ItemRepository itemRepository) {
        this.rentalRepository = rentalRepository;
        this.itemRepository = itemRepository;
    }

    public Rental createRental(RentalRequestDto requestDto) {
        Item persistItem = itemRepository.findById(requestDto.getItemId());
        Rental persistRental = new Rental(requestDto.getDate(), persistItem, "READY");
        return rentalRepository.save(persistRental);
    }
}


```

#### Repository TDD
```java
@SpringBootTest(classes = RentalRepository.class)
public class RentalRepositoryTest {
    private RentalRepository rentalRepository;

    @BeforeEach
    void setUp() {
        rentalRepository = new RentalRepository();
    }

    @Test
    void save() {
        // 2
        Item item = new Item(110920, "READY");
        Rental rental = new Rental("20191127", item, "READY");

        // 1
        Rental persistRental = rentalRepository.save(rental);

        assertThat(persistRental.getId()).isEqualTo(1);
    }
}

---

@Repository
public class RentalRepository {
    private List<Rental> rentals = new ArrayList<>();

    public Rental save(Rental rental) {
        Rental persistRental = new Rental(rentals.size() + 1, rental.getDate(), rental.getItem(), rental.getStatus());
        rentals.add(persistRental);
        return persistRental;
    }
}

```

```java
// ItemRepository도 같은 방식으로 진행

```

#### Controller TDD - 예외 케이스
정상적인 로직에 대한 테스트 작성이 끝나면 Side Case에 대한 테스트를 작성해야합니다.
Happy Case에 대한 테스트를 작성하는 것 처럼 테스트 케이스에 대한 메서드를 만들어주고 given 조건과 기대하는 then 조건을 설정합니다.

참고로 **도메인 클래스 작성은 어느 레이어에서 해도 상관이 없다**고 생각합니다.
Outside in 이라고 해서 반드시 그 방향대로(ex. Controller -> Service -> Repository 순) 개발을 해야하는 것이 아니라
테스트를 구현하고 로직을 구현하다가 **필요한 로직이 생기면 그 때** 해당 부분을 구현하기 위해 테스트 코드를 구현하고 로직을 구현하였습니다.

```java
@DisplayName("대여 신청 시 아이템이 이미 대여중인 경우 400에러를 응답한다.")
@Test
public void createRentalWithInvalidItemId() throws Exception {
    given(rentalService.createRental(any())).willThrow(new InvalidItemException());

    String inputJson = "{\"date\":\"" + "20191127" + "\", " +
            "\"itemId\":\"" + "111" + "\"}";

    mockMvc.perform(post("/rentals")
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .content(inputJson))
            .andExpect(status().isBadRequest())
            .andDo(print());
}

---

@ResponseStatus(HttpStatus.BAD_REQUEST)
public class InvalidItemException extends RuntimeException{
}

---

public class RentalTest {
    @Test()
    void checkItem() {
        Assertions.assertThrows(AlreadyRentItemException.class, () -> {
            new Rental("20191127", new Item(111, "RENT"), "READY");
        });
    }
}

---

public class Rental {
    ...

    public Rental(String date, Item item, String status) {
        item.checkStatus();
        this.date = date;
        this.item = item;
        this.status = status;
    }

    ...
}

---

public class ItemTest {
    @Test
    void checkStatus() {
        Item item = new Item(111, "RENT");
        Assertions.assertThrows(AlreadyRentItemException.class, () -> {
            item.checkStatus();
        });
    }
}


public class Item {
    ...

    public void checkStatus() {
        if (status.equals("RENT")) {
            throw new AlreadyRentItemException();
        }
    }
}

```

#### 리팩터링
추후 업로드 예정;;;

--
## 인수 테스트 동작 확인
TDD 프로세스를 통해 기능 구현이 끝나면 최초 작성한 인수 테스트가 정상적으로 동작하는지 확인을 합니다.
최초에 인수 테스트가 성공한 것은 Mock 서버가 동작하고 있었기 때문인데 이 기능을 실제 기능 구현으로 대체하게 되었습니다.
