# DEPS - 데이터 정의 및 통신규약

> [!WARNING]
> 이 문서는 아직 작성 중입니다.

### 서명과 그 표현

- 서명을 위한 텍스트 인코딩은 항상 UTF-8을 사용한다.
- `base64_urlsafe`는 [RFC 4648에서 정의된 "Base 64 Encoding with URL and Filename Safe Alphabet"](https://datatracker.ietf.org/doc/html/rfc4648#page-8)을 뜻한다.

#### 공개키의 텍스트 표현

본 프로토콜에서 "인코딩된 공개키"라 함은, 다음을 일컫는다.

```
base64_urlsafe(<Ed25519 공개 키>)
```

Ed25519 공개 키의 크기는 32바이트이다. 따라서 위 표현은 43자가 된다.


#### 서명의 텍스트 표현

본 프로토콜에서 "인코딩된 서명"이라 함은, 다음을 일컫는다.

```
base64_urlsafe(ed25519(Blake2b-512(<데이터>)))
```

Ed25519 서명은 64바이트이며, 따라서 위 표현은 86자가 된다.

#### 임의의 JSON 데이터에 대한 서명

임의의 JSON 데이터를 다음과 같이 서명할 수 있다.

```ts
{
    id: "<>"
}
```

이 때, 다음


### 식별자의 형식

DEPS에서 사용되는 식별자들은 다음과 같은 형태를 가진다.

#### 아이덴티티 식별자

앞에 `::`가 붙은 Ed25519 공개 키. 하나의 아이덴티티를 고유하게 결정하는 값이다.

```
::[인코딩된 공개키]

::Cd3FRCGA9i83mlvQbO4IzT51TGPpvAucNSCh1CzM0QT
```

#### 핸들네임

홈서버에서 어떤 아이덴티티에게 부여한 읽기 쉬운 별명.

```
@[유저네임]::[홈서버 도메인]

@alice::example.org
@Neo-Vim::pshome.example.kr
```

- 핸들네임은 대소문자를 구분하지 않는다.
- [유저네임] 부분은 정규식 `[a-zA-Z0-9-]{2,18}` 으로 나타낼 수 있어야 한다.

아이덴티티와 달리 핸들네임의 연결은 홈서버에 종속적이다. 따라서 핸들네임은 프로토콜 상에서 다음과 같은 기묘한 특징을 지니게 된다.

- 핸들네임이 없는 아이덴티티가 존재할 수 있다.
- 핸들네임은 불변이 아니다. 유저의 핸들네임은 변경될 수 있다.
- 여러 홈서버가 하나의 아이덴티티에게 서로 다른 핸들네임을 붙일 가능성이 있다.

#### 문제 식별자

어떤 문제서버의 특정한 문제를 지칭하는 식별자.

```
#[서버 내 문제 ID]::[서버 도메인]

#1000::ps.example.org
#AxcqVdCzQN::solutions.home
```


### 아이덴티티

아이덴티티는 아래에 정의된 `Identity` 타입의 데이터를 JSON으로 나타낸 문자열이다.

> [!NOTE]
> 모든 `Date` 타입은 typescript 변환 시 ISO 8601 타임스탬프를 담은 문자열(`"2026-04-16T10:20:33.123Z"`) 으로 직렬화한다.



```ts
type IdentityInfo = {
    nickname: string,    // 닉네임
    bio: string?,        // 소개글
    avatar_url: string?  // 프로필 사진의 URL
    signed_at: Date,     // 이 오브젝트를 서명한 날짜 (Identity 참조)
}

type Identity = {
    // 아이덴티티 식별자. Base64로 인코딩된 Ed25519 공개키.
    identity: string,

    // 위의 IdentityInfo 타입의 오브젝트를 JSON으로 직렬화한 문자열.
    info: string,

    // info 문자열을 identity에 대응하는 비밀 키로 서명한 값.
    signature: string,
}
```




### 문제증표

```ts
type SolveCertificate = {
    issuer: string,        // 문제서버 공개키
    user: string,          // 유저 공개키
    problem_id: string,
    verdict: "AC",
    score: 100,
    language: "C++23",
    solved_at: Date,
    nonce: string
    sign: string
}
```


## 4. 구현사항

### 홈 서버

- ```GET /api/deps/key```
    ```typescript
    {
        "key": string
    }
    ```
    key: Base64로 인코딩된 이 서버의 Ed25519 공개 키

- ```GET /api/deps/identity/{handleName}```
    handleName: 위의 handleName의 규격에 따르는 문자열
    ```typescript
    {
        "identity": string,
        "info": string,
        "signature": string
    }
    ```
    identity: 아이덴티티 식별자. Base64로 인코딩된 Ed25519 공개 키
    info: [IdentityInfo](###아이덴티티) 타입의 오브젝트를 typescript으로 직렬화한 문자열
    signature: info 문자열을 identity로 서명한 값
### 문제서버

- ```GET /api/deps/key```
    ```typescript
    {
        "key": string
    }
    ```
    key: Base64로 인코딩된 이 서버의 Ed25519 공개 키

#### 문제 정보 제공
- ```GET /api/deps/problems```
    ```typescript
    {
        "problems": [
            {
                "id": int, 
                "title": string
            },
            ...
        ]
    }
    ```
    id: 문제의 고유 id
    title: 문제의 제목
- ```GET /api/deps/problem/{problemId}```
    problemId: 문제의 id
    - response
        ```typescript
        {
            "id": int,
            "title": string,
            "content": string
        }
        ```
        id: 문제의 id
        title: 문제의 제목
        content: MD 형식으로 쓰여진 문제의 본문

#### 문제 채점
- ```POST /api/deps/submit/{problemId}```
    problemId: 문제 id
    - body
        ```typescript
        {
            "language": string,
            "code": string
        }
        ```
        language: 프로그래밍 언어 종류(C99, C++23, Java8, ..)
        code: 소스 코드
    - response
        ```typescript
        {
            "submissionId": int
        }
        ```
        submissionId: 채점 번호

- ```GET /api/deps/judge-status/{submissionId}```
    submissionId: 채점 번호
    - response
        ```typescript
        {
            "status": "waiting" | "judging" | "finished",
            "result": "AC" | "WA" | "TLE" | "MLE" | "RE" | undefined,
            "additionalInfo": string | undefined,
        }
        ```

### 랭킹 서버
