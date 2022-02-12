---
layout: post
title: "Spring Boot: How to disable Flyway in production"
---

This is a very simple question, yet the answer is not so easy to find in the Spring Boot docs. Stop googling and do the following:


If you have `application.properties`:

```
spring.flyway.enabled=false
```

or `application.yml`:

```yml
spring:
    flyway:
        enabled: false
```

The official docs are here https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.data-migration.spring.flyway.enabled.

I'm also recommending reading about [Spring profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles).
The profiles fit perfectly for this requirement, so you can disable Flyway only for certain profile (environment). Example:


```yml
# Default, production configuration in application.yml
spring:
    flyway:
        enabled: false
```


```yml
# Development configuration in application-development.yml
spring:
    flyway:
        enabled: true
```
