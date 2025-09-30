+++
title = "커스텀 HTTP 헤더 설정을 만들 때 고려해야 할 것들"
date = 2025-09-29
description = "HTTP 명세에 따라 고려해야 하는 multi-value 헤더 및 예약된 헤더 등의 엣지 케이스"
[taxonomies]
categories = ["infra"]
tags = ["HTTP"]
[extra]
lang = "ko"
toc = true
+++

Helm을 사용하면서 yaml과 go temlate의 조합에 불편함을 느끼던 중 Pkl을 알게 되었다. Pkl 이슈 중 패키지 저장소에 HTTP 요청을 보낼 때 인증을 적용할 방법이 없어서 난감하다는 내용의 이슈를 발견했고, HTTP 헤더를 커스텀할 수 있도록 해서 간단하게 이슈를 해결할 수 있겠다고 생각했다. 처음에는 헤더를 `Map<String, String>` 타입의 옵션으로 받으면 될 것이라고 생각했으나, HTTP 명세에 따르면 몇몇 특수한 경우에 한해 동일한 키의 헤더가 여러 번 사용되는 경우가 존재하는 문제가 있었다. 또, 브라우저 등 HTTP 클라이언트가 제어할 목적으로 예약된 Reserved Header의 특수한 경우도 고려해야 했다.


## multi-value 헤더

RFC 7230, 3.2.2. 필드 순서 (Field Order) 섹션

> A sender MUST NOT generate multiple header fields with the same field name in a message unless either the entire field value for that header field is defined as a comma-separated list [i.e., #(values)] or the header field is a well-known exception (as noted below).

송신자는 기본적으로 동일한 이름의 헤더를 여러 개 보내어서는 안 되지만, 스펙 상에 지정된 몇 가지 예외가 존재하고, 따라서 필요에 따라 동일한 이름의 헤더 여러 개를 보낼 수 있어야 한다.

### 멀티 헤더 실 사용 예

1. `Set-Cookie`

여러 개의 쿠키를 한 번에 응답에 설정하는 경우

```http
Set-Cookie: session_id=abcde12345; HttpOnly; Secure
Set-Cookie: theme=dark; Path=/; Expires=Wed, 21 Oct 2026 07:28:00 GMT
```

2. `Cache-Control`

캐시 동작을 상세하게 제어하는 경우 (RFC 7234 5.2. Cache-Control)

```http
Cache-Control: private
Cache-Control: no-cache
```

두 헤더는 각각의 옵션으로 적용된다.

+ 이 외에
`Accept-Language`를 포함한 `Accept-`접두사 헤더, 프록시 경로 추적을 위한 `Via`, 그리고 각종 커스텀 헤더 등의 다양한 사용예가 있다.

## 예약된 헤더

1. [금지된 헤더](https://developer.mozilla.org/ko/docs/Glossary/Forbidden_request_header)
샤용자 에이전트가 헤더에 대한 모든 권한을 보유하므로 프로그래밍 방식으로 수정할 수 없는 헤더의 이름들.
`Proxy-`와 `Sec-` 접두사를 사용하는 헤더 이름은 보안을 위해 예약되어 있으므로 허가해서는 안 되며,
다른 예약된 해더는 해당 페이지에 리스트로 존재한다.

2. [CORS 안전목록의 요청 헤더](https://developer.mozilla.org/docs/Web/HTTP/Guides/CORS)
CORS(교차 출처 공유 공유)를 위해 제어 요청에 사용되는 헤더들. `Origin` 또는 `Access-Control-` 접두사를 사용.

### 예외 헤더 규칙

1. `Content-Type`의 값은 반드시 type/subtype의 형식이어야 한다. (RFC 7231 3.1.1.1. Media Type)
2. `Authorization`의 값은 반드시 scheme credentials로 구성되어야 한다. (RFC 7235 4.2 Authorization)
3. `Accept`의 값은 반드시 type/subtype;q=value로 구성되어야 한다. (RFC 7231 5.3.2 Accept)

## 해결안

1. headers의 타입 변경

```kotlin
val headers: Map<String, Map<String, List<String>>>
```

2. HTTP 헤더 이름과 값의 유효성 기준 검사 (RFC 7230 3.2 Header Fields)

token 규칙(RFC 7230 3.2.6. Field Value Components)
- US-ASCII character set
- token that are allowed in tchar rule
- case-insensitive

field-content 규칙 (RFC 7230 3.2 Header Fields)
- Visiable ASCII Charaters
- OWS: Space and Horizontal Tab
- obs-text
- without Control Characters

```kotlin
fun String.isValidHttpHeaderName(): Boolean {
    // ^[a-zA-Z0-9!#\$%&'*+-.^_`|~]+$
    // ^: 문자열 시작
    // a-zA-Z0-9: 영문 대소문자와 숫자
    // !#\$%&'*+-.^_`|~: RFC 7230 에서 token에 허용하는 특수문자
    // +: 1번 이상 반복
    // $: 문자을 끝
    val headerNameRegex = Regex("^[a-zA-Z0-9!#\$%&'*+-.^_`|~]+$")
    return headerNameRegex.matches(this)
}

fun String.isValidHttpHeaderValue(): Boolean {
    // ^[\t\u0020-\u007E\u0080-\u00FF]*$
    // ^: 문자열 시작
    // [\t]: 탭 문자
    // \u0020-\u007E: 스페이스를 포함한 모든 보이는 ASCII 문자
    // \u0080-\u00FF: obs-text (과거 호환용)
    // *: 0번 이상 반복
    // $: 문자열 끝
    val headerValueRegex = Regex("^[\\t\\u0020-\\u007E\\u0080-\\u00FF]*$")
    return headerValueRegex.matches(this)
}
```

3. 금지 헤더 목록 검사

```kotlin
val forbiddenHeaderNames = setOf (
    "origin",
    "host",
    "referer",
    "cookie",
    "user-agent",
    "connection",
    // ...
)

fun String.isAllowedHeader(): Boolean {
    val lowerCaseHeaderName = headerName.toLowerCase() // 헤더 이름은 대소문자를 구분하지 않음

    // 접두사 검사 (기존 정규식)
    if (lowerCaseHeaderName.startsWith("proxy-") || lowerCaseHeaderName.startsWith("sec-") ||
        lowerCaseHeaderName.startWith("access-control-")) {
        return false
    }

    // B. 이름 검사 (금지 목록 확인)
    if (forbiddenHeaderNames.contains(lowerCaseHeaderName)) {
        return false // 금지된 이름이면 실패
    }

    return true // 모든 검사를 통과하면 성공
}
```

4. 예외 문법 처리

```kotlin
private val CONTENT_TYPE_REGEX = Regex("^[\\w+-.]+/[-.\\w+]+.*$")
private val AUTHORIZATION_REGEX = Regex("^\\w+ .+$")
private val ACCEPT_PART_REGEX = Regex("^(\\*|\\w+)/(\\*|[-.\\w+]+)(;\\s*q=\\d(\\.\\d+)?)?$")

fun validateSpecificHeaderSyntax(name: String, value: String): Boolean {
    return when (name.lowercase()) {
        "content-type" -> CONTENT_TYPE_REGEX.matches(value.trim())
        "authorization" -> AUTHORIZATION_REGEX.matches(value.trim())
        "accept" -> {
            if (value.isBlank()) false
            else value.split(',').all { part -> ACCEPT_PART_REGEX.matches(part.trim()) }
        }
        else -> true
    }
}
```
