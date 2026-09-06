# NecrassRs 설계 방향 조사

Pothos · Hot Chocolate · Tonic · Rust GraphQL 구현체 비교

- 작성일·자료 확인일: 2026-09-05
- 대상: NecrassRs 설계·구현 담당자
- 목적: 기존 실험의 불편을 재평가하고, 다음 프로토타입에서 검증할 설계 결정을 정한다.
- 논의 반영: 사용자가 첫 해결 대상으로 스키마·타입 선언의 반복을 지정하고, 공개 GraphQL 계약을 SDL에서 편집하는 방향을 선택했다. 구체적인 Rust 바인딩·코드 생성 API는 검증할 설계 항목이다.
- 범위: 제공된 2026년 7–9월 Discord 대화, 로컬 구현·개발일지, 각 프로젝트의 공식 문서·소스. 성능 순위와 전체 기능 목록 조사는 제외한다.
- 근거 표기: 제품 동작은 출처를 붙이고, NecrassRs의 권고안은 설계 제안으로 구분한다. 별도 날짜가 없는 웹 문서는 정확한 수정일을 확정하지 않았으며 위 확인일의 문서 상태를 사용한다.

## 1. 권고 방향

**NecrassRs의 첫 목표는 GraphQL 타입·필드·nullability의 반복 선언을 줄이는 것이다. 공개 계약은 SDL을 기준으로 삼고, Rust 표현과 기본 필드 연결, 타입·root 등록을 생성하는 개발 경험을 우선 검증한다.**

확정된 방향은 선언의 반복 감소와 SDL을 통한 공개 계약 관리다. SDL에서 Rust 바인딩을 생성하는 구체적인 방식을 실험한다. SDL와 사용자 Rust 구조체에 같은 타입 정보를 다시 쓰게 한다면 목표를 달성하지 못한 것이다. 관계 로딩·배칭·SQL 자동화는 이 첫 실험의 통과 이후에 평가한다.

Pothos에서 참고할 것은 타입 정보를 가진 스키마 선언과 데이터 요구사항의 연결이다. Hot Chocolate에서는 서비스·요청 문맥·DataLoader·데이터 조회 조건을 한 흐름으로 연결하는 방법을 참고할 수 있다. Tonic은 계약에서 생성되는 코드와 사용자가 작성하는 서비스 구현을 분리하는 방법론을 제공한다. 각 근거와 적용 한계는 3–5절에서 설명한다.

기존 일지의 **구조화된 스키마 표현과 계약 중심 코드 생성**이라는 방향은 유지할 가치가 있다. 다만 자체 실행기, 범용 SQL 생성기, 독자 스키마 언어까지 동시에 만들어야 한다는 결론은 현재 근거에서 나오지 않는다. 먼저 기존 실행기 위에서 해당 개발 경험을 검증하고, 재사용 경로로 해결되지 않는 제약이 재현될 때 실행기 개발 범위를 결정하는 편이 타당하다.

초기 사용자는 **GraphQL 계약과 Rust 구현 사이에서 타입·필드·등록 정보를 반복 작성하는 Rust 개발자**로 가정한다. 첫 실험은 DB 없이도 수행할 수 있어야 한다. ORM 메타데이터를 중심으로 API를 자동 생성하는 방향은 별도 선택이며, 그 단계에서는 Seaography도 비교한다.

## 2. 대화록과 현재 실험을 다시 읽으면

### 비교 대상의 계층부터 맞춰야 한다

현재 실험은 Pothos 단독과 async-graphql 단독의 비교가 아니다.

| 로컬 실험 | 실제 조합 | 비교에 미치는 영향 |
|---|---|---|
| TypeScript | Pothos + Drizzle 관계 플러그인 + Relay 플러그인 + Yoga + Node HTTP | 스키마 작성 외에 ORM 관계·페이지네이션 통합의 효과가 포함된다. |
| Rust | async-graphql + SQLx + Axum | 관계 SQL, 결과 매핑, 페이지네이션 조건을 상당 부분 직접 작성한다. |

Pothos 실험은 `builder.drizzleNode`, `t.relation`, `t.relatedConnection`을 사용한다. Rust의 `Issue.assignee`, `Issue.reporter`, `Comment.author`는 각각 SQL을 실행한다. 따라서 현재 코드량 차이를 모두 언어나 GraphQL 실행기의 차이로 설명할 수 없다. [로컬 Pothos builder](../../necrassrs-lab/services/api-pothos-drizzle/src/graphql/builder.ts), [Pothos Issue](../../necrassrs-lab/services/api-pothos-drizzle/src/graphql/issue.ts), [Rust Issue](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/issue.rs), [Rust Comment](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/comment.rs)

조사 시점의 `necrassrs/src/main.rs`는 Hello World이며 Cargo 의존성도 비어 있다. IR과 생성기는 아직 설계 방향이다. 실험 저장소의 확인 기준 커밋은 `3aeaa54`, 일지는 `67d4e0d`다. 구현 파일은 수정하지 않고 읽었다. [NecrassRs main](../../necrassrs/src/main.rs), [Cargo manifest](../../necrassrs/Cargo.toml), [기존 종합 일지](../20260726.md)

### 대화에서 나온 불편의 분류

| 대화의 문제의식 | 조사 결과 | 설계에 반영할 내용 |
|---|---|---|
| 단순 필드까지 getter를 반복 작성한다. | async-graphql의 `SimpleObject`와 `ComplexObject` 조합으로 줄일 수 있다. | 입문 경로에서 기본 필드와 계산·관계 필드의 조합을 바로 보여준다. |
| 관계 필드마다 SQLx를 다시 호출한다. | scalar getter 문제와 별개다. DataLoader는 있지만 배치 함수와 관계의 연결은 작성해야 한다. | 여러 관계에서 동일한 로딩 함수를 재사용하는 기본 경로를 만든다. |
| 헤더 하나 때문에 별도 handler를 만든다. | Pothos도 서버와 context factory가 필요하다. 차이는 통합 작업의 노출 방식이다. | 공식 기본 어댑터와 요청 context 생성 지점을 제공한다. |
| `QueryRoot`가 명세 위반처럼 보인다. | 명시적 root mapping이 있으면 유효하다. | `Query` 등 익숙한 기본값을 제공하고 사용자 이름도 허용한다. |
| 인터랙티브한 Rust 문서가 아쉽다. | 실행 가능한 코드 문서는 이미 가능하다. 서버·DB까지 자유롭게 실행하는 환경은 별도 문제다. | 코드→SDL→요청→응답·조회 흐름을 단계적으로 보여준다. |

