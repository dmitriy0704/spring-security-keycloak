# Настройка Keycloak по шагам

## Создать realm:

-> Manage realms 
--> create realm

## Создание пользователя:

-> Users
--> Add user
--> Указать Имя пользователя, Email - верифицированный
---> Credentials:
----> Добавить пароль, Temporary - off

## Создание клиента:

-> Clients
--> General setting: Указать client_id
--> Capability Config: указать Client authentication = on, "Standard Flow" и "Direct access grand"
--> Login Setting: Root URL: http://localhost:8081, Valid redirect URL: http://localhost:8081/*
--> Credentials: Client  Secret - скопировать

## Аутентификация пользователя:

HTTP request:

```text
POST http://localhost:8080/realms/{realms_name}/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=springsecurity&client_secret=Bmuu0eVpFB0YhBQ5k5WASrF3qYxyBYdS&username=j.deniels&password=password&grant_type=password
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

## Получает токен:

```text
### POST to keycloak
POST http://localhost:8080/realms/eselpo/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=springsecurity&client_secret=Bmuu0eVpFB0YhBQ5k5WASrF3qYxyBYdS&username=j.deniels&password=password&grant_type=password

```

Полученный токен вставляем:

```text
### GET информация о странице:
GET http://localhost:8081/authenticated.html
Authorization: Bearer eyJhb...
```

## Добавление ролей в Keycloak:

-> Realm role: добавляем роль из SecurityFilterChain(ROLE_MANAGER)

## Добавляем роль пользователю:

-> Users -> Role mappings --> Assign --> Выбрать роль