# Spring Higher-Order Components

[![Build Status](https://travis-ci.org/jpomykala/spring-higher-order-components.svg?branch=master)](https://travis-ci.org/jpomykala/spring-higher-order-components)

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.jpomykala/spring-higher-order-components/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.jpomykala/spring-higher-order-components)

[![codecov](https://codecov.io/gh/jpomykala/spring-higher-order-components/branch/master/graph/badge.svg)](https://codecov.io/gh/jpomykala/spring-higher-order-components)

Boilerplate components for Spring Boot. 
- Sending e-mails with ease (Amazon SES)
- Request logging 
- Uploading files to Amazon S3
- Response wrapping
- Custom CORS filter

## Installation
```xml
<dependency>
  <groupId>com.jpomykala</groupId>
  <artifactId>spring-higher-order-components</artifactId>
  <version>{spring-hoc.version}</version>
</dependency>
```

[Check in maven repository](https://mvnrepository.com/artifact/me.jpomykala.hoc/spring-higher-order-components)

## Motivation

- Write inline code
- Duplicate code a few times in different spots
- Extract duplicate code into methods
- Use your abstractions for a while
- See how that code interacts with other code
- Extract common functionality into internal library
- Use internal library for extended periods of time
- Really understand how all of the pieces come together
- Create external open source library (we are here now)

source: [https://nickjanetakis.com](https://nickjanetakis.com/blog/microservices-are-something-you-grow-into-not-begin-with)

## @EnableEmailSending

This component gives you simple API to send emails using Amazon SES service. 

### Configuration

- Provide **verified** sender email address ``spring-hoc.mail.sender-email-address``
- Provide AWS credentials ``spring-hoc.aws.access-token``, ``spring-hoc.aws.secret-key``, ``spring-hoc.aws.region``

#### Example `application.yml`

```yml
spring-hoc:
  aws:
    access-token: xxxxxxxx
    secret-key: xxxxxxxx
    region: eu-west-1
  mail:
    sender-email-address: no-reply@mydomain.com    
```

Spring HOC will automatically create for you Amazon SES component if bean doesn't exit.

### How to send e-mail?

Use ``EmailRequest`` step builder to create request.

```java
EmailRequest.builder()
            .to("jpomykala@example.com")
            .subject("Hey, I just met you and this is crazy")
            .body("But here's my number, so call me maybe")
            .build();
```

Now it's time to send email. You have 2 options here.
- ``@Autowire MailService`` and invoke ``sendEmail(EmailRequest)``.
- Publish ``EmailRequest`` using ``ApplicationEventPublisher``

That's all!

### Example application

```java
@SpringBootApplication
@EnableEmailSending
public class MySpringBootApplication {

  public static void main(String[] args) {
    SpringApplication.run(MySpringBootApplication.class, args);
  }

  // SEND E-MAIL BY EVENT PUBLISHING
  @Autowired
  private ApplicationEventPublisher eventPublisher;

  @GetMapping("/send-email-by-event-publishing")
  public void sendEmailByEventPublishing(){
    EmailRequest emailRequest = EmailRequest.builder()
            .to("jakub.pomykala@gmail.com")
            .subject("Hey, I just met you and this is crazy")
            .body("But here's my number, so call me maybe")
            .build();

    eventPublisher.publishEvent(emailRequest);
  }
  
  // SEND E-MAIL BY MAIL SERVICE
  @Autowired
  private MailService mailService;

  @GetMapping("/send-email-by-mail-service")
  public void sendEmailByMailService(){
    EmailRequest emailRequest = EmailRequest.builder()
            .to("jakub.pomykala@gmail.com")
            .subject("Hey, I just met you and this is crazy")
            .body("But here's my number, so call me maybe")
            .build();

    mailService.send(emailRequest);
  }
}
```


## @EnableRequestLogging

Adds logging requests, populate MDC with:
- user (IP address by default)
- requestId (UUID by default).

### Example application

```java
@SpringBootApplication
@EnableRequestLogging
public class MySpringBootApplication {

  public static void main(String[] args) {
    SpringApplication.run(ApiApplication.class, args);
  }

  @Autowired
  private MyUserService userService;

  // [OPTIONAL] customize configuration
  @Bean
  public LoggingFilter loggingFilter(LoggingFilterFactory loggingFilterFactory) {
    return loggingFilterFactory
            .withPrincipalProvider(new PrincipalProvider() {
              @Override
              public String getPrincipal(HttpServletRequest request) {
                return userService.findUserName(request);
              }
            })
            .createFilter();
  }
}

```


### More customization
```java
@Bean
public LoggingFilter loggingFilter(LoggingFilterFactory loggingFilterFactory){
  return loggingFilterFactory
          .withPrincipalProvider() // [optional] PrincipalProvider implementation 
          .withRequestIdProvider() // [optional] RequestIdProvider implementation
          .withCustomMdc("user", "[u:%s][rid:%s]") // [optional] MDC key, String.format()
          .createFilter();
}
```

## @EnableFileUploading

This annotation autoconfigures Amazon S3 component if bean doesn't exit.

``@Autowire UploadService`` gives you ability to upload files using overloaded methods:
- ``void upload(@NotNull UploadRequest uploadRequest)``
- ``void upload(@NotNull MultipartFile file)``
- ``void upload(@NotNull MultipartFile file, @NotNull String path)``
- ``void upload(byte[] bytes, String fileKey)``
- ``void upload(byte[] bytes, String fileKey, ObjectMetadata metadata)``
- ``String upload(byte[] bytes)`` // path is autogenerated (sha256 hash)

## example ``application.yml`` configuration

```
spring-hoc:
  aws:
    access-token: xxxxxxxx
    secret-key: xxxxxxxx
    region: eu-west-1
  cors:
    allowed-origins:
      - "http://localhost:3000"
  mail:
    sender-email-address: xxxxx@xxxx.com    
```

## @EnableResponseWrapping

Every `@RestController` output will be wrapped into `RestResponse<T>` object for JSON it will look like as follows:

```
{
  msg: "OK"
  status: 200
  data: <your data>
  pageDetails: <page details if you return Page from controller>
}
```

`RestResponse` static contructors:
  
  - `RestResponse ok(Object body)`
  - `RestResponse ok(Object body, PageDetails pageDetails)`
  - `RestResponse empty(String message, HttpStatus status)`
  - `RestResponse of(String message, HttpStatus status, Object data)`
  - `RestResponse of(String message, HttpStatus status, Object data, PageDetails pageDetails)`
  
Every output will be wrapped into `RestResponse` [see this issue](https://github.com/jpomykala/spring-higher-order-components/issues/4)

Response wrapping can be disabled for specific endpoinds by using `@DisableWrapping` annotation on method.

## @EnableCORS

This annotation adds filter which handles CORS requests. Right now you can configure only allowed origins using ``application.yml`` See example configuration below.


# Contribution

Would you like to add something or improve source? Create new issue, let's discuss it 

- **If in doubt, please discuss your ideas first before providing a pull request. This often helps avoid a lot of unnecessary work. In particular, we might prefer not to prioritise a particular feature for another while.**
- Fork the repository.
- The commit message should reference the issue number.
- Check out and work on your own fork.
- Try to make your commits as atomic as possible. Related changes to three files should be committed in one commit.
- Try not to modify anything unrelated.

# More
- Follow me on [Twitter](https://twitter.com/jakub_pomykala) :)
- Check out my [website](https://jpomykala.github.io)

# License
GNU General Public License v3.0
