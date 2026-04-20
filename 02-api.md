# DEPS - API 규격

## 기본 원칙

DEPS에서 정의하는 모든 API는 **반드시** HTTPS 프로토콜만을 사용한다.

DEPS에서 정의하는 모든 API는 `/_deps/`를 기본 경로로 사용한다. API의 경로는 **반드시** 도메인의 루트에서 시작해야 한다.

    https://some.example.org/_deps/...        // OK
    http://203.0.113.123/_deps/...            // X, HTTPS를 사용해야 함
    https://example.com/homeserver/_deps/...  // X, 도메인 루트에 존재해야 함

- 홈 서버의 모든 API는 `/_deps/home/`를 베이스로 한다.
- 문제 서버의 모든 API는 `/_deps/judge/`를 베이스로 한다.
- 하나의 도메인 아래에 홈 서버와 문제 서버가 같이 있을 수도 있다.
    - 이 경우 둘의 공개 키는 같을 수도, 다를 수도 있다.
    - 어떤 도메인을 해석할 때 이를 홈서버로 해석할 것인지 문제서버로 해석할 것인지의 여부는 문맥에 따라 달라진다.

DEPS에서 정의하는 모든 API는, 달리 정의하지 않는 한, JSON을 사용하여 통신한다. 서버는 JSON 데이터를 전송할 때 `Content-Type` 헤더를 `application/json`으로 설정하여야 한다.

### 오류 응답

DEPS에서 정의하는 API는 오류 상황이 아닐 때 `2xx` 계열의 HTTP 상태 코드를 반환하여야 한다.

만약 `4xx` 또는 `5xx` 을 반환하는 경우, 그 body는 달리 정의하지 않는 한 다음과 같도록 한다. 이를 **오류 응답**이라 한다.

API가 DEPS에서 정의하지 않은 다른 이유로 인해 `4xx`나 `5xx`를 반환하는 경우에도 서버는 올바른 **오류 응답**을 반환해야 한다.

```typescript
{
    "error": string,
    "reason": string,
    // 서버는 그 밖에 다른 필드를 포함할 수 있음
}
```

- `error`는 오류 코드이다.
- `reason`은 서버에서 임의로 작성한 오류 메시지이다.

문서에서 정의하는 `error` 코드의 목록은 다음과 같다. 그러나 서버는 아래에 없는 오류 코드를 반환할 수도 있다.

- `E_NOT_FOUND`: 리소스를 찾을 수 없음.
- `E_WRONG_SERVER`: 이 서버의 리소스가 아님.
- `E_INVALID_SIGN`: 서명 검증 실패.
- `E_OUTDATED_SIGN`: signedAt 값이 기준보다 오래됨.
- `E_INVALID_REQUEST`: 요청이 올바르지 않음.
- `E_INSUFFICIENT_POW`: PoW 검증 실패.
- `E_RATE_LIMITED`: Rate-Limit에 걸렸음.

#### PoW 검증 실패

서버가 PoW가 요구되는 엔드포인트에서의 PoW 검증에 실패했다면, **403 Forbidden** HTTP 상태 코드와 함께 다음처럼 응답한다.

```typescript
{
    "error": "E_INSUFFICIENT_POW",
    "reason": string,
    "powFactor": number,
}
```

- `reason`은 서버가 임의로 작성한다.
- `powFactor`는 서버가 요구하는 **PoW 수준**이다. 클라이언트는 이 값을 참고하여 다시 올바른 PoW를 계산해야 한다.

#### Rate-Limit

서버는 어떤 클라이언트의 너무 잦은 요청을 거부하고자 할 때, **429 Too Many Requests** HTTP 상태 코드와 함께 다음처럼 응답할 수 있다.

```typescript
{
    "error": "E_RATE_LIMITED",
    "reason": string,
    "retryAfter": Date
}
```

- `reason`은 서버가 임의로 작성한다.
- `retryAfter`는 요청을 다시 보낼 수 있게 되는 시각이다.

또한, 다음처럼 `Retry-After` HTTP 헤더를 같이 전송한다. 이 헤더는 `retryAfter`와 같은 날짜를 가리켜야 한다.

```
Retry-After: Mon, 20 Apr 2026 07:28:00 GMT
```

클라이언트는 `retryAfter` 시점이 지나기 전까지 같은 엔드포인트로 요청을 보내지 말아야 한다.

