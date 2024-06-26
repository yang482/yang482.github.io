---
# layout:
# permmalink
# published

title: REST API 설계
date: 2024-06-25 10:57:00 +0900
categories: ["시스템 디자인"]
tags: ["rest api"]     # TAG names should always be lowercase
comments: true
---

## REST API 설계

여기서는 HTTP를 활용한 WEB REST API 를 다루겠습니다.  
REST는 프로토콜은 아니기 때문에 다양한 형태로 설계 가능합니다.  
하지만 기본 원칙이 존재하며 이를 준수해야 합니다.  

<br />

### 기본 원칙

> 1. Uniform Interface  
> 2. Client - Server  
> 3. Stateless  
> 4. Cacheable  
> 5. Layered System  
> 6. Code on Demand (Optional)  

기본적으로 HTTP를 활용한다면 위의 원칙 중  
Client - Server, Stateless, Cacheable(Cache-Control 헤더 사용)은 자연스럽게 충족 됩니다.  
또한 URI와 HTTP Method를 사용함으로 Uniform Interface 를 충족 할 수 있습니다.  
마지막으로 리소스 중심 설계를 통해 Layered System의 조건을 충족 할 수 있습니다.

<br />

### REST API 설계 진행


#### 1. 리소스 중심 설계
REST API는 URI 구성 시 동사를 지양하고 명사를 기반으로 해야합니다.

| https://your-domain.com/users | **(O)** |  
| https://your-domain.com/get-users | **(X)** |  
| https://your-domain.com/get/users | **(X)** |


#### 2. Method 활용 하기

