# DEPS - 데이터 정의

본 문서에서는 DEPS 프로토콜에서 사용되는 주체, 객체, 동작들을 정의한다.

## 1. 서명과 그 표현

DEPS는 서명에 관해서는 항상 Ed25519를 사용한다.

- 서명 과정에서 텍스트 인코딩은 항상 UTF-8을 사용한다.
- `Blake2b-512`는 512비트의 출력값을 내어놓는 BLAKE2B 암호학적 해시 함수이다.
- `base64url_nopad`는 [RFC 4648에서 정의된 "Base 64 Encoding with URL and Filename Safe Alphabet"](https://datatracker.ietf.org/doc/html/rfc4648#page-8)에 **Padding 문자(`=`)를 붙이지 않은 것**을 뜻한다.
- `JCS`는 [RFC8785: JSON Canonicalization Scheme](https://datatracker.ietf.org/doc/html/rfc8785)에서 정의하는 JSON 정규화 스키마이다.
    - `JCS(<데이터>)`는 `<데이터>`를 JCS 형식으로 정규화하여 나타낸 문자열이다.

### 1.1. 공개키의 텍스트 표현

**인코딩된 공개키**를 다음과 같이 정의한다.

```
base64url_nopad(<Ed25519 공개 키>)
```

Ed25519 공개 키의 크기는 32바이트이다. 따라서 위 표현은 43자가 된다.


### 1.2. 서명의 텍스트 표현

> [!NOTE]
> 아래의 정의에서 데이터를 `Blake2b-512`로 해시하는 것은 [minisign](https://jedisct1.github.io/minisign/)의 설계를 참고한 것이다.

**인코딩된 서명**을 다음과 같이 정의한다.

```
base64url_nopad(ed25519(Blake2b-512(<데이터>)))
```

Ed25519 서명은 64바이트이며, 따라서 위 표현은 86자가 된다.

### 1.3. 임의의 JSON 데이터에 대한 서명

> [!NOTE]
> JCS (RFC8785)는 어떤 JSON 오브젝트가 항상 같은 문자열으로 직렬화됨을 보장하기 위해 고안된 규격이다. **`data`를 JCS로 나타내는 과정을 생략하면 서명값이 달라질 수 있다!**

임의의 JSON 데이터 `data`를 서명하여 다음과 같이 나타낼 수 있다.

```typescript
type Signed<T> = {
    data: T & {
        // 이 데이터를 서명한 시각. 이 필드는 반드시 포함되어야 한다.
        signedAt: Date,
    },

    key: string,   // 서명에 사용된 인코딩된 공개키
    sign: string,  // 인코딩된 서명
}
```

서명 과정은 아래와 같다.

1. `data`를 JCS 형식으로 나타낸 `JCS(data)`를 얻는다.
2. `JCS(data)`를 `key`로 서명하여, 인코딩된 서명을 `sign`에 삽입한다.

검증 과정은 아래와 같다. 서명된 데이터를 받는 주체는 **반드시** 검증을 수행해야 한다.

1. `key`가 기대한 것과 같은지 확인한다.
2. `sign`이 `JCS(data)`를 `key`로 올바르게 서명한 것인지 확인한다.

### 1.4. PoW (Proof of Work)를 포함하는 서명

DEPS 프로토콜은 스팸이나 서버 자원 소진(DoS) 공격을 방지하기 위해, 특정 요청을 제출할 때 클라이언트에게 [PoW(작업 증명)](https://en.wikipedia.org/wiki/Proof_of_work)를 요구할 수 있다.

이 경우 클라이언트는 데이터를 제출하기 전 일정 수준의 연산(Brute-Forcing)을 직접 수행하여 올바른 값을 찾아내야 한다. 서버는 이를 통해 무분별한 요청을 억제할 수 있다.

서버는 각 요청의 종류에 따라 요구하는 **PoW 수준** 을 클라이언트와 사전에 공유한다. 공유 방식은 사용처에 따라 다르다. PoW 수준은 다음과 같이 정의된다.

- PoW 수준은 512보다 작은 양의 정수이다.
- PoW 수준이 *N*이라면, 최종 해시 결과값의 가장 왼쪽 *N*개 비트는 모두 `0`이어야 한다.

> [!NOTE]
> PoW 수준이 1 증가할 때마다, 클라이언트에게 요구되는 연산량은 약 2배씩 증가한다.
> 
> PoW 수준이 너무 높은 값이라면 조건을 만족하는 제출을 생성하는 것은 사실상 불가능해지게 된다. 클라이언트는 이러한 경우를 고려해야 한다.

**PoW를 포함하는 서명**의 타입은 아래와 같이 정의한다.

```typescript
type SignedProofed<T> = {
    // 이하 3개는 Signed<T>의 정의와 같음
    data: T & { signedAt: Date, },
    key: string,
    sign: string,

    // Proof of Work를 만족하기 위해 포함되는 임의의 문자열
    pow: string,
}
```

- `pow` 필드는 ASCII 문자로만 이루어진 길이 64바이트 이하의 문자열이다.

PoW 계산에 필요한 값과 각 연산 과정은 아래와 같다. **이 때 `||`은 문자열 합치기 연산을 뜻한다.**

- **`CONTENT_HASH`** 는 `Blake2b-512( JCS(data) || key || sign )`를 연산한 값이다.
- **`FINAL_HASH`** 는 `Blake2b-512( CONTENT_HASH || pow )`를 연산한 값이다.

> [!NOTE]
> Brute-forcing 구현 시 `pow` 값을 무작위로 변경하며 올바른 값을 찾는 것보다는 값을 조금씩 증가시키며 찾는 것이 더 빠르다고 알려져 있다.

PoW를 포함하는 서명의 작성 과정은 다음과 같다.

1. 먼저 [1.3. 임의의 JSON 데이터에 대한 서명](#13-임의의-JSON-데이터에-대한-서명) 과정을 수행한 뒤, `CONTENT_HASH` 값을 구한다.
2. 다음 조건을 만족할 때까지 `pow` 값을 적절히 변경하며 Brute-Forcing을 수행한다.
    - 서버가 정한 `PoW 수준` 값을 *N*이라 할 때:
    - `FINAL_HASH`의 값에서 가장 왼쪽에 위치한 *N*개의 비트는 모두 `0`이어야 한다.

PoW를 포함하는 서명의 검증 과정은 다음과 같다.

1. 제출이 다음을 만족하여야 한다.
    - 서버가 정한 `PoW 수준` 값을 *N*이라 할 때:
    - `FINAL_HASH`를 연산한 값에서 가장 왼쪽에 위치한 *N*개의 비트는 모두 `0`이어야 한다.
2. (1)을 만족한다면, [1.3. 임의의 JSON 데이터에 대한 서명](#13-임의의-JSON-데이터에-대한-서명) 에서 정의된 서명 검증을 수행한다.


## 2. 식별자의 형식

DEPS에서 사용되는 식별자들은 다음과 같은 형태를 가진다.

### 2.1. 아이덴티티 식별자

하나의 아이덴티티를 고유하게 결정하는 불변한 값이다. 앞에 `::`가 붙은 `인코딩된 공개 키`로 나타낸다.

```
::[인코딩된 공개키]

::Cd3FRCGA9i83mlvQbO4IzT51TGPpvAucNSCh1CzM0QT
```

### 2.2. 핸들네임

> [!NOTE]
> 핸들네임은 안정된 식별자가 아니다! 핸들네임은 사용자 경험을 개선하기 위한 별명과 같은 용도로만 사용되어야 한다.

홈서버에서 어떤 아이덴티티에게 부여한 별명이다. 홈서버로의 요청을 통해 아이덴티티와 상호 변환할 수 있다.

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
    - 단, 하나의 홈서버는 각 아이덴티티에게 하나의 핸들네임만 붙일 수 있다.

### 2.3. 문제 식별자

어떤 문제서버의 특정한 문제를 지칭하는 식별자.

```
#[서버 내 문제 ID]::[서버 도메인]

#1000::ps.example.org
#AxcqVdCzQN::solutions.home
```

- [서버 내 문제 ID]는 정규식 `[a-zA-Z0-9-]{1,32}`으로 나타낼 수 있어야 한다.

## 3. 객체의 타입

### 3.1. 아이덴티티

아이덴티티는 아래에 정의된 `Identity` 타입의 데이터이다.

```typescript
type IdentityInfo = {
    nickname: string,   // 닉네임
    bio: string?,       // 소개글
    avatarUrl: string?  // 프로필 사진의 URL
    preferredHandle: string?  // 선호하는 핸들네임으로 표시한 값
    signedAt: Date,     // 이 오브젝트를 서명한 날짜
    
    // 추가로 임의의 필드가 포함될 수 있다.
    // 자세한 내용은 03-extensions.md 문서를 참고할 것.
}

// Signed<IdentityInfo> 타입과 같음
type Identity = {
    // 아이덴티티 식별자. Base64로 인코딩된 Ed25519 공개키.
    key: string,

    // 위에서 정의된 IdentityInfo.
    data: IdentityInfo,

    // info 문자열을 key로 서명한 값.
    // 즉, 아이덴티티의 서명은 아이덴티티 스스로의 키로 한다.
    sign: string,
}
```

- `preferredHandle`은 이 아이덴티티가 불리기를 원하는 핸들네임이다.
    - 이 핸들네임이 실제로 해당 아이덴티티를 가리키는지 여부는 **검증되지 않는다**.
    - 이를 알기 위해서는 직접 해당 홈서버로 요청을 보내야 한다.

### 3.2. 문제증표

문제증표는 한 아이덴티티가 어떤 문제서버의 특정 문제를 맞추었음을 증명하는 데이터이다.

```ts
type SolveInfo = {
    // 문제를 푼 주체의 아이덴티티 식별자.
    identity: string,

    // 문제 ID. `#ProblemId::domain.name` 형태여야 한다.
    problemId: string,

    // 점수. 구간 [0.0, 1.0]에 속하는 실수값.
    // 1.0일 경우 AC, 1.0 미만일 경우 PC 취급이다.
    score: number,

    // 문제를 푼 시각. 변하면 안 된다.
    signedAt: Date,
}

type SolveCertificate = {
    // key는 해당 문제를 소유한 문제서버의 공개 키이다.
    key: string,
    data: SolveInfo,
    sign: string,
}
```
