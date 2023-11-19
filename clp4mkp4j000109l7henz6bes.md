---
title: "A simple GraphQL service with Spring Boot and Netflix DGS"
datePublished: Sat Nov 18 2023 22:33:06 GMT+0000 (Coordinated Universal Time)
cuid: clp4mkp4j000109l7henz6bes
slug: a-simple-graphql-service-with-spring-boot-and-netflix-dgs
tags: graphql, springboot, dgs, netflixdgs

---

This article discusses how to implement a GraphQL server-side service with Spring WebFlux and Netflix DGS framework.

## Use Case

Let's say we have to implement a service/API that will return user information

for the given input.

This service will expose two query endpoints.

* To get a List of Users for given search criteria.
    
* To get a specific User for a given ID.
    

# Tech Stack

Java 11 or above

Spring Boot WerbFlux 2.7.x

Netflix DGS 5.5.x

Gradle

Find this article-specific code implementation here

[Simple Graphql service implementation](https://github.com/uthircloudnative/graphql-server-api/tree/feature_simplegrapgqlserviceimpl)

# Implementation Details

### Dependencies

We need to create a template project with all the necessary dependencies.

We can use [https://start.spring.io/](https://start.spring.io/) to bootstrap our project with the following given dependencies.

```plaintext
plugins {
	id 'java'
	id 'org.springframework.boot' version '2.7.17'
	id 'io.spring.dependency-management' version '1.0.15.RELEASE'
}

dependencyManagement {
	imports {
		mavenBom 'com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:5.5.5'
	}
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-webflux'
	implementation 'com.netflix.graphql.dgs:graphql-dgs-webflux-starter:5.5.5'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'io.projectreactor:reactor-test'
}
```

> Here main thing to notice is to make sure we are choosing the correct Spring boot-compatible Netflix DGS framework version. In our case, we are using the spring boot 2.7.x version its equivalent compatible DGS version is 5.5.x.

### GraphQL Schema

Now let's define our GraphQL schema as per our use case. As per our use case, we will define the below schema to support our use case.

> In the spring boot application, we have to define query schema under the **resources/schema/schema.graphqls** folder as it's the default path where schema will be defined.

```graphql

type Query {
    users(searchInput: SearchInput) : [User]
    user(id : ID): User
}

type UserResponse{
    users: [User]
}

type User {
    id: ID
    firstName: String
    lastName: String
    dateOfBirth: String
    gender: String
    address: [Address]
    phone: [Phone]
}

type Address {
    type: String
    street1: String
    street2: String
    city: String
    state: String
    zip: Int
}

type Phone {
    type: String
    number: String
    countryCode: String
}

// In input firstName & lastName is mandatory
input SearchInput {
    firstName: String!
    lastName: String!
    dateOfBirth: String
}
```

As per our use case, we defined two queries.

* users -&gt; To return a list of users for given criteria. We made this query input as a custom object **SearchInput** to support multiple search criteria.
    
* user -&gt; To return a specific given user based on ID.
    

Both these queries are defined with their respective return type objects.

### The code

Now let's write our core application code. For the sake of simplicity, we will be hard coding service response.

Our application code structure will look like below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700342442623/4a4c0fb0-ebc5-4d04-b983-6c81ed27a96c.png align="center")

We need to define a domain model as per our schema. So we defined **User, Address, Phone, SearchInput, and UserResponse** domain objects compatible with our query schema definition. All these objects will have required files and those will be accessed through Lombok library-generated getters/setters.

Next, define our DataFetcher to receive requests as per schema propagate the request to the next layer (Service layer) and get the response.

```java
@DgsComponent
@Slf4j
public class UserDataFetcher {

    private final UserSearchService userSearchService;

    public UserDataFetcher(UserSearchService userSearchService) {
        this.userSearchService = userSearchService;
    }

    @DgsQuery
    public Mono<List<User>> users(@InputArgument SearchInput searchInput) throws JsonProcessingException {
        List<User> users = userSearchService.searchUser(searchInput);
        return Mono.just(users);
    }

    @DgsQuery
    public Mono<User> user(@InputArgument Integer id){
        User user = userSearchService.searchById(id);
        return Mono.just(user);
    }
}
```

**@DgsComponent** will make this class a spring context bean which will act as a kind of request handler.

Using **the @DgsQuery** framework will resolve the query name and execute the respective query-matching request method in the DataFetcher class. Here we defined our method as **users** which is the same as our schema so we don't need to specify explicitly the query name in **@DgsQuery.**

Also, we are using Spring Boot reactive stack (Webflux) for our implementation. So we can implement a Non-Blocking type of functional programming model. Each method will return the Mono&lt;Object&gt; type.

Now it's time for our Service layer implementation to handle this request and respond.

```java
@Service
@Slf4j
public class UserSearchServiceImpl implements UserSearchService {

    @Override
    public List<User> searchUser(SearchInput searchInput) {
        log.info("searchUser Starts");
        return Arrays.asList(getUser());
    }

    @Override
    public User searchById(Integer id) {
        return getUser();
    }

    private User getUser() {
        User user = new User();
        user.setId(1);
        user.setFirstName("Jhon");
        user.setLastName("Victor");
        user.setGender("M");
        user.setDateOfBirth(LocalDate.now().toString());

        Address address = new Address();
        address.setType("Home");
        address.setStreet1("1234 First Street");
        address.setStreet2("Apt 9");
        address.setCity("Louisville");
        address.setState("NY");
        address.setType("Home");
        address.setZip(12345);

        user.setAddress(Arrays.asList(address));

        Phone phone = new Phone();

        phone.setType("Mobile");
        phone.setCountryCode("+1");
        phone.setNumber("1234567890");

        user.setPhone(Arrays.asList(phone));

        return user;
    }
}
```

To make it simple we hard-coded user object creation and returned respective required users or user objects.

### Test the Application

It's time to test our application. In spring boot we can enable an inbuilt client(Graphiql) which provides an easy user-friendly option to test this implementation.

To enable it add the below property in the **application.properties**

```java

spring.graphql.graphiql.enabled=true
```

Now when we start the application it will start by default in the 8080 port and we can access Graphiql with this URL [http://localhost:8080/graphiql](http://localhost:8080/graphiql)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700345913789/98262b85-2882-4936-ae91-2e15a01eb3e4.png align="center")

This will conclude our simple GraphQL server service implementation. Happy learning.