Rate-Limit의 기간, 적용 단위, 다른 엔드포인트까지의 적용 범위 등 Rate-Limit에 관한 세부적인 사항은 서버 구현체가 적절하게 결정한다. 


### 페이지네이션

특정 API는 많은 수의 데이터를 조금씩 분할하여 제공할 수 있다. 이를 **페이지네이션**이라 한다.

DEPS에서 제공하는 API들은 페이지네이션을 공통적으로 다음과 같이 처리한다.

1. 데이터 목록은 API 규격 문서에 정의되어 있는 기준으로 정렬되어 있어야 한다.
2. 해당 엔드포인트는 쿼리 파라미터로 `offset` 과 `limit`을 가진다.
    - `offset`은 **선택적 파라미터**이다.
        - 0 이상의 정수이다.
        - 데이터 목록의 맨 앞에서, 이 개수만큼의 데이터를 스킵해야 함을 뜻한다.
        - 없다면 `0`으로 간주한다.
    - `limit`은 **선택적 파라미터**이다.
        - 0보다 큰 정수이다.
        - 받아올 데이터 개수의 상한값을 뜻한다.
        - 없다면 서버가 한 번에 제공할 수 있는 데이터 개수의 상한값만큼 제공한다.
3. 해당 엔드포인트는 다음과 같은 형태의 응답을 가진다.
    ```typescript
    {
        // ... 임의의 데이터 목록 ...
        // (예시) "data": SomeData[],
        
        // 서버에 존재하는 데이터의 총 개수
        "totalCount": number,
    }
    ```
    - `offset`이 제공할 수 있는 데이터의 범위를 벗어났다면, 데이터로는 빈 목록 ( `[]` )을 제공한다.
    - `limit`이 서버가 제공할 수 있는 상한값보다 높다면, 해당 상한값만큼의 데이터를 제공한다.
    - 제공되는 데이터는 정렬되어 있어야 하며, 그 개수는 `limit`보다 **작거나 같아야** 한다.
    - `totalCount`는 서버에 존재하는 제공 가능한 데이터의 총 개수와 같아야 한다.


## 홈 서버 (`/_deps/home/...`)

홈 서버의 모든 API 엔드포인트는 `/_deps/home/`으로 시작한다.

예를 들어, 아래에서 `/info`라 함은 `/_deps/home/info`를 뜻한다.

### GET `/info`

홈서버의 기본 정보를 받아온다.

#### 응답
```typescript
{
    "key": string,
    "extensions": string[],
}
```

- key: 이 서버의 **인코딩된 공개 키**
- extensions: 이 서버가 지원하는 **익스텐션 버전 식별자**의 목록
    - 자세한 사항은 `03-extensions` 문서 참고.


### GET `/identities/[identity]`

홈서버에 등록된 아이덴티티의 정보를 받아온다.

- `[identity]`: `아이덴티티 식별자`

#### 응답
```typescript
// "데이터 정의" 문서에서 정의된 "Identity" 정의와 같음
{
    "key": string,
    "data": IdentityInfo,
    "sign": string
}
```

#### 오류 응답
- 이 홈서버에 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOTFOUND`)


### POST `/identities/[identity]`

> [!NOTE]
> 이 API는 홈서버에 이미 등록된 아이덴티티의 기본 정보를 수정하는 API이다.
> 
> 홈서버에 신규 아이덴티티를 등록하는 방법은 프로토콜 단에서 정의하지 않으며, 각 구현체가 적절한 방법을 제공해야 한다.
> 
> 마찬가지로 각 홈서버는 적절한 방법으로 아이덴티티의 반출/반입 (서버 이전) 방법을 제공해야 한다.

홈서버에 등록된 아이덴티티 정보를 갱신한다.

- `[identity]`: `아이덴티티 식별자`

이 엔드포인트에 대한 요청을 처리하기 전, 서버는 **반드시** 다음을 검증해야 한다.
1. `[identity]`는 이 홈서버에 등록된 아이덴티티 식별자여야 한다.
2. 요청의 `signedAt`는 서버에 등록된 아이덴티티의 `signedAt`보다 미래의 날짜여야 한다.
3. `key`는 `[identity]`와 같아야 한다.
4. `sign`은 올바른 서명이어야 한다.

#### 요청
```typescript
// "데이터 정의" 문서에서 정의된 "Identity" 정의와 같음
{
    "key": string,
    "data": IdentityInfo,
    "sign": string
}
```

#### 응답
```typescript
{
    "ok": true,
}
```

#### 오류 응답
- 이 홈서버에 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOTFOUND`)
- 요청의 `signedAt` 값보다 홈서버에 기록된 `signedAt` 값이 더 최신이라면, **409 Conflict** (`E_OUTDATED_SIGN`)


