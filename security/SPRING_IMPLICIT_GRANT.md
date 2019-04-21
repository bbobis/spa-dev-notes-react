### Implicit Grant
Notes on how to make a backend service a **Resource Server** and configuring it to handle `access_token`

In order to make our spring backend service a resource server, we can either use **Spring Security 5** and **Spring Boot 2.1** or we can use **Okta Spring Boot Starter** which also leverages the former 2 libraries but made it compact.

_NOTE: Okta is used as the Authorization Server provider using OpenID connect for authentication._

#### Using Spring Security 5.0 and Spring Boot 2.1

---
POM.xml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```
application.yml
```yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: {{issuer_url}}
```
##### Using Okta Spring Boot Starter

---
- Using Okta's library validates the token's
  - Signature
  - Issuer (`iss`)
  - Audience claim (`aud`)
  - Cliend Id claim (`cid`)
  - Expiry date (`exp`)

POM.xml
```xml
<dependency>
    <groupId>com.okta.spring</groupId>
    <artifactId>okta-spring-boot-starter</artifactId>
    <version>1.1.0</version>
</dependency>
```
application.yml
```yml
okta:
  oauth2:
    clientId: {{client_id}}
    issuer: {{issuer_uri}}
```
When using okta's spring boot starter, we need to configure `WebSecurityConfigurerAdapter` to let spring know that we want to handle jwt tokens

NOTE: If you're using `spring-boot-starter-oauth2-resource-server`, we don't need to configure this as this is already the default behaviour.

- Create a configuration class that extends `WebSecurityConfigurerAdapter`

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests().anyRequest().authenticated()
            .and()
            .oauth2ResourceServer().jwt();
    }
}
```
#### Adding groups in Okta as claims in access_token

---
We can lockdown endpoints (methods) / whole resource (classes) using Okta's groups.

- In Okta, create a group and add user to that group
- Go to your authorization server and add a custom claim to the `access_token` 
    - Name: groups
    - Include in token type: Access Token
    - Value type: Groups
    - Filter: Matches regex - `.*`
- The groups claim should now be included on your access token and will be converted to grants in spring.
- If you are using the okta library, it already knows how to convert the `groups` claim into granted authorities in spring. This all work via naming convention. If you change the claim name to `roles` instead of `groups` (default). You have to make a config change in your `application.yml` file for the okta library to use that claim name
```yml
okta:
  oauth2:
    groupsClaim: roles
```
- If you are using spring security, only scopes (`scp` claim) are converted to granted authorities. We need to let spring know to also look into the `groups` claim and add those as granted authorities as well.
```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests().anyRequest().authenticated()
                .and()
                .oauth2ResourceServer().jwt()
                .jwtAuthenticationConverter(grantedAuthoritiesExtractor());
    }

    Converter<Jwt, AbstractAuthenticationToken> grantedAuthoritiesExtractor() {
        return new GrantedAuthoritiesExtractor();
    }

    @SuppressWarnings("unchecked")
    static class GrantedAuthoritiesExtractor extends JwtAuthenticationConverter {
        protected Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
            Collection<String> groupsClaim = (Collection<String>)
                    jwt.getClaims().get("groups");

            Collection<String> scopesClaim = (Collection<String>)
                    jwt.getClaims().get("scp");

            return Stream.concat(groupsClaim.stream(), scopesClaim.stream())
                    .map(s -> "ROLE_" + s)
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toList());
        }
    }
}
```

**FYI** - we can remove the prefix `ROLE_` by creating a bean of type `GrantedAuthorityDefaults` if we want so that we don't have to prefix our groups and scopes.
```java
@Bean
GrantedAuthorityDefaults grantedAuthorityDefaults() {
    return new GrantedAuthorityDefaults(""); // Remove the ROLE_ prefix
}
```

#### Locking down endpoints

---
In order to lockdown endpoints, we can use `@PreAuthorize` annotation.

- On a config file enable it by using `@EnableGlobalMethodSecurity`
```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests().anyRequest().authenticated()
            .and()
            .oauth2ResourceServer().jwt();
    }
}
```
- You can now lockdown a method or a controller like so
```java
@RestController
@RequestMapping("/api/employees")
@PreAuthorize("hasRole('Staff')")
public class EmployeeController {
    private EmployeeRepository employeeRepository;

    @Autowired
    public EmployeeController(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    @GetMapping
    public List<Employee> getEmployees(){
        return StreamSupport.stream(employeeRepository.findAll().spliterator(), false).collect(Collectors.toList());
    }
}
```
