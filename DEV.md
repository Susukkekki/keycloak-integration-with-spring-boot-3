# 참고

- [Spring boot 3 Keycloak integration for beginners | The complete Guide](https://www.youtube.com/watch?v=vmEWywGzWbA)

- [참고](#참고)
  - [실행](#실행)
    - [Keycloak 실행](#keycloak-실행)
      - [렐름 환경 설정](#렐름-환경-설정)
      - [토큰 얻기](#토큰-얻기)
    - [spring boot app 실행](#spring-boot-app-실행)
      - [Test](#test)

## 생각해 보기

이 예제에서는 client role 을 사용하는데 realm role 과 어떤 차이가 있으며 use case 는 어떻게 될까?

[How are Keycloak roles managed?](https://stackoverflow.com/questions/47837613/how-are-keycloak-roles-managed) 에 설명이 잘 되어 있다.

Realm role 은 전체 적용, Client Role 은 해당 Client 에만 적용된다. 따라서 비즈니스 요건에 따라 Role 의 범위를 규정하고 올바른 것을 사용하면 된다는 것이다.

## 실행

### Keycloak 실행

```bash
docker-compose up -d
```

http://localhost:9090

#### 렐름 환경 설정

admin / admin 으로 로그인

- Alibou 렐름 생성

- Create client
  - General Settings
    - Client type : `OpenID Connect`
    - Client ID : `alibou-rest-api`
    - Name : `Demo client for Alibou REST API`
    - Description : Some description
  - Capability config
    - `default`
      - Client authentication : Off
      - Authorization : Off
      - Authentication flow : Standard flow and Direct access grants checked
    - Login settings
      - Root URL : `http://localhost:8081`
      - Home URL : `http://localhost:8081`
      - Valid redirect URIs : `http://localhost:8081/*`
      - Valid post logout redirect URIs : `http://localhost:8081`
      - Web origins : `*`

- Create role
  - admin
    - Role name : `admin`
    - Role description : Admin role
  - user
    - Role name : `user`
    - Role description : User role

- Create user
  - Username : `alibou`
  - After a user
    - Credentials
      - Set password : `alibou`
      - Temporary : `Off`

- Update Client : `alibou-rest-api`
  - Roles
    - Role name : `client_admin`
    - Role name : `client_user`

- Realm roles
  - Select `admin`
    - Action > Add associated roles
    - Select `Filter by clients`
    - Check `client_admin`
    - This will mark admin role as composite.
  - Select `user`
    - Action > Add associated roles
    - Select `Filter by clients`
    - Check `client_user`
    - This will mark user role as composite.

- Users
  - Select `alibou`
  - Select the Role mapping tab
    - Click `Assign role` button
      - Check admin role
    - Click `Assign role` button again
    - Select `Filter by clients`
      - Check `client_admin`

- http://localhost:9090/realms/Alibou/.well-known/openid-configuration

#### 토큰 얻기

- token_endpoint: http://localhost:9090/realms/Alibou/protocol/openid-connect/token

```bash
curl -X POST http://localhost:9090/realms/Alibou/protocol/openid-connect/token \
   -H "Content-Type: application/x-www-form-urlencoded"  \
   -d "grant_type=password&client_id=alibou-rest-api&username=alibou&password=alibou" 
```

그러면 다음과 같이 토큰을 얻는다.

```json
{
    "access_token": "eyJhbGciOiJS...",
    "expires_in": 300,
    "refresh_expires_in": 1800,
    "refresh_token": "eyJhbGciOiJI...",
    "token_type": "Bearer",
    "not-before-policy": 0,
    "session_state": "820e37bc-d377-400c-9802-18b0831e17eb",
    "scope": "email profile"
}
```

access token 을 http://jwt.io 에서 확인해 보자.

Header : ALGORITHMS & TOKEN TYPE

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "67ZPp1YUGWqoVS42tnsMpRBaPGgtEpRmSV_c4N5Rwq0"
}
```

PAYLOAD : DATA

```json
{
  "exp": 1715563660,
  "iat": 1715563360,
  "jti": "252b1eeb-5087-476f-8431-fafa646c2495",
  "iss": "http://localhost:9090/realms/Alibou",
  "aud": "account",
  "sub": "e11adba2-3a43-431d-9095-3a0ea22896f5",
  "typ": "Bearer",
  "azp": "alibou-rest-api",
  "session_state": "820e37bc-d377-400c-9802-18b0831e17eb",
  "acr": "1",
  "allowed-origins": [
    "*"
  ],
  "realm_access": {
    "roles": [
      "offline_access",
      "admin",
      "uma_authorization",
      "default-roles-alibou"
    ]
  },
  "resource_access": {
    "alibou-rest-api": {
      "roles": [
        "client_admin"
      ]
    },
    "account": {
      "roles": [
        "manage-account",
        "manage-account-links",
        "view-profile"
      ]
    }
  },
  "scope": "email profile",
  "sid": "820e37bc-d377-400c-9802-18b0831e17eb",
  "email_verified": false,
  "preferred_username": "alibou",
  "given_name": "",
  "family_name": ""
}
```

VERIFY SIGNATURE : 생략

### spring boot app 실행

```bash
mvn install
```

```bash
mvn spring-boot:run
```

#### Test

TOKEN 환경 변수 생성

```bash
TOKEN=$(curl -X POST http://localhost:9090/realms/Alibou/protocol/openid-connect/token \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=password&client_id=alibou-rest-api&username=alibou&password=alibou" \
    | jq -r '.access_token')
echo $TOKEN
```

```bash
curl -v -H "Authorization: Bearer ${TOKEN}" http://localhost:8081/api/v1/demo/hello-2
```

HTTP/1.1 200

```bash
curl -v -H "Authorization: Bearer ${TOKEN}" http://localhost:8081/api/v1/demo
```

HTTP/1.1 403