### GET `/identities/[identity]/solves?offset=..&limit=..`

등록된 아이덴티티가 가진 모든 문제증표를 **페이지네이션하여** 나열한다.

- `[identity]`: `아이덴티티 식별자`

#### 응답
```typescript
{
    "data": SolveCertificate[],
    "totalCount": number,
}
```

- 증표의 정렬 방법은 서버가 정하지만, 전체 목록은 잘 정렬되어 있어야 한다.
- 서버는 경우에 따라 일부 문제증표를 제공하지 않을 수 있다.
    - 이 경우 `totalCount`는 제공하지 않는 증표의 개수를 제외한 값이어야 한다.

페이지네이션은 증표의 등록 시점, 또는 서명(`signedAt`) 시점을 기준으로 내림차순으로 정렬한다.

#### 오류 응답
- 이 홈서버에 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOTFOUND`)


### PUT `/identities/[identity]/solves`

등록된 아이덴티티에 문제증표를 추가한다.

- `[identity]`: `아이덴티티 식별자`

서버는 해당 문제증표를 추가하기 전, 중복 확인 및 **문제증표의 검증**을 수행해야 한다.



#### 요청

> [!NOTE]
> 문제증표는 아이덴티티가 직접 추가해야 한다.
> 
> 악의적인 문제서버가 누군가에게 스팸성 문제증표를 추가하는 것을 막기 위해 이와 같이 설계하였다.

```typescript
{
    "key": string,
    "data": {
        "cert": SolveCertificate,
        "signedAt": Date,
    },
    "sign": string,
}
```

- `key`: 아이덴티티 식별자.
- `data`: 데이터.
    - `cert`: 문제 증표. **문제증표의 검증**을 통과해야 한다.
    - `signedAt`: 이 데이터의 서명 시각.
- `sign`: **아이덴티티의** 서명.

#### 응답
```typescript
{
    "ok": true,
}
```

#### 오류 응답
- 이 홈서버에 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOTFOUND`)
- `key`가 `[identity]`와 다르다면, **400 Bad Request** (`E_INVALID_REQUEST`)
- `sign`이 아이덴티티의 올바른 서명이 아니라면, **403 Forbidden** (`E_INVALID_SIGN`)
- 문제증표의 검증을 통과하지 못했다면, **403 Forbidden** (`E_INVALID_SIGN`)
- 문제증표가 에라타에 의해 무효하게 되었다면, **403 Forbidden** (`E_OUTDATED_SIGN`)
- 동등한 문제증표가 이미 존재하며 그 문제증표의 `signedAt` 값이 요청의 `signedAt` 값보다 같거나 미래라면, **409 Conflict** (`E_OUTDATED_SIGN`)
    - "동등한 문제증표"의 기준은 구현체가 적절히 결정한다.


### GET `/identities/[identity]/handle`

아이덴티티의 등록된 핸들네임을 받아온다.

- `[identity]`: `아이덴티티 식별자`

#### 응답

> [!NOTE]
> 서버는 본인이 아닌 다른 홈서버가 발급한 핸들네임을 제공할 수도 있다.
> 
> 이 경우 해당 핸들네임의 신뢰성은 그 홈서버에 요청하여 다시 확인해야 한다.

```typescript
{
    "identity": string,
    "handle": string | null,
}
```

- `identity`: 아이덴티티 식별자. `[identity]`와 동일한 값.
- `handle`: 이 서버에서 해당 아이덴티티에게 부여된 핸들네임. null일 수 있다.
    - `handle`은 `@<USER_NAME>::<SERVER_DOMAIN>` 형태이다.
    - `<SERVER_DOMAIN>`은 이 홈서버의 도메인 주소와 **같지 않을 수 있다.**

