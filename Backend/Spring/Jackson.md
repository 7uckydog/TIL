# Kotlin + Jackson: `is` 접두사 프로퍼티가 프론트에서 `undefined`로 오는 문제

> 상세 응답 DTO에 `isMain` 필드를 추가했는데, 프론트에서만 `undefined`로 찍혔다.
> BE 로그에는 값이 멀쩡히 있었다. 범인은 프론트도 API 로직도 아닌, **Jackson이 Kotlin의 `is`-getter를 getter로 인식하지 못한 것**이었다.

## 상황

- API → 프론트로 광고 상세 정보를 내려주는 응답에 `isMain`(메인 노출 여부, `Y`/`N`) 컬럼이 추가됨
- 상세 값 중 유독 `isMain`만 프론트에서 `undefined`
- 반면 **BE 로그에서는 `isMain`이 `Y`/`N`으로 정상 출력**

즉, 서버 객체에는 값이 있는데 JSON으로 나가는 순간 프론트가 찾는 키가 사라진 상황.

## 원인

Kotlin이 만든 getter 이름을 Jackson이 **getter로 인식하지 못해**, 그 프로퍼티가 JSON에서 통째로 빠진 것이 원인이었다.

1. **Kotlin의 `is` 접두사 getter 규칙**
   프로퍼티 이름이 `is`로 시작하면, Kotlin은 getter를 `getIsMain()`이 아니라 **`isMain()`** 으로 생성한다. (setter는 `setMain()`)

2. **Jackson의 getter 인식 규칙**
   Jackson(JavaBeans 관례)은 getter를 딱 두 형태로만 인식한다.
   - `getXxx()` → 일반 getter
   - `isXxx()` → **반환 타입이 `boolean`/`Boolean`일 때만** getter (프로퍼티 `xxx`)

   그런데 여기서 `isMain`은 `Y`/`N` **`String`** 이다. 그래서 `isMain()`은
   - `get`으로 시작하지 않으니 일반 getter도 아니고,
   - 반환 타입이 boolean이 아니니 is-getter도 아니다.
   → **어느 쪽으로도 getter로 인식되지 않는다.**

3. **결과**
   getter가 인식되지 않으면 Jackson 입장에서 이 프로퍼티는 직렬화할 접근자(accessor)가 없다.
   Kotlin 필드는 기본이 private이라 필드 직접 접근도 막혀 있다.
   → 프로퍼티가 **JSON에서 통째로 빠진다.** (`isMain`도 `main`도 둘 다 없음)

> 로그에 값이 멀쩡했던 이유: 로깅은 객체 필드(`toString`)를 그대로 찍기 때문에 직렬화 규칙과 무관하다. **문제는 오직 JSON 직렬화 단계에서만 발생한다.**

### 타입에 따라 증상이 갈린다

같은 `is` 접두사라도 필드 타입에 따라 결과가 다르다. 이게 디버깅을 헷갈리게 만드는 지점.

- **`Boolean isMain`** → `isMain()`이 is-getter로 인식됨 → 키가 **`main`으로 바뀌어** 나간다 (값은 `res.main`에 있음)
- **`String`(`Y`/`N`) 등 non-boolean `isMain`** → getter 인식 실패 → **키가 통째로 사라진다** ← 이번 케이스

### 진단 팁

특정 필드만 `undefined`면 응답 JSON 원본(Network 탭 / `curl`)을 열어
**그 키가 아예 없는지 / 접두사 빠진 다른 이름(`main`)으로 있는지**를 먼저 확인하면 이 케이스인지 바로 잡힌다.

## 해결

직렬화에 쓰이는 **getter에 `@get:JsonProperty`** 를 붙여 JSON 키를 고정한다.

```kotlin
// Before — isMain 이 JSON에서 통째로 빠짐 (isMain, main 둘 다 없음)
data class AdDetailResponse(
    val id: Long,
    val title: String,
    val isMain: String,   // "Y" / "N"
)

// After — JSON 키를 "isMain" 으로 고정
data class AdDetailResponse(
    val id: Long,
    val title: String,
    @get:JsonProperty("isMain")
    val isMain: String,
)
```

## 왜 `@get:` 이어야 하나

Kotlin에서 생성자 프로퍼티에 use-site target 없이 `@JsonProperty`만 붙이면,
어노테이션이 **생성자 파라미터/필드** 쪽에 붙어 직렬화(=getter 기반) 경로에서 원하는 대로 안 먹을 수 있다.

- **직렬화(BE → JSON)** 는 getter를 읽는다 → 이름 규칙을 확실히 바꾸려면 getter에 붙여야 한다.
- 그래서 `@get:JsonProperty("isMain")` 이 이 문제에 대한 가장 정확한 타깃이다. getter를 명시적으로 프로퍼티 `isMain`에 매핑해주니, 인식 실패로 빠지던 필드가 다시 살아난다.

## 정리

- Kotlin `isXxx` 프로퍼티 → getter는 `getIsXxx()`가 아니라 `isXxx()`
- Jackson은 `isXxx()`를 **반환 타입이 boolean일 때만** getter로 인식
- `Y`/`N` `String`이라 getter로 인식되지 못했고, private 필드라 접근자도 없어 **프로퍼티가 JSON에서 통째로 빠졌다** (`isMain`도 `main`도 없음)
- 해결: `@get:JsonProperty("isMain")` 으로 getter를 명시 매핑해 직렬화 이름 고정
- boolean이었다면 키가 `main`으로 바뀌었을 것 — `is` 접두사 플래그 필드는 타입 불문 어노테이션을 기본으로 챙기면 재발 방지

## 참고

- Kotlin Docs — Properties (getter/setter 네이밍): https://kotlinlang.org/docs/properties.html
- jackson-module-kotlin (is- 접두사 관련 이슈): https://github.com/FasterXML/jackson-module-kotlin
