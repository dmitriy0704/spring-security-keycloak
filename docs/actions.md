# Настройка и запуск Keycloak по шагам

## Запуск

При локальном запуске перейти в папку keycloack/bin и запустить ./kc.sh
start-dev.
Keycloack запустится на порту 8080.

## SERVER

## Создать realm:

-> Manage realms
--> create realm

## Создание пользователя:

(пользователь создается внутри realm)

```text
-> Users
--> Add user
--> Указать Имя пользователя, Email - верифицированный
---> Credentials:
----> Добавить пароль, Temporary - off

```

## Создание клиента:

```text
-> Clients
--> General setting: Указать client_id
--> Capability Config: указать Client authentication = on, "Standard Flow" и "Direct access grand"
--> Login Setting: Root URL: http://localhost:8081, Valid redirect URL: http://localhost:8081/*
--> Credentials: Client  Secret - скопировать

```

# Настройка приложения:

## Зависимости:

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

## Аутентификация пользователя:

HTTP request:

```sh
POST http://localhost:8080/realms/{realms_name}/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=springsecurity&client_secret=Bmuu0eVpFB0YhBQ5k5WASrF3qYxyBYdS&username=j.deniels&password=password&grant_type=password
```

## Добавить бин

```java

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.oauth2ResourceServer(
            oauth2 ->
                    oauth2.jwt(Customizer.withDefaults()));

    return http
            .authorizeHttpRequests(c -> c.requestMatchers("/error").permitAll()
                    .requestMatchers("/manager.html").hasRole("MANAGER")
                    .anyRequest().authenticated())
            .build();
}
```

## Получаем токен:

```sh
### POST to keycloak
POST http://localhost:8080/realms/eselpo/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=springsecurity&client_secret=Bmuu0eVpFB0YhBQ5k5WASrF3qYxyBYdS&username=j.deniels&password=password&grant_type=password

```

Полученный токен вставляем, чтобы проверить аутентификацию:

```text
### GET информация о странице:
GET http://localhost:8081/authenticated.html
Authorization: Bearer eyJhb...
```

## Добавляем роли в Keycloak:

```text
  .requestMatchers("/manager.html").hasRole("MANAGER")
```

Для доступа к странице, доступной с определенной ролью, эту роль надо создать
в Keycloack.

```text
-> Realm role: добавляем роль из SecurityFilterChain(ROLE_MANAGER)
```

Далее добавляем роль пользователю:

```text
-> Users -> Role mappings --> Assign --> Выбрать роль
```

Далее создаем бин, он из JWT получает роль, созданную нами ранее(MANAGER),
с которой можно получить доступ к странице(manager.html)

```java

@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    var converter = new JwtAuthenticationConverter();
    var jwtGrantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
    converter.setPrincipalClaimName("preferred_username");
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        var authorities = jwtGrantedAuthoritiesConverter.convert(jwt);
//            var roles = jwt.getClaimAsStringList("spring_sec_roles");
        var roles = (List<String>) jwt.getClaimAsMap("realm_access").get("roles");

        return Stream.concat(authorities.stream(),
                        roles.stream()
                                .filter(role -> role.startsWith("ROLE_"))
                                .map(SimpleGrantedAuthority::new)
                                .map(GrantedAuthority.class::cast))
                .toList();
    });

    return converter;
}
```

Еще вытаскивать роли можно маппером из Keycloack.

```text
Client scopes -> roles -> Mappers -> Add -> By configuration ->
User realm role:
Указать:
- multivalue
- Token Claim Name - в каком свойстве токена выводятся роли, "spring_sec_roles"
- включить: Add to ID token, Add to access token, Add to userinfo    
```

Далее в Clients -> Clients scope -> Evaluate -> Generated access token

Теперь:

```java

@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    var converter = new JwtAuthenticationConverter();
    var jwtGrantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
    converter.setPrincipalClaimName("preferred_username");
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        var authorities = jwtGrantedAuthoritiesConverter.convert(jwt);
        var roles = jwt.getClaimAsStringList("spring_sec_roles");

        return Stream.concat(authorities.stream(),
                        roles.stream()
                                .filter(role -> role.startsWith("ROLE_"))
                                .map(SimpleGrantedAuthority::new)
                                .map(GrantedAuthority.class::cast))
                .toList();
    });

    return converter;
}
```

---

# CLIENT

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

В application.yml добавляем "client":

```yaml
       spring:
       security:
         oauth2:
           resourceserver:
             jwt:
               issuer-uri: http://localhost:8080/realms/eselpo
           client:
             provider:
               keycloak:
                 issuer-uri: http://localhost:8080/realms/eselpo
                 user-name-attribute: preferred_username
             registration:
               keycloak:
                 client-id: springsecurity
                 client-secret: Bmuu0eVpFB0YhBQ5k5WASrF3qYxyBYds
                 scope: openid
```

Далее корректируем securityFilterChain:

```java

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.oauth2ResourceServer(
                oauth2 -> oauth2.jwt(Customizer.withDefaults()));
     //-> Добавили эту строку:
        http.oauth2Login(Customizer.withDefaults());

        return http
                .authorizeHttpRequests(c -> c.requestMatchers("/error").permitAll()
                        .requestMatchers("/manager.html").hasRole("MANAGER")
                        .anyRequest().authenticated())
                .build();
    }
```

Далее открываем http://localhost:8081/authenticated.html

Попадаем на страницу аутенитификации Keycloak

Вводим: j.deniels , password.

Но для страницы manager.html не хватает прав. 

### Добавляем права

```java
  // OidcUserRequest - способ получения информации и пользователе
    // OidcUser - информация, которая была возвращена от OIDC - провайдера
    @Bean
    public OAuth2UserService<OidcUserRequest, OidcUser> oAuth2UserService() {
        var oidcUserService = new OidcUserService();
        return userRequest -> {
            var oidcUser = oidcUserService.loadUser(userRequest);
            var roles = oidcUser.getClaimAsStringList("spring_sec_roles");
            var authorities = Stream.concat(oidcUser.getAuthorities().stream(),
                            roles.stream()
                                    .filter(role -> role.startsWith("ROLE_"))
                                    .map(SimpleGrantedAuthority::new)
                                    .map(GrantedAuthority.class::cast))
                    .toList();

            return new DefaultOidcUser(authorities, oidcUser.getIdToken(), oidcUser.getUserInfo());
        };
    }
```

Заходим на страницу http://localhost:8081/manager.html с ролью MANAGER