#### 예외
- 이 홈서버에 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOTFOUND`)


### GET `/handles/[handle]`

홈서버에 등록된 핸들네임을 통해 아이덴티티를 제공한다.

- `[handle]`: 올바른 핸들네임
    - `@<USER_NAME>::<SERVER_DOMAIN>` 형태이다.
    - `<SERVER_DOMAIN>`은 이 홈서버의 도메인 주소와 같아야 한다.

#### 응답
```typescript
// "데이터 정의" 문서에서 정의된 "Identity" 정의와 같음
{
    "key": string,
    "data": IdentityInfo,
    "sign": string
}
```

#### 오류 응답
- 핸들네임의 도메인이 이 홈서버의 도메인과 다르다면, **400 Bad Request** (`E_WRONG_SERVER`)
- 이 홈서버에 등록되지 않은 핸들네임이라면, **404 Not Found** (`E_NOT_FOUND`)


## 문제서버

문제 서버의 모든 API 엔드포인트는 `/_deps/judge/`으로 시작한다.

예를 들어, 아래에서 `/info`라 함은 `/_deps/judge/info`를 뜻한다.

### GET `/info`

문제서버의 기본 정보를 받아온다.

#### 응답
```typescript
{
    "key": string
    "extensions": string[],
}
```

- `key`: Base64로 인코딩된 이 서버의 Ed25519 공개 키
- extensions: 이 서버가 지원하는 **익스텐션 버전 식별자**의 목록
    - 자세한 사항은 `03-extensions` 문서 참고.


### GET `/problems?offset=..&limit=..`

문제 목록을 **페이지네이션하여** 받아온다.

#### 응답
```typescript
{
    "data": {
        "id": string,
        "title": string
    }[],
    "totalCount": number;
}
```
- `data`: 문제 목록
    - `id`: 문제 식별자.
    - `title`: 문제의 제목.
- 문제 식별자는 `<PROBLEM_ID>::<DOMAIN>` 형태이다. `<DOMAIN>`은 이 문제서버의 도메인과 같아야 한다.

페이지네이션은 문제의 id 또는 제목을 기준으로 작동하도록 한다.

### GET `/problems/[problemId]`

특정 문제의 정보를 받아온다.

- `[problemId]`: 문제 식별자
    - `<PROBLEM_ID>::<DOMAIN>` 형태. `<DOMAIN>`은 이 문제서버의 도메인과 같아야 한다.

#### 타입 정의

```typescript
// 특정한 값의 최소/최대값 제한을 나타내는 타입.
type MinMaxRequirement = {
    // 최소값 제한. undefined라면 최소값 제한은 없다.
    min: number?,

    // 최대값 제한. undefined라면 최대값 제한은 없다.
    max: number?,

    // 임의의 x에 대해, (min <= x <= max)가 참이라면 x는 이 조건을 만족한다.
    // ※ min <= max가 아니라면, 이 조건은 만족할 수 없는 것으로 간주한다.
}
```

#### 응답
```typescript
{
    "id": string,
    "title": string,
    "content": string,

    "powFactor": number,
    "formats": {
        [string]: {
            "totalBytes": MinMaxRequirement?,
            "fileCount": MinMaxRequirement?,
            "files": {
                "name": string,
                "languages": string[],
                "content": string?,
                "count": MinMaxRequirement?,
                "bytes": MinMaxRequirement?,
            }[]
        }
    };
}
```

- `id`: 문제의 id
- `title`: 문제의 제목
- `content`: Markdown 형식으로 쓰여진 문제의 본문
- `powFactor`: 제출은 이 **PoW 수준** 값을 만족해야 한다.
- `formats`: 이 문제에서 사용할 수 있는 **제출 형식** 목록. 키 값은 해당 형식의 이름이다.

**제출 형식**은 다음 규칙들을 가진다.

- `totalBytes`: _선택적._ 모든 파일 `content` 크기를 합산한 값의 허용 범위. 단위는 Byte.
- `fileCount`: _선택적._ 파일 개수의 허용 범위.
- `files`: 해당 포맷의 파일들이 만족해야 하는 규칙 목록.
    - `name`: 파일명. 와일드카드 (`*`, `?`)를 사용할 수 있다.
        - 제출된 한 파일이 여러 `name` 규칙을 만족한다면, 먼저 등장한 규칙이 우선시된다.
    - `languages`: 파일에 사용할 수 있는 언어 목록.
    - `content`: _선택적._ 이 파일에 기본으로 주어지는 내용.
    - `count`: _선택적._ 조건을 만족하는 파일 개수의 허용 범위.
        - **undefined라면 `{min:1, max:1}`으로 취급한다.**
        - 이를테면 `{min:0, max:1}`이라면 이 파일은 optional하다.
        `{min:1}`이라면 `name`을 만족하는 파일을 여러 개 제출할 수 있다.
    - `bytes`: _선택적._ 파일 크기의 허용 범위. 단위는 Byte.

조건 검증 과정은 아래와 같은 의사 코드로 나타낼 수 있다.

```python
# 1. 포맷 존재 여부 확인
if 제출.format not in 문제.formats:
    return False

