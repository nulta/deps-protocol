# DEPS - API 규격

## 기본 원칙

DEPS에서 정의하는 모든 API는 **반드시** HTTPS만을 사용한다.

DEPS에서 정의하는 모든 API는 `/_deps/`를 기본 경로로 사용한다. API의 경로는 **반드시** 도메인의 루트에서 시작해야 한다.

    https://some.example.org/_deps/...        // OK
    http://203.0.113.123/_deps/...            // X, HTTPS를 사용해야 함
    https://example.com/homeserver/_deps/...  // X, 도메인 루트에 존재해야 함

- 홈 서버의 모든 API는 `/_deps/home/`를 베이스로 한다.
- 문제 서버의 모든 API는 `/_deps/judge/`를 베이스로 한다.
- 하나의 도메인 아래에 홈 서버와 문제 서버가 같이 있을 수도 있다.
    - 이 경우 둘의 공개 키는 같을 수도, 다를 수도 있다.
    - 어떤 도메인을 해석할 때 이를 홈서버로 해석할 것인지, 문제서버로 해석할 것인지는, 문맥에 따라 달라지게 된다

DEPS에서 정의하는 모든 API는, 달리 정의하지 않는 한, JSON을 사용하여 통신한다. 서버는 JSON 데이터를 전송할 때 `Content-Type` 헤더를 `application/json`으로 설정하여야 한다.

### 오류 응답

DEPS에서 정의하는 API는 오류 상황이 아닐 때 `2xx` 계열의 HTTP 상태 코드를 반환하여야 한다.

만약 `4xx` 또는 `5xx` 을 반환하는 경우, 그 body는 달리 정의하지 않는 한 다음과 같이 한다. 이를 `오류 응답`이라 한다.

```typescript
{
    "error": string,
    // 서버는 그 밖에 다른 필드를 포함할 수 있음
}
```

> [!NOTE]
> `error`의 내용은 영어로 이해하기 쉽게 작성하는 것을 권장한다.

- `error`의 내용은 서버에서 임의로 작성한다.

## 홈 서버

### GET `/_deps/home/info`

홈서버의 기본 정보를 받아온다.

#### 응답
```typescript
{
    "key": string
}
```

- key: 이 서버의 **인코딩된 공개 키**


### GET `/_deps/home/identity/[identity]`

홈서버에 등록된 아이덴티티의 정보를 받아온다.

#### 요청
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

#### 예외
- 이 홈서버에 등록되지 않은 아이덴티티라면, `404 Not Found`


### GET `/_deps/home/identity/[identity]/handle`

홈서버에 등록된 아이덴티티의 핸들네임을 받아온다.

#### 요청
- `[identity]`: `아이덴티티 식별자`

#### 응답
```typescript
{
    "identity": string,
    "handle": string | null,
}
```

- `identity`: 아이덴티티 식별자. `[identity]`와 동일한 값.
- `handle`: 이 서버에서 해당 아이덴티티에게 부여된 핸들네임. null일 수 있음.

#### 예외
- 이 홈서버에 등록되지 않은 아이덴티티라면, `404 Not Found`


### GET `/_deps/home/handle/[handle]`

#### 요청
- `[handle]`: 올바른 핸들네임
    - 단, 이 서버에서 발급한 핸들네임만 쿼리할 수 있다.

#### 응답
```typescript
// "데이터 정의" 문서에서 정의된 "Identity" 정의와 같음
{
    "key": string,
    "data": IdentityInfo,
    "sign": string
}
```


### GET `/_deps/home/`

#### 요청
- `[handle]`: 올바른 `핸들네임`
    - 단, 이 서버에서 발급한 핸들네임만 쿼리할 수 있다.

#### 응답
```typescript
// "데이터 정의" 문서에서 정의된 "Identity" 정의와 같음
{
    "key": string,
    "data": IdentityInfo,
    "sign": string
}
```


## 문제서버

### GET `/_deps/judge/info`

#### 응답
```typescript
{
    "key": string
}
```

- `key`: Base64로 인코딩된 이 서버의 Ed25519 공개 키

### GET `/_deps/judge/problem?offset=..&limit=..`

문제 목록을 페이지네이션하여 받아온다.

