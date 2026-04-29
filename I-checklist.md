# DEPS - API 체크리스트

## 공통사항
- [ ] HTTPS 프로토콜 사용 여부
- [ ] 도메인 루트의 `/_deps/` 경로 시작 여부
- [ ] JSON 통신 및 Content-Type 헤더 설정
- [ ] 4xx/5xx 오류 응답 규격(error, reason) 준수
- [ ] PoW(Proof of Work) 검증 및 오류 응답 처리
- [ ] 공통 페이지네이션(offset, limit, totalCount) 규격 준수

## 홈 서버 엔드포인트 (`/_deps/home/`)
- [ ] **GET** `/info` (서버 정보 및 공개 키)
- [ ] **GET** `/identities/[identity]` (아이덴티티 정보 조회)
- [ ] **POST** `/identities/[identity]` (아이덴티티 정보 수정)
- [ ] **GET** `/identities/[identity]/solves` (문제증표 목록 조회)
- [ ] **PUT** `/identities/[identity]/solves` (문제증표 추가)
- [ ] **GET** `/identities/[identity]/handle` (아이덴티티 핸들 조회)
- [ ] **GET** `/handles/[handle]` (핸들 기준 아이덴티티 조회)

## 문제 서버 엔드포인트 (`/_deps/judge/`)
- [ ] **GET** `/info` (서버 정보 및 공개 키)
- [ ] **GET** `/problems` (문제 목록 조회)
- [ ] **GET** `/problems/[problemId]` (특정 문제 상세 정보)
- [ ] **POST** `/problems/[problemId]/submit` (답안 제출)
- [ ] **GET** `/submissions/[submissionId]` (채점 결과/상태 조회)
- [ ] **GET** `/errata` (에라타 목록 조회)
- [ ] **POST** `/errata/query` (특정 문제 에라타 질의)