형식 = 문제.formats[제출.format]

# 2. 전체 파일 크기 및 개수 확인
if ByteSize(제출.files) not in 형식.totalBytes:
    return False
if Count(제출.files) not in 형식.fileCount:
    return False

파일카운트 = {0,}

# 3. 개별 파일 단위 검증
for 파일명, 파일 in 제출.files:
    적용할_규칙 = None
    
    # 3-1. 파일명에 맞는 우선순위 규칙 찾기
    for 규칙 in 형식.files:
        if 규칙.name matches 파일명:
            적용할_규칙 = 규칙
            break

    if 적용할_규칙 is None:
        return False  # 허용되지 않은 파일명

    파일카운트[적용할_규칙] += 1
    
    # 3-2. 언어 및 파일 크기 제약 확인
    if 파일.language not in 적용할_규칙.languages:
        return False
    if ByteSize(파일.content) not in 적용할_규칙.bytes:
        return False

# 4. 규칙별 필수 제출 개수 충족 여부 확인
for 규칙 in 형식.files:
    if 파일카운트[규칙] not in 규칙.count:
        return False

return True  # 모든 검증 통과
```

#### 오류 응답

- 문제 식별자의 도메인이 이 홈서버의 도메인과 다르다면, **400 Bad Request** (`E_WRONG_SERVER`)
- 이 문제서버에 등록되지 않은 문제라면, **404 Not Found** (`E_NOT_FOUND`)


### POST `/problems/[problemId]/submit`

문제에 답안을 제출한다.

- `[problemId]`: 문제 식별자
    - `<PROBLEM_ID>::<DOMAIN>` 형태. `<DOMAIN>`은 이 문제서버의 도메인과 같아야 한다.

#### 타입 정의
```typescript
type SubmissionFile = {
    // 이 파일에서 사용한 언어. 서버에서 요구하는 조건을 만족해야 한다.
    language: string,

    // 파일 내용.
    content: string,
}
```

#### 요청

```typescript
{
    "key": string,
    "data": {
        "format": string,
        "files": {
            [string]: SubmissionFile,
        },
        "signedAt": Date,
    }
    "sign": string,
    "pow": string,
}
```

- `key`: 제출자의 `아이덴티티 식별자`
- `data`: 제출 데이터
    - `format`: 이 제출의 **제출 형식**
    - `files`: 제출할 파일들을 담은 object.
        - 키 값은 파일명이다.
    - `signedAt`: 제출 시각
- `sign`: data에 대한 서명
- `pow`: PoW를 만족하기 위해 계산된 64바이트 미만 ASCII 문자열.

#### 응답

```typescript
{
    "submissionId": string
}
```

- `submissionId`: 해당 제출에 부여된 **제출 ID**

#### 오류 응답

- 문제 식별자의 도메인이 이 홈서버의 도메인과 다르다면, **400 Bad Request** (`E_WRONG_SERVER`)
- 이 문제서버에 등록되지 않은 문제라면, **404 Not Found** (`E_NOT_FOUND`)
- 서명이 올바르지 않다면, **403 Forbidden** (`E_INVALID_SIGN`)
- 제출이 서버에서 요구하는 PoW 수준을 만족하지 못한다면, **403 Forbidden** (`E_INSUFFICIENT_POW`)
- 제출 데이터가 문제에서 요구하는 제출 형식의 조건을 만족하지 못한다면, **400 Bad Request** (`E_INVALID_REQUEST`)


### GET `/submissions/[submissionId]`

제출의 채점 결과, 혹은 상태를 받아온다.

- `[submissionId]`: 제출 ID

#### 타입 정의

```typescript
// 이 정의는 "데이터 정의" 문서에서 정의된 "SolveCertificate"의 정의와 같다.
type SolveCertificate = {
    "key": string,  // 문제서버의 공개 키.
    "data": {
        "identity": string,
        "problemId": string,
        "score": number,
        "signedAt": Date,
    },
    "sign": string,
}
```

#### 응답
```typescript
{
    "status": "waiting" | "judging",
    "additionalInfo": string?,
    "progress": number?,
}
// OR //
| {
    "status": "finished",
    "verdict": "AC" | "PC" | "WA" | "TLE" | "MLE" | "RE" | "CE" | "UE" | "NA",
    "additionalInfo": string?,

    "certificate": SolveCertificate?,
}
```
- `status`: 채점의 현재 상태
    - `waiting`: 대기 중
    - `judging`: 채점 중
    - `finished`: 채점 완료
- `progress`: 채점 진행률. 구간 [0.0, 1.0]에 속하는 실수값.
    - 제공하지 않을 수도 있다. 제공 여부 및 이 값의 의미는 전적으로 서버가 결정한다.
- `verdict`: 채점의 결과
    - `AC`: 전체 정답
    - `PC`: 부분 정답
    - `WA`: 오답
    - `TLE`: 시간 제한 초과
    - `MLE`: 메모리 제한 초과
    - `RE`: 런타임 에러
    - `CE`: 컴파일 에러
    - `UE`: 기타 에러
    - `NA`: 채점 불가
- `additionalInfo`
    - `RE`, `CE`, `UE`, `NA`와 관련된 오류 메세지를 포함할 수 있다.
    - 컴파일 시 경고 메세지 등을 포함할 수 있다.
    - 기타 서버에서 제공하고 싶은 정보를 포함할 수 있다.
- `certificate`: `verdict`가 `AC` 또는 `PC`인 경우, 문제 증표를 포함한다.

#### 오류 응답

- 존재하지 않는 제출 ID라면, **404 Not Found** (`E_NOT_FOUND`)


### GET `/errata?offset=..&limit=..`

문제서버가 발행한 모든 **에라타** 목록을 **페이지네이션하여** 나열한다.

**에라타**는 특정 문제에 대해, 정해진 시각 이전에 발행된 모든 문제증표들을 무효화하는 선언이다. 재채점이 요구되는 문제, 조건이 변경된 문제 등에 사용할 수 있다.

#### 응답
```typescript
{
    "data": {
        [string]: Date,
    },
    "totalCount": number,
}
```

- data
    - `[string]` 키: 문제 ID.
        - `<PROBLEM_ID>::<DOMAIN>` 형태.
        - 이 때 `<DOMAIN>`은 이 서버의 도메인과 같아야 한다.
    - `Date` 값: 문제가 개정된 시각.
        - 이 시점 이전에 발행된 `문제 증표`들은 무효함을 선언한다.

페이지네이션은 가장 현재 시점에 가까운 에트라가 가장 먼저 위치하도록(`offset=0`일 때 첫 번째에 위치하도록) 한다.


### POST `/errata/query`

> [!NOTE]
> 한 번에 매우 매우 많은 개수의 에라타를 질의해야 한다면, `/errata` 페이지네이션을 이용해 서버의 모든 에라타를 다운로드하는 방법을 검토하라.

문제서버가 발행한 에라타 목록에서, 여러 문제 ID를 검색하여 나열한다.

한 번에 최대로 검색 가능한 문제 수는 서버가 결정한다.

#### 요청

```typescript
{
    "query": string[],
}
```

- `query`: 질의할 문제 식별자의 목록.
    - `<PROBLEM_ID>::<DOMAIN>` 형태.
    - 이 때 `<DOMAIN>`은 이 서버의 도메인과 같아야 한다.

#### 응답

```typescript
{
    [string]: Date | null,
}
```
- 키는 문제 식별자, 값은 문제의 개정 시각이다. 순서는 없다.
- 값이 `null`이라면, 다음 중 하나를 뜻한다.
    - 해당 문제에 발행된 에라타는 없다.
    - 없는 문제이거나, 이 서버가 발행한 문제가 아니다.

한 번에 검색할 수 있는 문제 수를 초과했다면, 서버는 요청 중 일부 질의만 선택하여 처리할 수 있다. 이 경우 선택되지 않은 질의는 응답에 포함되지 않는다. 즉, `undefined`이다.

따라서 클라이언트는 일부 질의에 대한 답변이 `undefined`라면 (응답에서 빠져 있다면), 이미 결과를 받은 질의를 제외한 뒤 재차 질의를 시도하거나, 다른 방법을 통해 에라타를 알아내야 한다.
