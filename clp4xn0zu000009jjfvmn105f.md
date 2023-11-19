---
title: "GraphQL client service implementation with Spring Boot and Netflix DGS"
datePublished: Sun Nov 19 2023 03:42:50 GMT+0000 (Coordinated Universal Time)
cuid: clp4xn0zu000009jjfvmn105f
slug: graphql-client-service-implementation-with-spring-boot-and-netflix-dgs
tags: java, graphql, springboot, netflixdgs

---

This article discusses how to implement a GraphQL client service with Spring WebFlux and Netflix DGS framework.

## **Use Case**

Let's say we have to implement a service/API that needs to call a GraphQL service and get the response for the below-given use case.

GraphQL client service needs to call a backend GraphQL service and get user information.

* Get a List of Users for given search criteria.
    
* Get a specific User for a given ID.
    

# **Tech Stack**

* Java 11
    
* Spring Boot WerbFlux 2.7.x
    
* Netflix DGS 5.5.x
    
* Gradle.
    

Find this article-specific code implementation here

[GraphQL client implementation](https://github.com/uthircloudnative/graphql-client-api/tree/feature_graphql_simpleclient)

# **Implementation Details**

We need to create a template project with all the necessary dependencies.

We can use [**https://start.spring.io/**](https://start.spring.io/) to bootstrap our project with the following given dependencies.

```java
plugins {
	id 'java'
	id 'org.springframework.boot' version '2.7.17'
	id 'io.spring.dependency-management' version '1.0.15.RELEASE'
}

group = 'com.learntech'
version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '11'
}

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

ext {
	set('springCloudVersion', "2021.0.8")
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

dependencyManagement {
	imports {
		mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
		mavenBom 'com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:5.5.5'
	}
}
```

> ***Here main thing to notice is to make sure we are choosing the correct Spring boot-compatible Netflix DGS framework version. In our case, we are using the spring boot 2.7.x version its equivalent compatible DGS version is 5.5.x.***

### **GraphQL Schema**

We are going to create this client service also a GraphQL-based service.

So we have to define its schema.

> ***In the spring boot application, we have to define query schema under the resources/schema/schema.graphqls folder as it's the default path where schema will be defined.***

```java


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

input SearchInput {
    firstName: String!
    lastName: String!
    dateOfBirth: String
}
```

We are exposing this client service as a GraphQL service. It will support 2 types of queries.

* users -&gt; To return a list of users for given criteria. We made this query input as a custom object **SearchInput** to support multiple search criteria.
    
* user -&gt; To return a specific given user based on ID.
    

Both these queries are defined with their respective return type objects.

As this client service will make a call to another GraphQL service to get the above user details, we need to define that backend service GraphQL queries. Let's define those service queries in two files under the **resources/query** folder.

> userSearch.graphql //To get list of Users for given search criteria

```java
query users($searchInput : SearchInput){
    users(searchInput: $searchInput){
            id
            firstName
            lastName
            dateOfBirth
            gender
            address {
                type
                street1
                street2
                city
                state
                zip
            }
            phone {
                type
                countryCode
                number
            }
        }
}
```

> searchByUserId.graphql //To get a single User for a given ID

```java
query user($id : ID!){
    user(id : $id){
        id
        firstName
        lastName
        dateOfBirth
        gender
        address {
            type
            street1
            street2
            city
            state
            zip
        }
        phone {
            type
            countryCode
            number
        }
}
}
```

# Code

Our code structure will look like below.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700357237346/6dc2e6e0-ddc4-4758-9d5d-85d0e9d5acf5.png align="center")

## Model package

* We need to define the domain model for client service and its underlying backend service. In our case, for simplicity, both client and backend service models are defined with the same naming convention so we can reuse them.
    
* **User, Address, and Phone** classes will represent the actual domain model which carries the user data both in client and backend service.
    
* **SearchInput, SearchUserResponse, UserResponse** will be supporting model classes for input and backend GraphQL service response model.
    

### Config package

```java
package com.learntech.graphqlclientapi.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(){
        return WebClient.builder().build();
    }
}
```

* As our client service needs to make GraphQL calls to the backend service we need to define WebClient. Spring boot Webflux provides an HTTP-based **WebClient** to support this.
    
* It's based on a reactive stack and supports both blocking and non-blocking execution patterns. To make the backend HTTP call we need to define a Bean of type **Webclient**. It has so many custom configuration setups. But for this example, we will define a simple basic **Webclient** bean with default configurations.
    

### DataFetcher package

```java
package com.learntech.graphqlclientapi.datafetcher;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.learntech.graphqlclientapi.model.*;
import com.learntech.graphqlclientapi.service.UserSearchService;
import com.netflix.graphql.dgs.DgsComponent;
import com.netflix.graphql.dgs.DgsQuery;
import com.netflix.graphql.dgs.InputArgument;
import lombok.extern.slf4j.Slf4j;
import reactor.core.publisher.Mono;

import java.util.List;

@DgsComponent
@Slf4j
public class UserDatafetcher {

    private final UserSearchService userSearchService;

    public UserDatafetcher(UserSearchService userSearchService) {
        this.userSearchService = userSearchService;
    }

    @DgsQuery
    public Mono<List<User>> users(@InputArgument SearchInput searchInput) throws JsonProcessingException {
        log.info("users() Starts");
        return userSearchService.searchUsers(searchInput);
    }

    @DgsQuery
    public Mono<User> user(@InputArgument Integer id){
        log.info("user() Starts");
        return userSearchService.searchById(id);
    }

}
```

* **UserDataFetcher** class will act as the controller that will receive incoming requests and resolve the correct method to be executed for a given query. **@DgsComponent** will make this class a spring-managed bean.
    
* Using **@DgsQuery** framework will identify which method to be executed based on the given query name. Here method name and query name in schema match so we don't need to explicitly define query name.
    

### Service package

```java
package com.learntech.graphqlclientapi.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.learntech.graphqlclientapi.model.SearchInput;
import com.learntech.graphqlclientapi.model.User;
import com.learntech.graphqlclientapi.model.UserResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Service;
import org.springframework.util.FileCopyUtils;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Service
@Slf4j
public class UserSearchServiceImpl implements UserSearchService{
    private static final String USER_SEARCH_URL = "http://localhost:8080/graphql";

    private final WebClient webClient;
    private final ObjectMapper objectMapper;

    public UserSearchServiceImpl(WebClient webClient,
                                 ObjectMapper objectMapper) {
        this.webClient    = webClient;
        this.objectMapper = objectMapper;
    }

   
    @Override
    public Mono<List<User>> searchUsers(SearchInput searchInput) throws JsonProcessingException {
      //1. Set any header params
       MultiValueMap<String, String> header = new LinkedMultiValueMap<>();
       header.add("trace-id", UUID.randomUUID().toString());
       header.add("Content-Type", "application/json");

       //2.Get backend service GraphQL query
       String query = getQuery("/query/","userSearch.graphql");

        //3.Define query input 
        Map<String, Object> variableMap = new HashMap<>();
        variableMap.put("searchInput", Map.of("firstName","Jhon","lastName","Vicky"));
        //4.Define GraphQL request query & variables
        Map<String, Object> request = new HashMap<>();
        request.put("query",query);
        request.put("variables",variableMap);
        //5.Make call to backend service using webClient
        return webClient.post()
                       .uri(USER_SEARCH_URL)
                       .headers(headers -> headers.addAll(header))
                       .body(BodyInserters.fromValue(request))
                       .retrieve()
                       .bodyToMono(UserResponse.class)
                       .flatMap(resp -> transform(resp));
    }

    private Mono<List<User>> transform(UserResponse userResponse) {
        return Mono.just(userResponse.getData().getUsers());
    }

    @Override
    public Mono<User> searchById(Integer id) {
        log.info("searchById() Starts");
        MultiValueMap<String, String> header = new LinkedMultiValueMap<>();
        header.add("trace-id", UUID.randomUUID().toString());
        header.add("Content-Type", "application/json");

        String query = getQuery("/query/","searchByUserId.graphql");

        Map<String, Object> request = new HashMap<>();
        request.put("query",query);

        Map<String, Object> variablesMap = new HashMap<>();
        variablesMap.put("id", id);

        request.put("variables", variablesMap);

        return webClient.post()
                .uri(USER_SEARCH_URL)
                .headers(headers -> headers.addAll(header))
                .body(BodyInserters.fromValue(request))
                .retrieve()
                .bodyToMono(UserResponse.class)
                .flatMap(resp -> getUserData(resp));
    }

    private Mono<User> getUserData(UserResponse userResponse){
       return Mono.just(userResponse.getData().getUser());
    }

    private String getQuery(String filePath, String queryName)  {
        ClassPathResource classPathResource = new ClassPathResource(filePath+queryName);
        try {
           return new String(FileCopyUtils.copyToByteArray(classPathResource.getInputStream()));
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

* **UserSearchServiceImpl** will have the implementation to call the backend service to fetch users & user query data. It is an HTTP POST call we can pass the request header, body information.
    
* In step 1 we are setting header params. Though for our use case, it is optional we just included it for informational purposes.
    
* Step 2 get the GraphQL query from the classpath file by passing the correct filename and path.
    
* Step 3 will set the input values for the query. In GraphQL we will set the values as key-value pairs.
    
* Step 4 we will set both query and variables in the single map and pass it to the client. It will internally convert it into a combined graphQL query and variables json and make an HTTP call.
    
* Step 5 finally passes these values to WebClient and makes a retrieve() method call which in turn, helps to return a Mono body object that contains the GrapQL schema response with two params (Key/Value pair). It will contain **data** and/or **error** key. If no errors data key will have the requested query response.
    
* In our case once method execution is completed we parse the response object using the transform method and get the intended/requested response object.
    

> For this basic implementation we haven't covered error handling. It will only consider a happy path scenario.

### Test the application

It's time to test our application. In spring boot we can enable an inbuilt client(Graphiql) which provides an easy user-friendly option to test this implementation.

To enable it add the below property in the [**application.properties**](http://application.properties/)

```java
#http://localhost:8081/graphiql
spring.graphql.graphiql.enabled=true

server.port=8081
```

Now when we start the application it will start by default in the 8080 port but in this config, we changed that port 8081 and we can access Graphiql with this URL

[**http://localhost:8081/graphiql**](http://localhost:8080/graphiql)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700364210656/5ca67bba-ef1d-40ed-a4fc-a7b836eb1534.png align="center")

> To test this application end to end run this code and below given GraphQL service.
> 
> [GraphQL server service implementation](https://github.com/uthircloudnative/graphql-server-api/tree/feature_simplegrapgqlserviceimpl)

This will conclude our simple GraphQL client service implementation. Happy learning.