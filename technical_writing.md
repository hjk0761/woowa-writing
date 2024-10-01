# 테크니컬 글쓰기 - 초안
제목: Github OAuth 를 이용하여 소셜 로그인 구현하기

작성자: 우아한 테크코스 6기 BE 김현중(무빈)

날짜: 2024-10-01 (화)

# 목차

0. [서론](#서론)
1. [로그인 구현](#로그인-구현)
    1. [로그인의 필요성](#로그인의-필요성)
    2. [토큰? 쿠키? 세션?](#토큰-쿠키-세션)
2. [소셜 로그인이란](#소셜-로그인이란)
    1. [OAuth 란](#oauth-란)
    2. [Github OAuth 사용하기](#github-oauth-사용하기)
3. [지금 우리 프로젝트에서는](#지금-우리-프로젝트에서는)
    1. [책임 분리](#책임-분리)
    2. [토큰 사용 여부](#토큰-사용-여부)
    3. [최종 구조](#최종-구조)
4. [참조](#참조)

<br>

# [서론](#목차)

본 글은 CoReA(Code Review Area) 프로젝트에서, 로그인 관련 기능을 설명하는 글이다.

우선적으로 팀 내부에서 해당 문서를 통해 로그인 프로세스를 이해함을 목적으로 두기 때문에 기술이 선정된 이유, 구현한 방법 등이 주가 된다.

추가적으로 로그인 기능을 구현하려는 백엔드 개발자들이 소셜 로그인 특히 Github OAuth 를 통한 로그인에 대해 프로세스를 이해하고 적용할 수 있기 위함을 목표로 둔다.

<br>

# [로그인 구현](#목차)

## 로그인의 필요성

CoReA(Code Review Area) 서비스는 개발자의 코드 리뷰 문화를 좀 더 용이하게 이용할 수 있게 돕는 서비스이다.

코드 리뷰란 개발자가 상호 간의 리뷰이, 리뷰어가 되어 서로의 코드를 리뷰하고 개선하는 활동을 의미한다.

따라서 코드 리뷰를 중계하기 위해선 사용자를 인식할 필요가 있었고, 이를 위해 로그인 기능을 도입하기로 결정했다.

대부분의 개발자가 형상 관리 도구로 Git 을 이용하고 코드 리뷰 또한 Github 를 사용하기에 이와 연계된 정보가 필요했고, 따라서 로그인 기능을 직접 구현하는 것 보다 Github 에서 제공하는 OAuth 를 이용하여 구현하기로 결정했다.

## 토큰? 쿠키? 세션?

기본적으로 HTTP 통신은 무상태성을 지향한다. 따라서 매 요청은 독립된 요청이고, HTTP 통신 만으로는 사용자를 특정할 수 없다.

로그인과 같이 상태를 기록해야 할 때 사용하는 기술의 예로 토큰, 쿠키, 세션 등이 있다.

아래는 각 기술에 대한 간단한 설명이며, 각 기술의 장단점을 알고 프로젝트의 방향성에 맞는 기술로 상태를 기록할 수 있기 위한 설명이다.

### 쿠키

브라우저에 딸린 저장소 혹은 저장하는 객체를 의미하는 단어이다. 혹은 공개 가능한 정보를 브라우저의 저장소(쿠키)에 저장하는 방법을 일컫기도 한다.

서버에서 클라이언트로 응답할 때에, 쿠키 저장소에 저장하고 싶은 정보를 `Set-Cookie` 헤더의 값으로 담아 응답하면 이는 클라이언트의 브라우저에 저장되며, 이후 요청마다 `Cookie` 헤더에 해당 값이 담겨서 서버로 전달된다.

브라우저에 별다른 암호화 없이 저장되기 때문에 별다른 조치를 취하지 않았다면 제 3자가 탈취하여 사용할 수도 있다.

### 세션

직접 정보를 브라우저에 저장하는 것이 아닌, 서버측의 별도의 저장소인 세션 저장소에 정보를 저장하는 방식을 말한다. 클라이언트에는 세션 저장소에 해당 정보가 저장된 식별자인 `session id` 를 전달하여 보관하게 한다. 따라서 세션 저장소에 직접 접근할 수 없는 한 `session id` 자체는 의미를 가지지 못하지만, 매 요청마다 세션 저장소와 통신해야한다는 코스트가 있다.

### 토큰

내부의 별도 로직을 통해 인증 정보를 암호화한 임의의 문자열을 의미한다. 서버는 토큰을 암호화 혹은 복호화 할 알고리즘을 가지고 있으며, 정보를 암호화하여 클라이언트에 전달하고, 클라이언트로부터 온 요청에 담긴 토큰을 복호화하여 인증 정보가 맞는지 확인한다.

<br>

# [소셜 로그인이란](#목차)

내부 로직으로 로그인울 구현하는 것은 쉽지 않다.

사용자 정보를 어떻게 저장하고 불러올지, 이 값들을 어떻게 보안을 유지하면서 관리할지 등 다양한 문제 상황을 마주할 수 있다.

이런 상황에서 이미 검증된 사이트의 인증 로직을 통해 사용자에 대한 인증을 위임할 수 있다면 어떨까?

이에 대한 답이 바로 OAuth(OpenId Authentication) 이다.

## OAuth 란

OAuth 는 사용자 인증 프로토콜로, 실제 사용자 자격을 공유하지 않고도 한 서비스에서 다른 서비스로 권한 부여를 전달하기 위한 프로토콜이다.

동작 원리를 알기에 앞서 통신의 주체를 정의한다.

1. Resource Owner: 리소스를 소유하는 주체. 우리 서비스를 이용하려는 사용자에 해당.
2. Authoization Server: Resource Owner 를 인증하고 Access Token 을 발급하는 주체.
3. Resource Server: 보호되는 리소스를 가지는 주체.
4. Client: 위 서버들에 접속하는 주체. 우리가 개발하려는 서비스가 이에 해당.

이들을 아래와 같은 과정을 통해 인증을 위임하여 가능케 한다.
```
     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+
     (다이어그램 출처: https://datatracker.ietf.org/doc/html/rfc6749#section-1.2)
```

1. Client 가 Resource Owner 로 부터 인증을 요구한다.
2. Resource Owner 는 인증 정보를 응답한다.
3. Client 는 Authorization Server 로 인증 정보를 통해 Access Token 을 요구한다.
4. Authorization Server 는 해당 정보를 확인하여 올바른 Access Token 을 응답한다.
5. Client 는 해당 Access Token 을 통해 Resource Server 에 리소스를 요청한다.
6. Resource Server 는 올바른 Access Token 인 경우 이에 해당하는 리소스를 응답한다.

이런 과정을 신빙성 있는 Authorization Server, Resource Server 를 운영하는 서비스가 구현해 두었다면, 우리는 이 과정을 통해서 인증 작업을 위임할 수 있는 것이다.

## Github OAuth 사용하기

앞서 설명하였듯이, 우리 서비스는 Github OAuth 를 사용하기로 결정했다.

아래는 Github 에서 제공하는 OAuth API 를 어떻게 이용하는지에 대한 설명이다.

아래 과정에 앞서, Github OAuth 를 사용하고자 하는 서비스의 도메인을 Github 에 등록해야 한다.

`https://github.com/settings/developers` 의 `OAuth Apps` 를 통해 서비스의 도메인을 등록하면, `client_id` 와 `cliend_secret` 을 발급받는다.

위 두 인자를 통해 이후 프로세스를 진행하기 때문에 내부적으로 저장해둘 필요가 있다.

Github 에서 설명하는 OAuth 프로세스는 다음 단계를 거친다.

1. 사용자는 GitHub ID를 요청하도록 리디렉션됩니다.
```
GET https://github.com/login/oauth/authorize
```

해당 엔드포인트에는 `client_id` 를 매개변수로 필요로 한다.

요청이 정상적이라면, 이전 `OAuth Apps` 에 등록할 당시 callback URL 을 기입하게 되는데, 해당 callback URL 의 매개 변수로 code 를 응답한다.

2. GitHub가 사용자를 사이트로 다시 리디렉션합니다.
```
POST https://github.com/login/oauth/access_token
```

이 과정은 `client_id`, `client_secret`, 그리고 1단계에서 획득한 `code` 를 모두 매개 변수로 전달한다.

정상적인 요청의 경우 다음과 같은 형태로 응답한다. 아래는 `application/json` 형태로 받은 예시이다.
```
Accept: application/json
{
  "access_token":"gho_16C7e42F292c6912E7710c838347Ae178B4a",
  "scope":"repo,gist",
  "token_type":"bearer"
}
```

3. 앱이 사용자의 액세스 토큰을 사용하여 API에 액세스합니다.
```
Authorization: Bearer OAUTH-TOKEN
GET https://api.github.com/user
```

2단계에서 얻은 Access Token 을 통해 Github User 에 대한 리소스를 요청한다.

정상적인 요청의 경우 필요한 정보를 반환하게 된다.

앞서 설명한 통신의 주체로 비교해보자면,
1. Resource Owner: 1단계에서 Github Id 를 전달하는 사용자
2. Authoization Server: 1, 2단계에서 `https://github.com/login/oauth/` 도메인을 소유한 Github OAuth 서버
3. Resource Server: 3단계에서 `https://api.github.com/` 도메인을 소유한 Github API 서버
4. Client: 위 통신을 구현할 우리 서비스

로 정리할 수 있다.

위 과정에 맞게 기능을 구현했다면, 해당 서비스는 Github OAuth 를 통해 사용자 인증 작업을 위임할 수 있게 된다.

<br>

# [지금 우리 프로젝트에서는](#목차)

아래는 현재 CeReA 에서 구현된 Github OAuth 를 이용한 소셜 로그인 처리가 어떻게 되어 있는지에 대한 설명이다.

## 책임 분리

앞서 말한 OAuth 를 통한 로그인 과정에는 `client_id` 를 통해 `code` 를 받아오는 과정과, `client_id` 와 `client_secret` 그리고 `code` 를 통해 실제로 인증하는 과정으로 구분된다.

`code` 를 받는 과정에서 사용자가 본인의 Github 아이디로 로그인하는 과정이 필요하므로 두 과정에 대한 책임을 클라이언트와 서버로 각각 분리했다.

클라이언트를 통해 사용자가 Github 인증 작업을 시작하면, 클라이언트는 Github API 를 통해 직접 `code` 를 얻어온다.
위에서 얻은 code 를 서버로 전달하면, 서버는 Github API 를 통해 사용자를 인증하고 사용자의 Github 정보를 받아온다.

## 토큰 사용 여부

위 과정 중 서버가 Github API 를 이용하여 사용자를 인증하는 단계에서 Github API 는 해당 사용자에 대한 조회를 위해 Access Token 을 발급해 반환한다.

우리는 이 토큰을 서비스 내에서 직접 사용하지 않는다.

인증 성공 직후 사용자 정보를 가져와 계정을 생성 혹은 조회하는 데에 사용하고 이후는 서버 내부에서 만든 토큰을 이용한다.

이렇게 선택한 이유는 다음과 같다.

먼저 Github API 를 통해 발급받은 토큰이 클라이언트로 유출되지 않아 해당 토큰을 통한 비정상적인 접근을 원천 차단할 수 있다.

또한 내부 정책으로 정의된 토큰을 클라이언트와의 소통에 참여하면서 외부에선 해당 토큰을 탈취해도 다른 프로세스에 사용할 수 없다.

프로세스로 돌아가, Github 를 통한 인증에 성공하고 Github Access Token 을 통해 사용자의 정보를 가져왔다면, 내부적으로 JWT 를 이용해 두 종류의 토큰을 생성한다.

Access Token 과 Refresh Token 이며, 사용처는 다음과 같다.

전자는 로그인 정보가 필요한 모든 순간에 쓰이며 로그인한 사용자의 정보를 인코딩해 전달한다. 그렇기 때문에 클라이언트에서만 저장하며, 짧은 만료 시간을 가진다는 특성을 지닌다.

후자는 Access Token 이 만료된 시점에 사용되며, 만료된 토큰을 사용하는 사용자가 기존의 사용자가 맞는지 확인하고 Access Token 을 재발급해주는 용도이다. 따라서 클라이언트와 서버 모두 해당 토큰을 가지고 있다.

Access Token 이 만료되고 재발급 받기까지의 과정은 사용자에게 전달되지 않고 클라이언트와 서버에서 처리하므로 유저는 Access Token 의 만료 여부와 상관 없이 서비스를 이용할 수 있다.

단, Refresh Token 만료 시 로그인 정보를 지움으로써 남아있는 로그인 정보를 없애기 때문에 이 경우 사용자는 다시 로그인을 해야 한다.

로그아웃 요청 시 클라이언트와 서버 모두 Refresh Token 을 지우는 작업을 수행한다.

## 최종 구조

아래는 현재 CoReA 에서 사용하고 있는 로그인 구조를 나타낸 다이어그램이다.

![login_process](images/login_process.png)
![login_token](images/login_token.png)

<br>

# [참조](#목차)

1. [Github OAuth 권한 부여 API Docs](https://docs.github.com/ko/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
2. [Github User API Docs](https://docs.github.com/ko/rest/users/users?apiVersion=2022-11-28#get-a-user-using-their-id)
3. [JWT](https://jwt.io/)
4. [OAuth 2.0](https://oauth.net/2/)
5. [OAuth Standard](https://datatracker.ietf.org/doc/html/rfc6749)