API 설게 단계에서 Method 에 따른 [멱등성(idempotent)](https://developer.mozilla.org/ko/docs/Glossary/Idempotent)은 중요한 부분입니다.
GET, PUT, DELETE 에 대해서는 멱등성이 보장 되어야 합니다.  
POST, PATCH 에 대해서는 멱등성이 보장 되지 않을 수 있습니다.  

- **주로 사용되는 방식**

| 1 | GET | https://your-domain.com/board | 조회 |
| 2 | POST | https://your-domain.com/board | 등록 |
| 3 | POST | https://your-domain.com/board/{id} | 수정 |
| 4 | PUT | https://your-domain.com/board/{id} | 수정 |
| 5 | PATCH | https://your-domain.com/board/{id} | 수정 |
| 6 | DELETE | https://your-domain.com/board/{id} | 삭제 |  

POST 요청에 대해서는 보통 2번의 형태로 사용하게 됩니다.    
이는 같은 데이터로 넘어온 요청 모두에 대해 신규 id로 데이터를 등록 하여    
중복 데이터가 발생 하는 형태로 사용 되어 멱등성이 보장 되지 않을 수 있습니다.   

하지만 3번 요청처럼 기존 데이터의 id를 함께 전송하여  
마치 PUT처럼 데이터를 수정 할 수도 있습니다.  
이 경우 마치 멱등성을 보장 받는 것 처럼 동작하게 됩니다.

PUT 요청은 4번의 형태로 사용되며 멱등성이 보장되어야 합니다.  
즉, PUT을 사용할 때에는 항상 id 값 등을 통해 기존 데이터에 대해 수정만 하고  
새로 생성되거나 삭제 되지 않도록 하는 것을 권장합니다.  

PATCH 요청의 경우는 통상 데이터의 일부만 수정을 하도록 구현 합니다.
조금 단적인 예로 주문의 상태 값만을 변경 하는 요청등이 있습니다. 


#### 3. 다양하게 생각하기

| 유저별 게시글 리스트 | GET | https://your-domain.com/user/{userId}/board |
| 특정 게시글의 댓글 리스트 | GET | https://your-domain.com/board/{boardId}/comment | 

상기 예시의 경우에서 조금 더 복잡한 경우를 생각해 보고자 합니다.  
보통은 댓글과 대댓글은 재귀를 활용해 계층 형태로 가져오겠지만..    
예시를 위해 댓글 단건(상세) 조회, 특정 댓글에 대한 대댓글 조회가 있다고 가정하겠습니다.  
단순하게 생각해보면 다음과 같이 진행해 볼 수 있습니다.

| 댓글 단건(상세) | GET | https://your-domain.com/board/{boardId}/comment/{commentId} |

위 처럼 댓글 단건(상세) 조회의 경우 사용할 URI를 생각해 볼수 있습니다.  
특정 댓글에 대한 대댓글의 경우 여러가지로 생각해 볼 수 있습니다.

| 1 | GET | https://your-domain.com/board/{boardId}/comment/{upperCommentId} |
| 2 | GET | https://your-domain.com/board/{boardId}/comment/{commentId}/reply-comment |
| 3 | GET | https://your-domain.com/board/{boardId}/upper-comment/{upperCommentId} |
| .. | GET | .. |

1의 경우 댓글 단건(상세)와 URI의 형식이 겹치게 되어 사용이 불가 합니다.  
2나 3의 경우에는 사용 가능한 URI이나 구성이 복잡해지게 됩니다.

이런 경우는 쿼리 파라미터를 활용하는 것도 방법입니다.  

| https://your-domain.com/board/{boardId}/comment?upperCommentId= |

댓글 리스트를 조회하는 URI에 upperCommentId 라는 파라미터 값의 유무를 통해    
URI의 본연의 기능을 유지하여 댓글 조회는 물론 특정 댓글의 대댓글을 가져올 수 있습니다.

정답이 정해져 있지는 않습니다.  
다만 모든 기능을 URI를 통해 해결하려고 하여   
REST API가 복잡하고 어려워 지고 있는건 아닌지 생각해 보면 좋을 듯 합니다.    


#### 4. 요청에 대해 응답하기

REST API의 응답 데이터는 보통 JSON 형태의 포맷을 가집니다.  
이는 표준 포맷을 활용함으로 어떠한 플랫폼, 애플리케이션에 독립성을 가지게 합니다.  

응답 역시 중요한 고려 사항입니다.  

우선 아래와 같이 리소스 본연의 데이터 응답만을 할 수도 있습니다.  
```json
{
  "userId": "user01",
  "userName": "유저1",
  "age": 30
}
```

HTTP의 응답 상태 코드 외에 아래와 같은 방법으로 요청에 대한 처리 결과를 전달 할 수도 있습니다.  
처리 결과에 대한 메시지로 alert 등이 가능합니다.
```json
{
  "userId": "user01",
  "userName": "유저1",
  "age": 30,
  "result": {
    "code": "S",
    "message": "조회에 성공 하였습니다."
  }
}
```

리소스를 특정 키 (data 같은)에 담아 주는 방법도 있습니다.  
이 방법의 경우 클라이언트 측에서 응답을 처리할 때  
정형화된 방법으로 처리 결과와 데이터를 분리해 줄수 있다는 장점이 있습니다.  
```json
{
  "data": {
    "userId": "user01",
    "userName": "유저1",
    "age": 30
  },
  "result": {
    "code": "S",
    "message": "조회에 성공 하였습니다."
  }
}
```

`HATEOAS`와 `HAL`은 [Richardson이 제안한 REST 성숙도 모델](https://grapeup.com/blog/how-to-build-hypermedia-api-with-spring-hateoas)에서  
가장 성숙한 모델인 Level 3에 해당할 수 있게 해줍니다.

기본 적인 형태로 `_links` 키에 해당 리소스와 관련된 링크를 제공합니다.
```json
{
  "userId": "user01",
  "userName": "유저1",
  "age": 30,
  "_links": {
    "self": {
      "href": "https://your-domain.com/users/user01"
    },
    "friends": {
      "href": "https://your-domain.com/users/user01/friends"
    }
  }
}
```

또는 `_embedded`를 통해 리스트, 또는 다른 리소스를 제공할 수도 있습니다.

```json
{
  "userId": "user01",
  "userName": "유저1",
  "age": 30,
  "_embedded": {
    "friends": [
      {
        "userId": "user02",
        "userName": "유저2"
      }
    ]
  },
  "_links": {
    "self": {
      "href": "https://your-domain.com/users/user01"
    },
    "friends": {
      "href": "https://your-domain.com/users/user01/friends"
    }
  }
}
```
```json
{
  "_embedded": {
    "users": [
      {
        "userId": "user01",
        "userName": "유저1",
        "age": 30
      },
      {
        "userId": "user02",
        "userName": "유저2",
        "age": 24
      }
    ]
  },
  "_links": {
    "self": {
      "href": "https://your-domain.com/users"
    }
  }
}
```

HATEOAS는 클라이언트와 서버간의 관계를 느슨하게 하고  
리소스에 대한 추가작업을 명시할 수 있는 장점이 있는 반면  
link에 대한 정보가 추가됨에 따라 트래픽 데이터가 증가되고  
link를 관리하는 복잡도가 증가하는 단점이 있습니다.  

#### 5. REST API 버전 관리

REST API 의 버전을 관리하기 위해서는 3가지 방법을 사용할 수 있습니다.

> 1. URI 관리
> 2. 쿼리 파라미터 관리
> 3. HTTP Custom Header 관리
> 4. HTTP MIME Type (HTTP Header의 Accept 사용) 관리

**URI 관리**  

> GET https://your-domain.com/v1/users

장점 :  
- URI로 관리 하는 방법을 사용하면 버전이 URI에 명시되어 있어 한눈에 파악하기가 쉽습니다.  
- 클라이언트가 캐시를 처리하기가 쉽습니다.    

단점 :  
- 모든 URI에 버전이 명시되어 있어 서버가 추가되는 버전에 대한 URI 를 모두 가지고있어야 하므로  
  코드량(특히, Spring MVC의 경우 Controller)이 많이 증가 할 수 있습니다.  
- 기본 버전 설정이 어렵게 됩니다.  

<br />

**쿼리 파라미터 관리** 

> GET https://your-domain.com/users?version=1

장점 :  
- 기본 버전 설정이 쉽습니다.  
- 클라이언트가 캐시를 처리하기가 쉽습니다. (일부 브라우저 등은 제외)

단점 :  
- 쿼리 파라미터에 따른 라우팅이 어려워 서버가 분기 처리 하여 응답하도록 해야합니다.

<br />

**HTTP Custom Header 관리**

> GET https://your-domain.com/users  
> API-VERSON: 1

장점 :   
- 기본 버전 설정이 쉽습니다.
- URI 상에 버전이 명시되지 않아 URI의 간결함을 유지할 수 있습니다.

단점 :  
- 로드밸런서나 웹서버에 Custom Header를 허용하도록 설정 해야 할 수 있습니다.

<br />

**HTTP MIME Type 관리**

> GET https://your-domain.com/users  
> Accept: application/vnd.**identifier**.v1+json

**identifier** 부분에 회사 이름, 프로젝트 이름 등을 명시할 수 있습니다.  
ex) Accept: application/vnd.company.sample.v1+json

장점 :
- 기본 버전 설정이 쉽습니다.
- URI 상에 버전이 명시되지 않아 URI의 간결함을 유지할 수 있습니다.

단점 :
- Accept 라는 표준 Header를 사용하기는 하나 잘 알려진 MIME Type 값을 지정하지 않는 경우  
  웹 서버는 406 코드를 반환 할 수 있는 단점이 있습니다.

<br />

### 마치며

REST API는 위에서도 언급 했듯이 프로토콜이 아니기 때문에 다양한 형태로 설계 가능합니다.  
하지만 설계의 기본 6원칙은 되도록 준수하는 것이 좋습니다.  

위의 사항을 항상 명심하여 리소스 중심의 설계, HTTP 메서드를 통한 동작 지정,  
필요한 비즈니스를 충족할수 있도록 다양하게 생각해보기,  
응답에 대한 고민, 버전 관리 방법 선택의 순서로 차근차근 진행하면  
근사한 REST API가 만들어 질 수 있다고 생각합니다.

특히나 MSA에서 REST API는 중요한 인터페이스라고 생각합니다.  
꼭 설계자가 아니더라도 본인이 맡은 업무를 REST 답게 생각해보고  
더 나은 방법을 고민해보면 좋겠습니다.  