- `offset`: **선택적.** 문제 목록의 맨 앞에서 이 개수만큼의 데이터를 스킵한다.
    - 없으면 0으로 가정한다.
- `limit`: 받아올 문제 개수의 상한값

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
- `id`: 문제의 id
- `title`: 문제의 제목
- `totalCount`: 서버에 존재하는 문제 개수

잘 정렬된 전체 문제 목록 중에서, 일부를 반환한다. 

### GET `/_deps/judge/problem/[problemId]`

#### 요청
- `[problemId]`: 문제의 id

#### 응답

```typescript
{
    "id": string,
    "title": string,
    "content": string
}
```
- `id`: 문제의 id
- `title`: 문제의 제목
- `content`: Markdown 형식으로 쓰여진 문제의 본문

### POST `/_deps/judge/problem/[problemId]/submit`

#### 요청
- `[problemId]`: 문제 id

##### `SubmissionFile` 정의
```ts
type SubmissionFile = {
    language: string,
    name: string | "",
    content: string,
}
```
- `language`: 프로그래밍 언어 종류
    - 문제 서버에서 채점 가능한 언어 중 한 종류여야 한다.
- `name`: 파일명. 빈 문자열일 수 있다.
- `code`: 소스 코드

##### body
```typescript
{
    key: string,
    data: SubmissionFile[]
    sign: string,
}
```

"데이터 정의" 문서의 "임의의 JSON 데이터에 대한 서명"과 같은 규격을 따른다.

- `key`: 제출자의 `아이덴티티 식별자`
- `data`: 제출할 파일 목록
- `sign`: data에 대한 서명. 자세한 사항은 정의 문서 참조.


#### 응답
```typescript
{
    "submissionId": int
}
```
- `submissionId`: 채점 번호

### GET `/_deps/judge/errata?offset=..&limit=..`

#### 응답
```typescript
{
    "data": {
        "problemId": string,
        "invalidBefore": Date,
    }[],
    "totalCount": number;
}
```

- data
    - `problemId`: 문제 ID
    - `invalidBefore`: 문제가 개정된 시각.
        - 이 시점 이전에 발행된 `문제 증표`들은 무효함을 선언한다.

data는 `invalidBefore`의 내림차순으로 정렬되어 있어야 한다.

### GET `/_deps/judge/submission/[submissionId]`

#### 요청
- `[submissionId]`: 채점 번호

#### 응답
```typescript
{
    "status": "waiting" | "judging" | "finished",
    "verdict": "AC" | "PC" | "WA" | "TLE" | "MLE" | "RE" | "CE" | "UE" | undefined,
    "additionalInfo": string | undefined,
}
```
- `status`: 채점의 현재 상태
    - `waiting`: 대기중
    - `judging`: 진행중
    - `finished`: 채점 완료
- `verdict`: 채점의 결과
    - `AC`: 전체 정답
    - `PC`: 부분 정답
        - 문제 중 부분 점수가 존재하는 문제에만 해당
    - `WA`: 오답
    - `TLE`: 시간 제한 초과
    - `MLE`: 메모리 제한 초과
    - `RE`: 런타임 에러
    - `CE`: 컴파일 에러
    - `UE`: 정의되지 않은 에러
- `additionalInfo`
    - `RE`, `CE`, `UE`시 발생하는 에러 메세지를 포함해야 한다.
    - 기타 정보(컴파일 시 경고 메세지 등)을 포함할 수 있다.

### GET `/_deps/judge/submission/[submissionId]/cert`

#### 요청
- `[submissionId]`: 채점 번호

#### 응답
```typescript
{
    "key": string,
    "data": {
        "identity": string,
        "problemId": string,
        "score": number,
        "signedAt": Date,
    },
    "sign": string
}
```

- `key`: 문제서버의 공개키
- `identity`: 아이덴티티 식별자
- `problemId`: 문제 id. `#ProblemId::domain.name` 형태.
- `score`
    - `[0, 1]` 범위의 실수이다.
    - `1.0`일 때는 AC, `[0, 1)`일 때는 PC로 간주한다.
- `signedAt`: 문제를 푼 시간
- `sign`: data를 key로 서명한 값. definitions 문서 참고.