첫 두 항목은 [async-graphql — SimpleObject](https://async-graphql.github.io/async-graphql/en/define_simple_object.html)와 [Optimizing N+1 queries](https://async-graphql.github.io/async-graphql/en/dataloader.html), 서버 경계는 [Pothos — Using Context](https://pothos-graphql.dev/docs/guide/context), root 이름은 [GraphQL September 2025 — Root Operation Types](https://spec.graphql.org/September2025/#sec-Root-Operation-Types), 문서 가능 범위는 [mdBook — Editor](https://rust-lang.github.io/mdBook/format/theme/editor.html)를 근거로 분류했다.

`SimpleObject`에 대한 7월 27일 지적은 정확하지만, 이를 발견했다고 관계 조회의 불편까지 사라지는 것은 아니다. 반대로 그 지적을 반영하지 않은 코드로 새로운 프레임워크의 생산성 우위를 주장해서도 안 된다. 현재 Rust 코드도 mutation payload에는 이미 `SimpleObject`를 사용하고, 일반 `User`에는 수동 getter를 사용한다. [User](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/user.rs), [UpdateIssuePayload](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/update_issue.rs)

### 기존 일지와 현재 소스의 차이

7월 26일 종합 일지에는 댓글 관계와 일부 필터·정렬이 미구현으로 남아 있지만, 현재 소스에는 `Issue.comments`, `Comment.author`, `Project.issues`의 필터·양방향 정렬·역방향 페이지네이션 코드가 있다. 이는 구현의 존재 확인이며, 동작이 모두 검증되었다는 뜻은 아니다. [Rust Project](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/project.rs), [Rust Issue](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/issue.rs)

동등성 검증 전에 정리할 차이도 확인했다. Pothos의 `Issue.status`는 `exposeString`이고 Rust는 `IssueStatus` enum을 반환한다. 또한 Rust `Project.issues`는 `assigneeId` 문자열을 SQL 조건에 직접 바인딩하는 반면, Pothos는 global ID에서 추출한 내부 ID를 사용한다. Rust의 ID 인코딩·디코딩 코드와 대조하면 후자는 불일치 후보다. 두 서버를 실행해 재현하지는 않았으므로, 프레임워크의 결함이 아닌 **비교 기준선의 확인 과제**로 다룬다. [Pothos Project](../../necrassrs-lab/services/api-pothos-drizzle/src/graphql/project.ts), [Rust Project](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/project.rs), [Rust global ID](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/global_id.rs)

## 3. Pothos: 스키마와 실제 데이터 사이의 연결을 선언한다

### 공개 타입과 backing data를 구분한다

`objectRef<T>`의 `T`는 resolver가 받거나 반환하는 실제 데이터의 타입이다. 그 데이터의 모든 속성이 자동으로 GraphQL 필드가 되는 것은 아니다. 필드를 명시적으로 공개하고, 이름을 바꾸거나 계산 필드를 추가할 수 있다. [Pothos — Objects](https://pothos-graphql.dev/docs/guide/objects)

이 분리는 NecrassRs에 직접 유용하다. DB row, 도메인 객체, 공개 GraphQL 타입이 반드시 같은 모양일 필요는 없다. 내부 키와 권한 판정용 정보를 보유하면서도 공개 필드는 제한할 수 있어야 한다. 동시에 기존 객체를 연결할 때 불필요하게 동일한 구조체를 한 번 더 작성하게 만드는지도 평가해야 한다.

Pothos의 `SchemaTypes`와 개별 `Ref`는 전역 설정과 모듈별 타입 정보를 나눈다. 순환 참조 처리를 위해 스키마 구성을 `toSchema()`까지 지연하는 부분도 있다. 장점은 모듈화지만, 순환 import와 추론의 모든 문제가 자동으로 없어지는 구조는 아니다. [Pothos — Design](https://pothos-graphql.dev/docs/design), [Circular References](https://pothos-graphql.dev/docs/guide/circular-references)

### 관계 플러그인의 핵심은 메타데이터와 query 연결이다

Drizzle 플러그인은 Drizzle에 정의된 관계를 이용한다. 타입·필드의 `select`로 필요한 column과 관계를 선언하고, `drizzleField`의 resolver가 전달받은 `query` 함수를 실제 DB 조회에 적용한다. 이때 GraphQL의 하위 선택과 데이터 조회 옵션이 연결된다. 현재 공식 문서는 RQBV2 기반, beta 의존, 기능·API 변경 가능성을 명시한다. 로컬의 Drizzle `1.0.0-rc.4` 고정값과 현행 문서를 동일한 API 버전으로 간주해서는 안 된다. [Pothos — Drizzle plugin](https://pothos-graphql.dev/docs/plugins/drizzle), [로컬 package.json](../../necrassrs-lab/services/api-pothos-drizzle/package.json)

Prisma 플러그인도 공개 GraphQL 타입을 DB 모델과 무조건 동일하게 만들지는 않는다. 관계 요구를 상위 조회에 전달하지만, preload할 수 없는 경로와 서로 다른 관계 인자 등에는 추가 조회가 필요하다. 관계 query 설정은 부모 데이터가 로드되기 전에 평가되므로 부모 객체를 사용할 수 없다는 제약도 있다. 따라서 자동화는 **항상 한 번의 SQL로 처리한다는 보장**이 아니다. [Pothos — Prisma plugin](https://pothos-graphql.dev/docs/plugins/prisma), [Prisma Relations](https://pothos-graphql.dev/docs/plugins/prisma/relations)

NecrassRs에 필요한 질문은 `t.relation`과 비슷한 문법을 만들지 여부보다 **관계 키·조회 함수·필요 데이터·권한 범위 정보를 어디에 선언할 것인가**다. 현재 SQLx 실험에는 Drizzle 플러그인에 대응하는 관계 메타데이터 계층이 없다.

### 플러그인과 서버의 경계를 구분한다

Pothos 플러그인은 스키마 구성 단계의 hook과 resolver wrapper를 구분한다. 이는 등록 시 준비할 작업과 매 요청마다 수행할 작업을 구분하는 좋은 참고점이다. 다만 TypeScript의 interface 확장과 prototype 기반 구현을 Rust에서 그대로 재현할 필요는 없다. [Pothos — Writing plugins](https://pothos-graphql.dev/docs/guide/writing-plugins)

Pothos는 표준 graphql.js 스키마를 생성하며, 공식 시작 예시는 Yoga와 Node HTTP 서버에 연결한다. 인증 정보도 Yoga context factory에서 만든다. 따라서 NecrassRs는 서버 의존성을 없애는 목표보다 **서버에서 인증 주체를 한 번 구성하고 resolver가 일관되게 받는 경험**을 목표로 삼는 편이 적절하다. [Pothos — Guide](https://pothos-graphql.dev/docs/guide), [Using Context](https://pothos-graphql.dev/docs/guide/context)

### nullability는 버전까지 확인해야 한다

Pothos v4는 출력 필드 기본값을 nullable로 바꿨다. 이전 v3는 non-null이 기본이었다. 로컬 설치된 core 4.13.0의 생성자도 이 규칙과 일치한다. Rust의 `T`와 `Option<T>`에 따른 기본 매핑과는 작성 경험이 다르지만, 어느 기본값을 선택하든 최종 계약을 명확하게 표현하고 검증하는 것이 핵심이다. [Pothos — v4 migration](https://pothos-graphql.dev/docs/migrations/v4#default-field-nullability)

## 4. Hot Chocolate: 요청, 서비스, 데이터 실행을 통합한다

여기서는 2026-09-05에 열람한 **v16 계열 현행 설계**를 참고한다. 현행 문서에는 후속 preview 기능도 반영될 수 있으므로, 모든 예제가 특정 stable minor에서 실행된다고 보증하지 않는다. 버전 변화는 공식 migration 문서를 기준으로 읽었다. [ChilliCream — Hot Chocolate 16 Migration Guide, 2026-08-28](https://chillicream.com/docs/hotchocolate/migrating/migrate-from-15-to-16)

### 평범한 서비스 함수가 스키마에 연결된다

Hot Chocolate는 클래스·메서드·속성을 읽는 source generator 기반 방식과, `ObjectType<T>`·descriptor로 공개 타입을 따로 설정하는 방식을 제공한다. resolver 인자에 필요한 서비스를 전달하며 서비스 수명도 실행 방식에 맞춰 관리한다. [ChilliCream — Hot Chocolate 소개, 2026-07-01](https://chillicream.com/docs/hotchocolate), [Dependency Injection, 2026-08-30](https://chillicream.com/docs/hotchocolate/resolvers/dependency-injection)

참고할 핵심은 큰 DI 컨테이너 자체가 아니다. 사용자가 서비스 호출을 resolver에 연결하기 위해 몇 단계의 등록과 조회를 반복하는지, 필요한 의존성을 어느 시점에 확인하는지가 중요하다. Rust에서는 구체적인 context 구조체와 함수 인자로 먼저 검증할 수 있다.

### 선택 정보가 서비스 경계를 넘어간다

`QueryContext<T>`는 선택 필드에 대응하는 selector, filter predicate, sorting을 묶는다. 이를 서비스나 DataLoader에 전달하고 `.With(query)`로 `IQueryable`에 적용할 수 있다. 공식 문서는 filter→sort→project 순서로 적용된다고 설명한다. 이는 GraphQL resolver 내부에 DB query 조립을 모두 몰아넣지 않도록 돕는다. [ChilliCream — Projections, 2026-08-28](https://chillicream.com/docs/hotchocolate/fetching-data/projections)

이 기능은 .NET의 식 트리와 데이터 공급자에 기반한다. `IQueryable`·`IExecutable` 데이터 접근 통합을 임의의 SQL 문자열에 대한 최적화로 일반화해서는 안 된다. [ChilliCream — Fetching Data, 2026-07-01](https://chillicream.com/docs/hotchocolate/fetching-data)

또한 클라이언트가 요청하지 않은 키도 서버에는 필요하다. Hot Chocolate의 `Parent(requires: ...)`, `Include()`는 관계 조회나 결과 매핑에 필요한 데이터를 남기는 방법이다. NecrassRs에서도 projection을 도입한다면 외래키·권한 판정용 owner ID·추상 타입 판별값을 빠뜨리지 않아야 한다. [ChilliCream — Projections: Nested Resolvers / Including Additional Fields](https://chillicream.com/docs/hotchocolate/fetching-data/projections)

### DataLoader는 실행기의 진행과 연결된다

현재 공식 설명에서 `LoadAsync`는 먼저 키를 모은다. 즉시 실행할 resolver 작업이 더 없을 때 pending batch를 dispatch한다. 고정된 GraphQL 깊이마다 한 번 실행하는 모델과는 다르다. 배치 조회 함수는 여전히 데이터 접근 로직을 작성해야 한다. [ChilliCream — DataLoader, 2026-08-30](https://chillicream.com/docs/hotchocolate/fetching-data/batching/dataloader)

따라서 NecrassRs의 목표는 DataLoader 타입의 존재뿐 아니라 **관계 resolver가 같은 로딩 경로에 도달하고, 실제 호출이 합쳐지며, 그 경계를 관찰할 수 있는가**여야 한다. 이를 근거 없이 breadth-first 실행기 재작성으로 바로 연결할 필요는 없다.

### 인증과 미들웨어도 공식 연결 지점이 있다

HTTP interceptor는 `HttpContext`와 GraphQL 요청 사이를 연결한다. 간단한 헤더 처리에는 delegate 방식도 있고, 인증은 ASP.NET Core 인증 체계를 재사용한다. endpoint 전체 인증과 필드별 authorization은 구분된다. [ChilliCream — Interceptors, 2026-08-17](https://chillicream.com/docs/hotchocolate/server/interceptors), [Authentication, 2026-06-30](https://chillicream.com/docs/hotchocolate/security/authentication), [Authorization, 2026-06-30](https://chillicream.com/docs/hotchocolate/security/authorization)

미들웨어의 풍부함에는 조합 비용도 따른다. 전통적인 paging·projection·filtering·sorting 설정에는 순서가 중요하며, field middleware는 호출과 결과 처리의 순서가 반대다. NecrassRs에서는 일반적인 실행 순서를 기본 경로로 정하고, 실제 필요한 확장 지점만 드러내는 편이 유리하다. [ChilliCream — Field Middleware, 2026-06-30](https://chillicream.com/docs/hotchocolate/resolvers/field-middleware)

## 5. Tonic: 계약에서 구현해야 할 Rust 경계를 만든다

### 방법론의 중심은 생성 코드와 사용자 코드의 분리다

Tonic의 기본 Prost 기반 개발 흐름은 다음과 같다. 조사한 API 문서의 표시 버전은 0.14.6이다.

```text
.proto: 서비스와 요청·응답 메시지 선언
  → build.rs: tonic-prost-build로 생성
  → Rust 메시지 타입 + 서비스 trait + server wrapper + client
  → 사용자가 서비스 trait 구현
  → 생성된 server wrapper를 기본 서버에 연결
```

공식 예시에서는 `Greeter` trait을 구현하고 `GreeterServer::new(...)`를 서버에 장착한다. 프로토콜을 연결하는 규칙적 코드는 생성물에, 실제 업무 처리는 사용자 구현에 위치한다. 최신 Prost 통합의 진입점은 `tonic-prost-build`다. [Tonic 프로젝트 — HelloWorld tutorial](https://github.com/grpc/grpc-rust/blob/master/examples/helloworld-tutorial.md), [tonic-prost-build 0.14.6](https://docs.rs/tonic-prost-build/latest/tonic_prost_build/)

이 방법을 NecrassRs에 적용하면 다음 흐름을 실험할 수 있다. **아래는 NecrassRs의 미구현 설계 제안이다.**

```text
schema.graphql: 공개 계약
  + 필요한 Rust 바인딩: 기존 backing type 또는 사용자 resolver 연결
  → 스키마 검증과 Rust 바인딩 생성
  → 입출력 타입·기본 필드 연결·필수 resolver 계약·등록 코드
  → 사용자가 값을 구성하고 계산·도메인 resolver 구현
  → 기존 GraphQL 실행기에 연결
```

여기서 공개 스키마의 필드와 Rust의 어떤 함수·데이터를 연결할지는 SDL만으로 알 수 없다. **GraphQL 계약과 구현 바인딩의 책임을 나누되 같은 필드 타입을 두 곳에서 손으로 반복 선언하지 않도록 검증해야 한다.** SQL 매핑까지 전부 SDL directive에 집어넣으면 독자 언어를 유지하는 비용이 생긴다.

### 계약 변경을 구현 작업으로 연결한다

생성 서비스 trait은 구현 시그니처의 기준이 된다. Tonic 생성기에는 기본 stub 생성 옵션도 있으므로, 모든 설정에서 미구현 메서드가 반드시 컴파일 오류라는 일반화는 피해야 한다. [Tonic — Builder 0.14.6](https://docs.rs/tonic-prost-build/latest/tonic_prost_build/struct.Builder.html)

NecrassRs에서는 다음 변경 루프가 유용한지 확인하면 된다.

1. 계약에 일반 필드를 추가하거나 nullability를 변경한다.
2. Rust 타입과 기본 필드 연결이 재생성되고, 사용자 코드가 새 계약에 맞지 않으면 오류로 드러난다.
3. 사용자는 필요한 값 구성이나 계산 로직을 수정한다. 구조체 필드 타입·getter·등록을 다시 선언하지 않는다.
4. 사용자 resolver가 필요한 필드는 누락을 식별하고, 구현 후 SDL과 실행 결과를 함께 확인한다.

단순 column field까지 모두 수동 trait 메서드로 구현하게 하면 기존 getter 반복을 다른 형태로 옮긴다. 기본 필드는 검증된 매핑에서 연결하고, 사용자의 구현 의무는 계산·관계·도메인 처리에 집중해야 한다.

### 코드 생성은 비용을 이동시킨다

Tonic 생성물의 기본 위치는 Cargo `OUT_DIR`이며 `include_proto!`는 이 기본 경로를 전제로 한다. 출력 위치를 바꾸면 별도 include 구성이 필요하다. 생성 파일을 볼 수 있다는 장점과 빌드 단계·경로를 관리해야 하는 비용이 함께 있다. [Tonic — include_proto! 0.14.6](https://docs.rs/tonic/latest/tonic/macro.include_proto.html)

따라서 SDL codegen이 proc-macro보다 항상 빠르고 IDE 경험도 좋다는 주장은 현재 근거로 할 수 없다. 생성 파일 탐색, 잘못된 타입의 오류 위치, 증분 빌드 시간은 실험 항목이다. 생성 타입이 실제 도메인 모델과 다르면 변환도 필요하다.

Tonic은 `.proto` 없이 Rust 선언으로 서비스 코드를 생성하는 manual 경로와 custom codec 경로도 소개한다. 그러므로 Tonic의 성공을 외부 IDL만이 유일한 정답이라는 근거로 해석하면 안 된다. 가져올 것은 **명시적인 계약과 재생성 가능한 연결 코드**다. [Tonic — tonic-prost-build Modules](https://docs.rs/tonic-prost-build/latest/tonic_prost_build/#modules)

### Request와 middleware는 연결 비용을 줄이는 또 다른 축이다

`Request<T>`에는 타입이 있는 메시지, gRPC metadata, 프로세스 내부 extensions가 있다. interceptor에서 extensions를 넣고 서비스에서 읽는 공식 예시도 있다. 다만 `extensions().get::<T>()`는 해당 값이 반드시 등록됐다는 컴파일 시점 보장은 아니다. [Tonic — Request 0.14.6](https://docs.rs/tonic/latest/tonic/struct.Request.html)

Tonic `Interceptor`는 동기 함수이며 `Request<()>`를 받는다. metadata 검사·변경과 요청 거부에는 적합하지만, 메시지 본문이나 비동기 DB 인증을 처리하는 범용 hook은 아니다. 더 풍부한 처리와 응답 관찰에는 공식적으로 Tower를 권한다. NecrassRs도 HTTP 변환, 비동기 인증, 필드 권한, 응답 관찰을 하나의 모호한 hook으로 합치지 않는 편이 좋다. [Tonic — Interceptor 0.14.6](https://docs.rs/tonic/latest/tonic/service/trait.Interceptor.html)

Tonic의 기본 transport는 Hyper·Tower·Tokio 기반이며 관련 기능을 feature로 나눈다. 이는 편한 기본 서버 구성과 하위 계층의 분리를 함께 제공하는 사례다. NecrassRs에서도 공식 Axum 경로를 짧게 제공하면서 핵심 실행을 HTTP 없이 사용할 수 있게 만드는 방향을 검증할 수 있다. [Tonic — Feature flags and Structure 0.14.6](https://docs.rs/tonic/latest/tonic/)

### GraphQL에서는 추가 실행 모델이 필요하다

일반적인 gRPC 메서드는 선언된 요청·응답 메시지 타입을 가진다. GraphQL은 operation의 선택 집합·인자·변수·fragment에 따라 필드 실행을 결정한다. 입력 coercion, null 전파, field error와 부분 데이터 처리는 코드 생성 이후에도 필요하다. [Tonic — HelloWorld tutorial](https://github.com/grpc/grpc-rust/blob/master/examples/helloworld-tutorial.md), [GraphQL September 2025 — Execution](https://spec.graphql.org/September2025/#sec-Execution)

**Tonic 스타일 생성기가 관계 SQL의 재사용과 N+1까지 해결하지는 않는다.** `IssueResolver`를 생성해도 관계 키·배치 함수·권한 범위가 없으면 어떤 SQL을 묶을지 결정할 수 없다. NecrassRs에는 계약을 Rust로 연결하는 빌드 단계와, 요청별 데이터 접근을 수행하는 실행 단계가 모두 필요하다.

## 6. 기존 Rust GraphQL의 한계는 어디까지인가

### 이미 있는 기능을 제외하고 비교한다

| 항목 | 기존 지원 | 여전히 검증할 사용자 부담 |
|---|---|---|
| 기본 필드와 계산 필드 | async-graphql `SimpleObject` + `ComplexObject` | 어떤 조합을 써야 하는지 발견하고 적용하는 과정 |
| 관계 batching | async-graphql DataLoader, Juniper의 loader 연동 예시 | 배치 SQL·키·결과 매핑·수명·resolver 연결 |
| 선택 집합 확인 | async-graphql Lookahead, Juniper Executor | 선택 정보를 실제 데이터 query에 반영하는 과정 |
| 요청 상태·권한·확장 | async-graphql Context, Guard, Extension | 필요한 의존성 등록과 정책 조합 |
| 동적 스키마 | async-graphql dynamic API | 등록·바인딩 오류가 드러나는 시점 |
| 구체적인 Context 타입 | Juniper Context | transport에서 해당 context를 구성하는 과정 |

근거: [async-graphql — Context](https://async-graphql.github.io/async-graphql/en/context.html), [Field Guard](https://async-graphql.github.io/async-graphql/en/field_guard.html), [Extension API](https://docs.rs/async-graphql/latest/async_graphql/extensions/trait.Extension.html), [Dynamic API](https://docs.rs/async-graphql/latest/async_graphql/dynamic/index.html), [Juniper — Using contexts](https://graphql-rust.github.io/juniper/master/types/objects/using_contexts.html), [Dataloaders](https://graphql-rust.github.io/juniper/master/advanced/dataloaders.html), [graphql_object API](https://docs.rs/juniper/latest/juniper/attr.graphql_object.html). 표의 나머지 기본 필드·loader 근거는 2절과 같다.

async-graphql 7.2.1의 기본 `DataLoader::new`는 `NoCache`다. 배칭과 레코드 캐싱을 혼동해서는 안 된다. 캐시를 선택적으로 켜거나 인증 범위가 있는 loader를 구성할 때 수명과 키 설계가 별도로 필요하다. [async-graphql — DataLoader API](https://docs.rs/async-graphql/latest/async_graphql/dataloader/struct.DataLoader.html)

또한 `ctx.data::<T>()`는 반환 타입을 알고 있지만 데이터가 등록됐는지는 실행 시 오류로 드러날 수 있다. NecrassRs의 typed context 제안은 이런 필수 의존성의 계약을 명확하게 만드는 데 목적이 있다. typed context 자체를 Rust 생태계 최초의 기능으로 내세울 수는 없다. [async-graphql — ContextBase API](https://docs.rs/async-graphql/latest/async_graphql/context/struct.ContextBase.html)

### Rust 언어의 제약과 라이브러리 선택을 구별한다

Rust의 orphan rule 때문에 외부 trait을 임의의 외부 타입에 자유롭게 구현할 수는 없다. wrapper나 별도 descriptor가 대응 방법이다. GraphQL interface도 단순히 Rust trait 이름 하나로 끝나지 않고 실제 반환값을 나타낼 enum 등의 표현이 필요하다. [Rust Reference — Trait implementation coherence](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementation-coherence), [Juniper — Interfaces](https://graphql-rust.github.io/juniper/master/types/interfaces.html)

이는 GraphQL 구현 불가능의 근거가 아니다. NecrassRs가 공개 타입과 backing type을 분리하거나 생성 코드를 사용할 때 고려해야 할 바인딩 조건이다. 문자열 registry가 존재한다는 사실만으로 실제 명세 오류를 증명할 수 없고, code-first라는 이유만으로 schema drift가 필연적이라고 할 수도 없다. 기존 일지의 그런 해석은 검증할 가설로 낮춰야 한다.

### Seaography를 비교군에 포함한다

Seaography는 SeaORM과 async-graphql의 동적 스키마를 연결하고, 관계·필터·페이지네이션과 사용자 query·mutation 확장을 제공한다. 관계 DataLoader 통합도 공식 소개에 등장한다. 따라서 Rust 생태계에 관계 자동화가 전혀 없다는 전제는 성립하지 않는다. [SeaQL — Seaography 공식 저장소](https://github.com/SeaQL/seaography), [SeaQL Team·Chris Tsang — Seaography 2.0, 2025-10-08](https://www.sea-ql.org/blog/2025-10-08-seaography/)

NecrassRs의 후보 차별점은 **SQLx·기존 서비스 중심으로도 공개 계약과 데이터 로딩의 연결을 단순하게 만드는 것**이다. 사용자가 SeaORM 중심 접근을 받아들일 수 있다면 기존 조합을 개선하거나 확장하는 편이 더 적합할 수도 있다. Seaography의 성능·컴파일 시간 홍보 문구는 독립 측정 없이 비교 결론에 사용하지 않았다.

## 7. 제안하는 설계 경계

### 공개 계약, Rust 바인딩, 데이터 정책을 나눈다

아래는 초기 프로토타입의 책임 구분이며 확정 API나 새 crate 목록이 아니다.

| 책임 | 담을 정보 | 초기 구현 방침 |
|---|---|---|
| 공개 GraphQL 계약 | 타입, 필드, 인자, nullability, 기본값, 설명 | SDL을 사용자가 편집하는 기준 계약으로 둔다. |
| Rust 바인딩 | backing type, scalar 변환, resolver 연결, 추상 타입 표현 | 규칙적 연결만 생성하고 누락·불일치를 일찍 보고한다. |
| 데이터 접근 정책 | 키 추출, batch 함수, 권한 범위, 필요한 서버 데이터 | 선언 생성 실험 이후 사용자 함수 재사용을 검증한다. |
| 전송 연결 | 요청 추출, 인증 결과, context 생성, 응답 변환 | 첫 실험은 HTTP 없이 실행하고, 이후 공식 Axum 경로를 검증한다. |
| GraphQL 실행 | 검증, coercion, 필드 실행, 오류 응답 | 기존 실행기 재사용으로 시작한다. |

내부 스키마 표현이 필요하다면 `Named`, `List`, `NonNull` 같은 구조화된 표현과 정의 위치를 사용한다. 다만 이는 **구현 내부의 표현**이며 사용자가 새 언어를 배워야 한다는 뜻은 아니다. 스키마 IR과 요청별 DB query plan도 같은 것으로 취급하지 않는다.

### 한 번 선언한다는 기준

일반 필드의 이름·타입·nullability는 SDL에서 한 번 작성한다. 생성기는 그 계약에 대응하는 Rust 타입, 기본 필드 접근, root와 타입 등록을 담당한다. 사용자 코드가 해당 필드의 실제 값을 만들어야 하는 것은 유지되지만, 타입 정보를 다시 선언하거나 값을 반환하기만 하는 getter를 작성할 필요는 없어야 한다.

| 정보·작업 | 작성 책임 |
|---|---|
| GraphQL 타입·필드·인자·nullability | 사용자가 SDL에 선언 |
| 생성 입출력 타입과 일반 필드 접근 | 생성기 |
| root·타입·interface/union 등록 연결 | 생성기 |
| 계산 필드와 업무 처리 | 사용자가 resolver 구현 |
| 기존 도메인 모델과 공개 타입의 차이 | 필요한 변환 또는 명시적인 바인딩 |

첫 프로토타입에서는 생성한 output 타입을 직접 반환하는 경로부터 검증한다. 이어 기존 모델이 있는 경우의 변환 비용을 별도로 확인한다. 동일한 구조의 DTO와 모든 필드별 바인딩을 함께 손으로 작성해야 한다면 반복 제거 효과가 약해진다. 반면 내부 정보 비공개나 데이터 표현 변경을 위한 변환은 의미 있는 경계다.

출력 객체와 생성·수정 입력도 의미가 다를 수 있다. 이름이 비슷하다는 이유만으로 같은 필드 집합과 nullability를 자동으로 강제하지 않는다. 첫 실험은 같은 계약 정보의 중복 투영을 줄이는 데 집중하며, 서로 다른 계약 간 필드 재사용 문법은 별도 사례로 평가한다.

### 입력 의미론을 정확히 보존한다

부분 수정 입력의 생략·명시적 null·값은 서로 다른 의미를 가질 수 있다. 그러나 기본값이 정의된 경우 GraphQL coercion은 생략을 기본값으로 대체한다. 따라서 원래 요청의 생략을 무조건 resolver까지 보존한다는 규칙은 부정확하다. **명세의 coercion을 적용한 뒤 남는 구분을 잃지 않는 것**이 요구사항이다. [GraphQL September 2025 — Input Objects](https://spec.graphql.org/September2025/#sec-Input-Objects)

초기 정책으로는 patch용 필드에 의도하지 않은 기본값을 넣지 않고 삼중 상태를 노출하는 방식을 권한다. 기존 Rust 구현체의 `MaybeUndefined`와 Juniper `Nullable`이 비교 기준이다. 모든 출력과 입력에 삼중 상태를 강제할 이유는 없다. [로컬 UpdateIssueInput](../../necrassrs-lab/services/api-async-graphql-sqlx/src/graphql/update_issue.rs), [Juniper — Nullable 0.17.1](https://docs.rs/juniper/latest/juniper/enum.Nullable.html)

root 이름은 `Query`, `Mutation`, `Subscription`을 기본값으로 제공하되 사용자 지정도 허용한다. SDL의 schema 정의 생략에는 기본 이름 외에도 조건이 있으므로, exporter가 올바른 명시적 mapping을 출력할 수 있어야 한다. [GraphQL September 2025 — Default Root Operation Type Names](https://spec.graphql.org/September2025/#sec-Root-Operation-Types)

### 데이터 접근의 세 가지 문제를 따로 해결한다

1. **재사용:** 여러 resolver가 동일한 서비스·조회 함수를 호출한다.
2. **배칭:** 같은 실행 구간의 키 조회를 모아 데이터 소스 호출을 줄인다.
3. **조회 계획:** 선택·필터·정렬·관계를 데이터 소스가 실행할 query로 변환한다.

선언 생성의 첫 프로토타입을 검증한 이후, 데이터 접근 단계에서는 1과 2부터 평가한다. 3은 관계·필터 정보를 얻는 경로가 검증된 어댑터부터 도입한다. Pothos의 ORM 플러그인과 Hot Chocolate의 `QueryContext`가 보여주는 것은 이 연결의 가능성과 전제다. 임의의 SQLx query에 동일한 자동화를 약속하는 근거는 아니다.

배칭에는 기본 정책이 필요하다. 요청별 권한 범위를 공유하고, 결과 의미가 다른 인자·정렬·페이지 조건을 섞지 않으며, 조회 실패와 해당 키의 값 부재를 구분한다. 캐시를 사용한다면 mutation 후 무효화나 갱신 규칙도 정한다. Pothos의 DataLoader 문서도 인자별 분리와 subscription 수명의 캐시를 별도로 다룬다. 이는 라이브러리 선택만으로 모든 수명 문제가 사라지지 않는다는 근거다. [Pothos — Dataloader plugin](https://pothos-graphql.dev/docs/plugins/dataloader)

특히 여러 부모의 `comments(first: 10)`은 모든 댓글을 합쳐 한 번 `LIMIT 10` 하는 문제와 다르다. 부모별 페이지 조건을 표현해야 한다. 데이터 접근을 다루는 단계에서는 단일 키 조회 관계를 재사용·배칭하고, 관계별 페이지네이션은 별도 실험으로 확장하는 편이 범위를 명확하게 한다.

### 사용자 로직을 감추지 않는다

인증 주체·tenant는 검증된 요청 context에서 전달하고, 실제 행 접근 권한은 데이터 조회 조건에서도 지킨다. SQLx transaction은 mutation의 명시적인 범위로 둔다. 예상 가능한 도메인 오류 payload와 GraphQL 실행 오류의 처리 경계도 문서화한다. 생성기는 SQL·권한·transaction을 추측해 자동 완성하는 역할을 맡지 않는다.

실험의 `x-user-id` 방식은 요청 전달을 확인하는 개발용 경로다. 이후 인증 튜토리얼에서는 검증된 인증 주체를 만드는 단계와 그 주체를 resolver에 전달하는 단계를 나눠야 한다. 이 구분은 4–5절의 Hot Chocolate·Tonic 연결 방식과도 맞닿는다.

## 8. 무엇을 먼저 만들 것인가

### 구현 경로의 선택

| 경로 | 검증할 가치 | 비용·판정 조건 |
|---|---|---|
| 기존 async-graphql 활용 개선 | 새 프레임워크 없이 얼마나 불편이 줄어드는가 | 가장 먼저 만들 비교 기준선 |
| SDL → Rust 바인딩 → 기존 실행기 | Tonic식 계약 변경 루프가 등록·구현 반복을 줄이는가 | 첫 NecrassRs 프로토타입으로 권고 |
| 자체 GraphQL 실행기 | 기존 실행기의 확장 지점으로 표현할 수 없는 요구가 있는가 | 재현된 제약과 유지보수 여력이 확인된 뒤 판단 |

Pothos와 Hot Chocolate의 강점이 code-first에서도 구현된다는 점은 중요하다. **좋은 개발 경험과 schema-first는 동의어가 아니다.** NecrassRs는 사용자가 선택한 SDL 중심 방향 안에서 반복 감소와 변경 경험을 검증한다. 기존 derive 방식은 효과를 비교할 기준선으로 사용하며, 별도의 Rust builder를 동등한 스키마 작성 언어로 추가하지 않는다.

### 실험 순서와 통과 기준

| 순서 | 실험 | 확인할 결과 |
|---|---|---|
| 1 | DB 없는 `User`·root query·작은 수정 입력으로 derive 방식의 기준선 구성 | `SimpleObject` 등 기존 기능을 활용한 상태의 선언·getter·등록 작성량을 확인한다. |
| 2 | 동일 계약의 SDL → Rust 타입·바인딩 생성 | 생성 output을 직접 반환하고 일반 필드의 수동 getter·타입 등록을 없앤다. |
| 3 | 필드 추가·이름 변경·nullability 변경·입력 타입 변경 | 계약 정보는 한 곳에서 바꾸고, 값 구성 등 영향받은 사용자 로직만 수정한다. |
| 4 | 기존 backing type 연결과 interface/union 확장 | 필드별 재선언 비용을 확인하고, 추상 타입 등록·연결 누락을 검출한다. |
| 5 | 선언 생성 검증 후 실제 lab로 확대 | 동일 SDL·동작을 맞춘 뒤 SQLx 관계 loader·pagination을 후속 검증한다. |

각 실험은 기존 lab의 타입과 계약을 축소해 수행한다. 첫 통과 기준은 일반 필드의 계약을 바꿀 때 Rust 구조체 필드 타입·getter·등록 코드를 함께 손으로 고치지 않아도 되는가다. 전체 사용자 작성량과 값 변환 코드까지 비교해야 한다. Seaography·관계 로딩 비교는 후속 데이터 접근 단계에서 진행한다.

관찰 지표는 다음처럼 구체적으로 둔다. 선언·변경 비용과 진단을 먼저 평가하고, SQL·배칭 지표는 후속 단계에 적용한다. 아래는 측정 계획이며 이번 조사에서 얻은 성능 수치가 아니다.

- 새 필드·관계 하나를 추가할 때 수정하는 사용자 파일 수와 중복 선언 수
- 잘못된 인자 타입·누락된 resolver·누락된 context를 발견하는 시점과 오류 위치
- 부모 1·10·100개에서 SQL 호출 수, batch 크기, 반환 row 수
- 서로 다른 alias·인자·사용자 요청에서 데이터가 섞이지 않는지
- 입력 생략·null·값, mutation 후 재조회, pagination 경계의 결과
- 동일 환경의 증분 빌드 시간과 생성 코드 탐색 경험

SQL 호출 수 하나로 전체 성능을 평가하지 않는다. 반환 row 수, 불필요한 중복 데이터, 지연도 함께 봐야 한다. 자체 실행기 개발을 시작한다면 지원 GraphQL 판본과 동작 검증 범위를 별도로 선언한다. HTTP 상호운용 규칙은 언어·실행 명세와 나누고, 현재 GraphQL over HTTP 문서가 draft라는 상태도 기록한다. [GraphQL over HTTP — Stage 2 Draft](https://graphql.github.io/graphql-over-http/draft/)

## 9. 문서도 첫 기능에 포함한다

Discord에서 드러난 문서 요구는 단순한 장식 요구로 보이지 않는다. `User` 타입을 만들면 root query도 생기는지, `QueryRoot`가 왜 나오는지, `SimpleObject`와 관계 resolver를 어떻게 합치는지처럼 **코드의 어느 부분이 GraphQL의 어떤 개념이 되는지**를 이해하려는 요구다.

권장 튜토리얼은 하나의 예제를 다음 순서로 확장한다.

1. 계약에 공개 필드를 선언하고 생성된 Rust 타입과 연결 코드를 확인한다.
2. root resolver에서 값을 반환하고 operation과 응답을 실행한다.
3. 필드와 nullability를 변경하며 생성물·오류·사용자 수정 범위를 확인한다.
4. mutation 입력을 생략·null·값으로 바꾸며 결과를 비교한다.
5. 후속 예제에서 관계 필드와 loader를 연결하며 DB 호출 변화를 확인한다.
6. 헤더에서 만든 인증 주체가 context와 권한 검사에 도달하는 흐름을 본다.

mdBook은 편집·실행 가능한 Rust 예제를 지원한다. 다만 공용 Rust Playground의 실행 컨테이너는 외부 네트워크가 없고 시간·메모리 제한이 있다. 이를 실제 DB·서버 프로젝트의 자유로운 실행 환경과 동일시해서는 안 된다. [mdBook — Editor](https://rust-lang.github.io/mdBook/format/theme/editor.html), [Rust 프로젝트 — Playground Resource Limits](https://github.com/rust-lang/rust-playground#resource-limits)

첫 버전에서는 저장소의 실행 가능한 예제와 문서 코드를 공유하고, SDL·응답·조회 흐름을 연결해 보여주는 구성이 적절하다. Rust 전체를 브라우저에서 재컴파일하는 서비스는 별도 요구가 확인된 뒤 판단한다. 문서에서 설명한 실제 생성 결과와 실행 결과가 바뀌면 함께 검출되도록 만드는 것이 우선이다.

## 10. 다음 설계 논의의 출발점

**확정된 첫 과제는 “스키마·타입 선언의 반복 감소”이며, 공개 계약은 SDL에서 관리한다. 다음 검증 대상은 SDL에서 Rust 타입·기본 필드 연결·등록 코드를 생성하는 방식이다.** Tonic의 생성 코드 경계를 중심으로 검증하고, Pothos의 backing type 연결을 비교한다. Hot Chocolate의 실행 통합과 관계 로딩은 후속 단계에서 참고한다.

현재 결정과 남은 검증 항목은 다음과 같다.

| 논점 | 현재 상태 | 후속 검증 |
|---|---|---|
| 첫 해결 과제는 무엇인가 | 스키마·타입 선언의 반복 감소 — 사용자 지정 | 데이터 접근은 후속 과제로 둔다. |
| 공개 계약은 어디에서 편집하는가 | SDL — 사용자 선택 | SDL와 Rust 바인딩 사이에 타입 정보의 수동 재선언이 생기지 않는지 확인한다. |
| 처음부터 자체 실행기가 필요한가 | 기존 실행기에 연결해 사용자 경험부터 검증 | 필요한 바인딩·배치 정책을 기존 확장 지점으로 표현할 수 없다는 재현 사례가 생기면 재검토한다. |

Gelite는 향후 관계 메타데이터를 제공하는 어댑터 후보가 될 수 있다. 그러나 첫 핵심 구조가 Gelite의 완료를 기다리게 만들 필요는 없다. 이 판단은 제공된 대화의 개발 일정과 현재 lab 구성을 고려한 범위 제안이며, Gelite 구현을 조사해 호환성을 확인한 결론은 아니다.

## 11. 조사 범위와 남은 불확실성

이번 조사는 로컬 코드의 구조와 공식 문서의 동작 계약을 확인했다. 서버·DB·.NET 프로그램을 새로 실행하거나 성능 측정, compiler 진단 비교, 전체 GraphQL 명세 적합성 검증을 수행하지는 않았다. 따라서 제품 간 성능 우위, 컴파일 속도 개선, 기존 실행기의 구조적 결함은 결론으로 제시하지 않는다.

Rust 비교의 API 기준은 async-graphql 7.2.1, Juniper 0.17.1이다. 로컬 Pothos core 설치 버전은 4.13.0이며, 공식 가이드는 현행 문서로 읽었다. Hot Chocolate는 v16 계열 현행 문서, Tonic은 표시 버전 0.14.6을 참고했다. 로컬 의존성 고정값과 현행 문서의 API 차이는 실제 프로토타입에서 다시 맞춰야 한다.

7월 개발일지는 사용자의 경험과 당시 가설을 이해하는 자료로 사용했다. 현재 구현 상태는 소스와 대조했고, 과거 이슈의 현재 해결 여부나 미측정 성능 주장을 그대로 옮기지 않았다. Discord에서 정확한 프로젝트를 식별할 수 없는 “페디파이” 언급은 특정 프레임워크로 추정하지 않았다.

핵심 설계 질문마다 공식 근거 또는 명시적인 한계를 확보하고, Tonic 방법론과 Pothos 기본값의 버전 차이까지 추가 확인한 뒤 검색을 종료했다. 이제 결정을 바꿀 가능성이 큰 증거는 더 많은 기능 소개보다 **동일한 작은 시나리오를 작성·변경·실행해 얻는 결과**다.

Markdown의 제목·표·코드 블록·로컬 상대 링크를 구조적으로 검수했다. 별도 Markdown 렌더러를 사용할 수 없어 렌더링 화면의 시각 검수는 수행하지 않았다.
