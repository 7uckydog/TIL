# Harness Engineering with Claude Code

> 롯데카드용 어드민 백오피스를 Claude Code를 통한 바이브 코딩으로 진행하면서 정리한, "AI에게 일을 잘 시키기 위한 틀 만들기"에 대한 기록.

## 배경

신규 백오피스 개발 진행을 Claude Code 바이브 코딩으로 진행했다. 처음에는 단순히 "이 기능 구현해줘", "이 페이지 만들어줘" 같은 요청만으로도 결과가 나오긴 했지만, 작업이 길어질수록 같은 컨벤션을 반복해서 설명하거나, 이미 정해둔 구조를 까먹은 AI에게 다시 안내해야 하는 일이 많아졌다.

그러다 보니 자연스럽게 **"AI가 매번 같은 맥락에서 작업을 시작하도록 하는 장치"**가 필요해졌고, 그게 바로 하네스 엔지니어링이었다.

## 하네스 엔지니어링이란

하네스(harness)는 원래 "마구·받침대·고정 장치"라는 뜻이다. 개발 맥락에서는 **AI에게 작업을 시키기 전에 작업의 맥락·제약·규칙을 정리해둔 틀**을 의미한다.

### 단순 프롬프트와의 차이

| 단순 프롬프트 | 하네스 엔지니어링 |
|--|--|
| 매번 같은 컨텍스트를 다시 설명 | 컨텍스트를 파일로 영속화 |
| 결과가 들쭉날쭉 | 일정한 품질 유지 |
| AI가 컨벤션을 까먹음 | 컨벤션을 매번 참조 가능 |
| 한 번 쓰고 버리는 대화 | 재사용 가능한 자산 |

### 왜 .md 파일로 관리하는가

- **사람도 읽기 좋다** — 팀원에게 공유할 수 있는 문서가 됨
- **버전 관리 가능** — Git으로 추적, 변경 이력 남김
- **AI가 자연어로 이해** — 별도 포맷 학습 없이 바로 활용
- **점진적 개선** — 작업하면서 발견한 룰을 계속 누적

## 내가 쓴 하네스 구조

롯데카드 백오피스에서 실제로 쓴 구조는 대략 이렇다.

```
project-root/
├── CLAUDE.md                               # root 경로에 CLAUDE.md가 있을 경우 자동 반영
│   ├── docs
│   │   ├── harness-engineering.md          # 프로젝트 전체 컨텍스트
│   │   ├── application-architecture.md     # 프로젝트 아키텍쳐 설명  
│   │   ├── api-contract.md                 # DTO, 에러, 보안 등의 규칙
│   │   ├── persistence-querydsl-jpa.md     # Repository 구성, QueryDSL 작성 규칙 
│   │   ├── testing-harness.md              # 테스트 코드 작성
└── src/
```

### `harness-engineering.md`
프로젝트가 무엇이고, 어떤 기술 스택을 쓰고, 어떤 도메인을 다루는지 적어둔다.
- 서비스 한 줄 소개
- 목적
- 상황별 | 피처 분리 및 타 md 파일 import
- 기술 스택 (Kotiln, Spring 등)
- 주요 도메인 용어 (키오스크, 시재, 환전 등)
- 외부 연동

### `application-architecture.md`
프로젝트 아키텍쳐 설명.
- 패키지와 계층 규칙
- 폴더 구조
- 컴포넌트 작성 패턴
- 트랜잭션과 상태 변경 규칙

### `api-contract.md`
API 관련 계약 규칙을 정의한다.
- 요청 | 응답 DTO 규칙
- 입력·출력
- 보안과 감사 기준
- 관련 파일 목록

### `persistence-querydsl-jpa.md`
QueryDSL/JPA 규칙을 정의한다.
- Repository 구성
- QueryDSL 작성 규칙
- QueryDSL 테스트 기준 및 마이그레이션

### `testing-harness.md`
테스트 코드 작성 방법 및 범위를 설정한다.
- 테스트 위치 | BDD 이름
- MockK 기반 테스트 원칙
- 테스트 범위 기준
  
-> 예시
```
# 테스트 하네스

## 목적

이 문서는 LotteCard Admin API에서 피처 시나리오를 코드로 고정하기 위한 테스트 작성 기준을 정의한다.

테스트는 피처의 시나리오를 코드로 고정하는 장치다. 구현보다 테스트 이름이 먼저 업무 언어를 설명해야 한다.

## 테스트 위치

기본 테스트 위치는 다음과 같다.

```text
src/test/kotlin/kr/co/dozn/mx/lottecard
  module
    [feature-domain]
      command
        service
      query
        service
      controller
  domain
    [domain-name]
      repository
