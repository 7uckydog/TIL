# Spring 요청/응답 로깅: Interceptor 대신 Filter + AOP

## 배경
API 서버에서 요청/응답을 로그로 남기고 싶었다. 흔히 `HandlerInterceptor`의
`postHandle`에서 응답 바디를 찍는 방식을 쓰는데, 두 가지 문제를 만났다.

1. 응답 바디를 읽으려면 `ContentCachingResponseWrapper`로 **응답을 통째로 버퍼링**해야 한다.
2. 그렇게 찍은 payload는 **마스킹 없이 원문**이라, 비밀번호·카드번호 같은 민감정보가 그대로 로그에 남는다.

그래서 로깅 방식을 다시 정리했다.

## Filter vs Interceptor vs AOP — 무엇으로 찍을까

| 방식 | 잡히는 위치 | 손에 들어오는 것 |
|---|---|---|
| Servlet Filter | DispatcherServlet 앞 | HttpServletRequest/Response (헤더, status, 바이트) |
| HandlerInterceptor | 컨트롤러 호출 전후 | 요청/응답 (단, 바디는 스트림 → 캐싱 필요) |
| AOP (@Around) | 컨트롤러 메서드 호출 | 메서드 인자/반환값 = 객체 그 자체 |

핵심 차이: **인터셉터는 "바이트"를, AOP는 "객체"를 잡는다.**
마스킹은 객체 단위로 해야 깔끔하다(필드별로 가릴 수 있으니까). 그래서:

- HTTP 메타(method/uri/헤더/status/소요시간) → Filter
- 컨트롤러 요청 인자 / 응답 본문 → AOP

## 1) HTTP 메타는 Filter에서

추적 ID(MDC)도 같은 필터에서 세팅하면 모든 로그에 자동으로 붙는다.

    class TraceLoggingFilter : OncePerRequestFilter() {
        override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
            MDC.put("cid", correlationId(req))
            val start = System.currentTimeMillis()
            try {
                chain.doFilter(req, res)
            } finally {
                log.info {
                    "httpRequest method=${req.method} uri=${req.requestURI} " +
                    "headers=${maskHeaders(req)} status=${res.status} elapsedMs=${System.currentTimeMillis() - start}"
                }
                MDC.clear()
            }
        }
    }

## 2) 컨트롤러 요청/응답은 AOP에서

    @Aspect
    @Component
    class ControllerLoggingAspect(private val masker: LogMasker) {
        @Around("@within(org.springframework.web.bind.annotation.RestController)")
        fun log(jp: ProceedingJoinPoint): Any? {
            log.info { "request ${sig(jp)} args=${masker.mask(jp.args)}" }   // 객체를 마스킹
            val result = jp.proceed()
            log.info { "response ${sig(jp)} body=${masker.mask(result)}" }
            return result
        }
    }

> 인터셉터처럼 응답을 버퍼링하지 않아도 된다. 반환 객체를 그대로 받아 마스킹해서 찍으면 끝.

## 마스킹: 어노테이션 vs 필드 이름
- 어노테이션 방식(@Mask): 정확하지만, 모든 DTO에 일일이 붙여야 한다.
- 필드 이름 방식: password, cardNo, email 처럼 이름으로 자동 마스킹. 어노테이션 안 붙여도 됨.
- 둘을 같이 쓰면 좋다: 이름으로 자동 + 예외 케이스만 어노테이션.

주의: 이름 기반은 이름이 안 맞고 어노테이션도 없으면 그대로 노출된다(한계).
그래서 card 처럼 너무 짧은 키워드는 위험하다 — cardName(비민감)까지 가려버린다.
도메인에 맞춰 cardNo/accountNo 같이 구체적인 키워드를 골라야 한다.

## 코루틴(suspend) 함수 + AOP 주의점
컨트롤러가 suspend fun이면 AOP에서 두 가지를 조심해야 한다.
- 인자 마지막에 Continuation 객체가 숨어 들어온다 → 로그에 직렬화되면 쓰레기. 필터링 필요.
- proceed()가 실제 값 대신 COROUTINE_SUSPENDED 마커를 반환할 수 있다 → 응답 로깅 스킵 처리.

    val args = jp.args.filterNot { it is Continuation<*> }
    val result = jp.proceed()
    if (result?.javaClass?.name != "kotlin.coroutines.intrinsics.CoroutineSingletons") {
        logResponse(result)
    }

## 덤: SQL 로그는 파일을 따로
SQL은 양이 많아 앱 로그에 섞이면 추적이 어렵다. P6Spy + Log4j2로 분리했다.
- p6spy 로거를 전용 appender로만 보내고 additivity: false → 앱 로그로 안 샘
- 결과: app.log(애플리케이션) / app-sql.log(쿼리) 분리

## 정리
- 로깅은 한 군데서 다 하려 하지 말고 계층별로 나눈다: HTTP 메타=Filter, 본문=AOP, SQL=P6Spy.
- 본문 로깅은 객체를 잡는 AOP가 마스킹에 유리하다(인터셉터는 바이트라 불리).
- 마스킹은 이름 자동 + 어노테이션 보완, 키워드는 구체적으로.
- suspend 컨트롤러는 Continuation / COROUTINE_SUSPENDED 처리를 잊지 말 것.