```

## BDD 테스트 이름

테스트 이름은 BDD 문장으로 작성한다.

```kotlin
@Test
fun `Given 사용 가능한 loginId When 관리자 계정을 생성하면 Then ACTIVE 계정이 저장된다`() {
    // given
    // when
    // then
}
```

## MockK 기반 테스트 원칙

- Service 테스트는 Spring context를 띄우지 않는다.
- Repository, encoder, 외부 client, clock 등 협력 객체는 MockK로 대체한다.
- relaxed mock은 필요한 경우에만 사용한다.
- `every`, `justRun`, `verify`, `confirmVerified`로 상호작용을 명시한다.
- 예외 시나리오에서는 저장/변경 메서드가 호출되지 않았음을 검증한다.
- 테스트 데이터는 시나리오 안에서 읽히는 fixture builder를 사용한다.

MockK 의존성이 없는 경우 테스트 하네스 구성 시 먼저 추가한다.

```kotlin
testImplementation("io.mockk:mockk")
```

서비스 테스트 예시는 다음과 같다.

```kotlin
@ExtendWith(MockKExtension::class)
class AdminCommandServiceTest {

    @MockK
    lateinit var adminRepository: AdminRepository

    @MockK
    lateinit var passwordEncoder: PasswordEncoder

    @InjectMockKs
    lateinit var service: AdminCommandService

    @Test
    fun `Given 중복 loginId가 없을 때 When 관리자를 생성하면 Then 암호화된 비밀번호로 저장된다`() {
        // given
        val request = CreateAdminRequest(
            loginId = "admin01",
            password = "raw-password",
            name = "관리자",
            email = "admin@lottecard.co.kr",
            roleType = RoleType.ADMIN,
            userType = UserType.INTERNAL
        )

        every { adminRepository.existsByLoginId("admin01") } returns false
        every { passwordEncoder.encode("raw-password") } returns "encoded-password"
        every { adminRepository.save(any()) } answers { firstArg() }

        // when
        service.createAdmin(request)

        // then
        verify {
            adminRepository.existsByLoginId("admin01")
            passwordEncoder.encode("raw-password")
            adminRepository.save(match {
                it.loginId == "admin01" &&
                    it.password == "encoded-password" &&
                    it.status == AccountStatus.ACTIVE
            })
        }
        confirmVerified(adminRepository, passwordEncoder)
    }
}
```

## 테스트 범위 기준

테스트는 위험도에 따라 다음 중 하나를 선택한다.

| 범위 | 목적 | 기본 도구 |
| --- | --- | --- |
| Service mock test | 유스케이스 분기, 예외, 협력 객체 호출 검증 | JUnit5 + MockK |
| Controller slice test | validation, status code, JSON 계약, security rule 검증 | MockMvc |
| Repository test | QueryDSL where/order/page/count 검증 | `@DataJpaTest` |
| Integration test | transaction, security, serialization, DB mapping 결합 검증 | SpringBootTest |

기본은 Service mock test다.

다음 경우에는 Repository 또는 Integration 테스트를 추가한다.

- QueryDSL 동적 조건이 2개 이상이다.
- 페이징, 정렬, count query가 업무 결과에 직접 영향을 준다.
- 통화, 금액, 거래 상태처럼 정합성 위험이 큰 쿼리다.
- JPA 연관관계, fetch join, batch size, lazy loading이 결과에 영향을 준다.
- security rule 또는 controller validation이 피처의 핵심 acceptance criteria다.

## 관련 문서

- QueryDSL 조회 테스트 기준은 [Persistence, QueryDSL, JPA 규칙](persistence-querydsl-jpa.md)을 함께 확인한다.
- 인증/인가 테스트 기준은 [인증 프로세스](authentication-process.md)를 따른다.
```



## IntelliJ + Claude Code 셋업

### 설치
- Claude Code 플러그인 설치
- 인증 (API 키 또는 로그인)
- 프로젝트 루트에 하네스 파일 배치

### 자주 쓰는 명령어
/plan

## 배운 점

### 하네스로 만들면 효과적인 것

- **반복되는 컨벤션** — 매번 설명하기 귀찮은 룰
- **도메인 용어** — 비즈니스 맥락이 깊은 용어들
- **외부 API 스펙** — 매번 문서 찾아 붙이는 대신 정리해두기
- **에러 케이스** — "이럴 땐 이렇게 처리한다"의 베이스라인

### 하네스보다 직접 코딩이 빠른 것

- **한 번만 쓰는 일회성 코드** — 하네스 만드는 시간이 더 걸림
- **명확한 정답이 있는 작업** — 단순 리팩토링, 변수명 변경 등
- **고도로 창의적인 설계** — AI가 맥락만으로는 풀기 어려운 영역

### 가장 큰 교훈

> 하네스 엔지니어링은 결국 **"내가 무엇을 만들고 있는지를 스스로 명확히 정리하는 일"**이다.
> AI에게 잘 설명하려고 정리하다 보면, 내가 모호하게 알고 있던 부분이 드러난다.

AI 도구를 잘 쓰는 사람은 결국 본인 작업을 잘 정리하는 사람이라는 걸 체감했다.

## 참고
- [Claude Code 공식 문서](https://docs.claude.com/en/docs/claude-code)
