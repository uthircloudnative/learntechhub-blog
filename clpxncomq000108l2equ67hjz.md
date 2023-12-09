---
title: "Spring Boot GraphQL service with InMemory DB (H2) integration"
datePublished: Sat Dec 09 2023 06:00:11 GMT+0000 (Coordinated Universal Time)
cuid: clpxncomq000108l2equ67hjz
slug: spring-boot-graphql-service-with-inmemory-db-h2-integration
tags: graphql, springboot, h2db

---

In this article, we will explore how to integrate an InMemory database in the Spring Boot application.

We will use H2 DB as our In-Memory DB. It's a popular open-source lightweight DB engine that can be easily embedded with the spring boot application.

H2 DB is mostly used for local development and integration testing purposes.

Refer below link to learn more about H2 DB

[H2 DB](https://www.h2database.com/html/main.html)

> H2 DB can support basic SQL functionalities and will operate on different popular DB vendor offering modes. However, it may not have all the features of actual DB offerings.

> Complete source code examples of this article can be found here.
> 
> [Code-Repository](https://github.com/uthircloudnative/graphql-server-api/tree/feature_graphqlinmemorydbint)

## **Use Case**

Let's say we have to implement a service/API that will return user information

for the given input.

This service will expose two query endpoints.

* To get a List of Users for given search criteria, for this use case let's have **firstName** and **lastName** as input.
    
* To get a specific User for a given ID.
    
* A user can have multiple types of address information.
    
* A user can have multiple types of phone information.
    

> **Note: We will be building this use case based on our earlier simple service implementation. Find the complete source code of the earlier implementation here.**
> 
> [SimpleGraphQL Service-Code Repository](https://github.com/uthircloudnative/graphql-server-api/tree/feature_simplegrapgqlserviceimpl)

# **Tech Stack**

Java 11 or above

Spring Boot WerbFlux 2.7.x

Netflix DGS 5.5.x

Gradle

H2 DB

# **Implementation Details**

### **Dependencies**

We need to have the following dependencies in our build file.

```java
dependencies {
	//implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-webflux'
	implementation(platform("com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:5.5.5"))
	implementation 'com.netflix.graphql.dgs:graphql-dgs-webflux-starter:5.5.5'
	//Spring Data JPA
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	//H2 DB Engine
    implementation 'com.h2database:h2'
	compileOnly 'org.projectlombok:lombok'
	
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'io.projectreactor:reactor-test'
}
```

Along with other dependencies we need 2 main dependencies specific to this use case.

* **com.h2database:h2** -&gt; Provide an embedded SQL DB.
    
* **org.springframework.boot:spring-boot-starter-data-jpa** -&gt; ORM implementation to support Data access and mapping between DB tables and Java applications.
    

With this configuration upon application startup, it will spin a SQL H2 DB engine.

> Any data stored in H2 DB won't be persisted by default. It will be available only life cycle of the application. Any application shutdown or restart will clean up the DB.

## DB Configuration

Now we have to specify the Database configuration for the application to connect and operate on.

We can define this in **application.yml** or **application.properties**. As we are developing an application locally and we intend to use H2 only for local development we can define that under local configuration.

This we can achieve using spring profile configuration. Let's define a local profile with the following configuration.

```yaml
spring:
  config:
    activate:
      on-profile: local
  graphql:
    graphiql:
      #http://localhost:8080/graphiql
      enabled: true
  datasource:
     url: jdbc:h2:mem:testdb;MODE=PostgreSQL
     driverClassName: org.h2.Driver
     username: sa
     password: password
```

We have defined the data source URL with MODE=PostgresSQL as we plan to use PostgreSQL as our actual DB for the actual environment. With this mode, H2 DB will provide or support most of PostgreSQL functionality.

Let's define the DB table schema for our use case. We need 3 tables to support our use case. By default spring data jpa will look for the **resources** folder and if any file is named with **schema.sql** it will use that script to create tables.

```sql
CREATE TABLE IF NOT EXISTS  tb_user  (
    id VARCHAR(255) DEFAULT random_uuid() PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    date_of_birth DATE,
    gender VARCHAR(1) NOT NULL,
    created_date TIMESTAMP,
    modified_date TIMESTAMP
);

CREATE TABLE IF NOT EXISTS  address  (
    id VARCHAR(255) DEFAULT random_uuid() PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    street1 VARCHAR(255) NOT NULL,
    street2 VARCHAR(255) NOT NULL,
    city VARCHAR(255) NOT NULL,
    state VARCHAR(255) NOT NULL,
    zip INTEGER NOT NULL,
    created_date TIMESTAMP,
    modified_date TIMESTAMP,
    FOREIGN KEY (id) REFERENCES tb_user(id)
);

CREATE TABLE IF NOT EXISTS  phone  (
    id VARCHAR(255) DEFAULT random_uuid() PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    number VARCHAR(10) NOT NULL,
    country_code VARCHAR(1) NOT NULL,
    created_date TIMESTAMP,
    modified_date TIMESTAMP,
    FOREIGN KEY (id) REFERENCES tb_user(id)
);
```

* We need to have 3 tables named **tb\_user, address, phone** to store user details.
    
* **tb\_user -&gt;** Will have basic user details with auto-generated UUID as Primary Key.
    
* **address -&gt;** This table will store the user's address details and it has a Foreign Key relationship with **tb\_user.**
    
* **phone -&gt;** This table will store the user's phone details and it also has a Foreign Key relationship with **tb\_user.**
    
* All 3 table primary key ID fields will be auto-generated using the **UUID** function of the database engine.
    

> On application startup, spring will refer to this **schema.sql** file execute this script on the configured DB and create defined tables. This type of DDL operation should be used only in local development.

## The Code

So far we have defined the template of the use case now it's time to implement the actual code.

First, let's define our entity classes which is Java object representation of our DB tables.

As we have 3 tables we need to define 3 entities to represent those three tables.

```java
package com.learntech.graphqlserverapi.entity;


import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.Type;
import org.hibernate.annotations.UpdateTimestamp;

import javax.persistence.*;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "tb_user")
@Getter
@Setter
public class UserEntity {

    @Id
    @GeneratedValue(generator = "UUID") //Let the underlying DB take care of UUID creation.
    @Type(type = "uuid-char")
    private UUID id;

    @Column(name = "first_name", nullable = false)
    private String firstName;

    @Column(name = "last_name", nullable = false)
    private String lastName;

    @Column(name = "date_of_birth", nullable = false)
    private LocalDate dateOfBirth;

    @Column(name = "gender", nullable = false)
    private String gender;

    @Column(name = "created_date", nullable = false)
    @CreationTimestamp
    private LocalDateTime created_date;

    @Column(name = "modified_date", nullable = false)
    @UpdateTimestamp
    private LocalDateTime modified_date;


    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<AddressEntity> addresses;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<PhoneEntity> phones;
}
```

* **UserEntity** is a Java representation of the tb\_user table.
    
* This entity is defined by a id property which is the primary key of our table.
    
    @Id will indicate that filed is primary key mapping field.
    
* **@GeneratedValue(generator = "UUID")** will indicate key generation strategy.
    
    For our use case, we are going to use the underlying DB engine to create this field value and its type UUID. Upon each row insert DB engine will execute a UUID function and it will create a unique UUID value for each row.
    
* **@Type(type = "uuid-char")** annotation will indicate this UUID field should be stored as a String value in DB. By default, Hibernate will store UUID filed value as Binary in DB.
    
* Another important thing to be noted here as per our use case a user can have multiple address and phones associated with them. To define this relationship we need to use @OneToMany annotation on the **UserEntity.**
    
* **@OneToMany(mappedBy = "user", cascade = CascadeType.*ALL*, orphanRemoval = true)** will define UserEntity and will have one-to-many relationships with **AddressEntity**. **mappedBy** will indicate a field with the name **user** on many sides (**AddressEntity**) will own the relationship
    
* **cascade = CascadeType.*ALL*** will indicate all the operations(persist, merge, remove etc) on this entity should be cascaded to the other side as well.
    
* **orphanRemoval = true** will indicate when any entity removed from the collection in the many side respective database entry should be deleted from the many sides of the table.
    
* Like the above **address** mapping **phone** entity will also defined with similar mapping.
    

Next, let's define our **address** table entity definition as like below.

```java
@Entity
@Table(name = "address")
@Getter
@Setter
public class AddressEntity {

    @Id
    @GeneratedValue(generator = "UUID")
    @Type(type = "uuid-char")
    private UUID id;

    @Column(name = "type", nullable = false)
    private String type;

    @Column(name = "street1", nullable = false)
    private String street1;

    @Column(name = "street2", nullable = true)
    private String street2;

    @Column(name = "city", nullable = false)
    private String city;

    @Column(name = "state", nullable = false)
    private String state;

    @Column(name = "zip", nullable = false)
    private Integer zip;

    @Column(name = "created_date", nullable = false)
    @CreationTimestamp
    private LocalDateTime created_date;

    @Column(name = "modified_date", nullable = false)
    @UpdateTimestamp
    private LocalDateTime modified_date;

    @ManyToOne //Defines a Bi-Directional mapping/relationship between User & Address entities.
    @JoinColumn(name = "user_id", referencedColumnName = "id", nullable = false)
    private UserEntity user;
}
```

* As discussed earlier this entity will also have **@Id** field to indicate its own Primary Key for every row and its type of UUID.
    
* Also, it has a field called **user** with **@ManyToOne** to indicate **AddressEntity** has Bi-Directional mapping/relationship with another entity which is **UserEntity** in this case. Also, this annotation will indicate this is the owning side of the relationship. Also, **@JoinColumn** will defined with a column name **user\_id** which is referenced with **id** column in **UserEntity** to indicate the Foreign Key relationship.
    

Like the **AddressEntity** we have to define a **PhoneEntity** to represent phone table.

```java
@Entity
@Table(name = "phone")
@Getter
@Setter
public class PhoneEntity {

    @Id
    @GeneratedValue(generator = "UUID")
    @Type(type = "uuid-char")
    private UUID id;

    @Column(name = "type", nullable = false)
    private String type;

    @Column(name = "number", nullable = false)
    private String number;

    @Column(name = "country_code", nullable = false)
    private String countryCode;

    @Column(name = "created_date", nullable = false)
    @CreationTimestamp
    private LocalDateTime created_date;

    @Column(name = "modified_date", nullable = false)
    @UpdateTimestamp
    private LocalDateTime modified_date;

    @ManyToOne
    @JoinColumn(name = "user_id", referencedColumnName = "id", nullable = false)
    private UserEntity user;
}
```

Let's define our Repository to interact with our DB tables and get the data. As per our use case, we need to support the following use cases.

* To get a List of Users for given search criteria for this use case let's have firstName and lastName-based search as input.
    
* To get a specific User for a given ID.
    

To support this we will declare the following methods in our Repository class.

```java
@Repository
public interface UserRepository extends JpaRepository<UserEntity, UUID> {

    @Query("SELECT distinct u FROM UserEntity u JOIN FETCH u.addresses a  WHERE  u.firstName = :firstName AND u.lastName = :lastName")
    List<UserEntity> findUserAddressByFirstNameAndLastName(@Param("firstName") String firstName, @Param("lastName") String lastName);

    @Query("SELECT distinct u FROM UserEntity u JOIN FETCH u.phones p  WHERE  u.firstName = :firstName AND u.lastName = :lastName")
    List<UserEntity> findUserPhoneByFirstNameAndLastName(@Param("firstName") String firstName, @Param("lastName") String lastName);

    UserEntity findUserEntityById(UUID id);

}
```

Our **UserRepository extends JpaRepository&lt;UserEntity, UUID&gt;** which provides some JPA DB operations out of the box and we can define our custom implementation.

* **findUserAddressByFirstNameAndLastName** will fetch the given user Address information from **address** table for the given **firstName** and **lastName** along with user information.
    
* **findUserPhoneByFirstNameAndLastName** will fetch the given user's Phone information from **phone** table for the given **firstName** and **lastName** along with user information.
    
* **findUserEntityById** will fetch User information from the user table for a given UUID.
    

Once the repository is defined we have to register/enable it so Spring will load this class into the application context during start-up. We can do this by using **@EnableJpaRepositories**(basePackages = {"com.learntech.graphqlserverapi.repository"}) in our main application.

```java
@SpringBootApplication
@EnableJpaRepositories(basePackages = {"com.learntech.graphqlserverapi.repository"})
public class GraphqlServerApiApplication {

	public static void main(String[] args) {
		SpringApplication.run(GraphqlServerApiApplication.class, args);
	}

}
```

As we are using In-Memory DB and our current use case is only fetch operation we need some data to be populated in the table. There are multiple ways we can achieve this. In our case, we will create a DataLoader class which will load/insert data into tables upon startup. For this, we will use our **UserRepository.**

Let's define a **DataLoader** class to load some data for our testing.

```java
@Component
@Slf4j
public class DataLoader implements CommandLineRunner {

    private final UserRepository userRepository;

    public DataLoader(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    /**
     * @param args incoming main method arguments
     * @throws Exception
     */
    @Override
    public void run(String... args) throws Exception {

        UserEntity userEntity = new UserEntity();

        //userEntity.setId(UUID.randomUUID());
        userEntity.setFirstName("Jhon");
        userEntity.setLastName("Victor");
        userEntity.setDateOfBirth(LocalDate.now());
        userEntity.setGender("M");
        userEntity.setCreated_date(LocalDateTime.now());
        userEntity.setModified_date(LocalDateTime.now());

        AddressEntity homeAddress = new AddressEntity();

        //homeAddress.setId(UUID.randomUUID());
        homeAddress.setType("Home");
        homeAddress.setStreet1("1234 First Street");
        homeAddress.setStreet2("Apt 10");
        homeAddress.setCity("Louesville");
        homeAddress.setState("TX");
        homeAddress.setZip(12345);

        homeAddress.setUser(userEntity);
        AddressEntity officeAddress = new AddressEntity();

        officeAddress.setType("Office");
        officeAddress.setStreet1("1234 First Street");
        officeAddress.setStreet2("Suite 3");
        officeAddress.setCity("Louesville");
        officeAddress.setState("TX");
        officeAddress.setZip(12345);
        //To associate Parent UserEntity to Child to create Foreign Key relationship
        officeAddress.setUser(userEntity);
        userEntity.setAddresses(Arrays.asList(homeAddress,officeAddress));

        PhoneEntity homePhone = new PhoneEntity();

        //homePhone.setId(UUID.randomUUID());
        homePhone.setType("Home");
        homePhone.setNumber("1234567890");
        homePhone.setCountryCode("+1");
        homePhone.setCreated_date(LocalDateTime.now());
        homePhone.setModified_date(LocalDateTime.now());

        homePhone.setUser(userEntity);
        PhoneEntity officePhone = new PhoneEntity();

        officePhone.setType("Office");
        officePhone.setNumber("9876543210");
        officePhone.setCountryCode("+1");
        officePhone.setCreated_date(LocalDateTime.now());
        officePhone.setModified_date(LocalDateTime.now());

        officePhone.setUser(userEntity);
        userEntity.setPhones(Arrays.asList(homePhone,officePhone));

        userRepository.deleteAll();
        //Load data into tables during application startup
        userRepository.save(userEntity);

        List<UserEntity> users =userRepository.findUserAddressByFirstNameAndLastName("Jhon","Victor");

        log.info("Fetched user entities {}", users.get(0).getId());

     }
}
```

* **DataLoader** implements **CommandLineRunner** and Override run method spring boot will detect this during start-up and execute this method.
    
* We are just constructing **UserEntity** and setting all values and respective Address and Phone information. However, we are not setting ID value of this entity as it will be created by DB engine as per our entity configuration.
    
* Upon successful application start this class will save this record in H2 DB.
    

Now let's define our service layer with the implementation of our use case to connect the repository to fetch the data.

```java
@Service
@Slf4j
public class UserSearchServiceImpl implements UserSearchService {

    private final UserRepository userRepository;

    public UserSearchServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    @Override
    public User searchById(UUID id) {
        UserEntity userEntity = userRepository.findUserEntityById(id);
        User user = new User();
        BeanUtils.copyProperties(userEntity,user);

        user.setAddress(buildAddress(userEntity.getAddresses()));
        user.setPhone(buildPhone(userEntity.getPhones()));
        return user;
    }
    
    private List<Address> buildAddress(List<AddressEntity> addressEntities){
        List<Address> addresses = new ArrayList<>();
        addressEntities.forEach(addressEntity -> {
            Address address = new Address();
            BeanUtils.copyProperties(addressEntity, address);
            addresses.add(address);
        });
        return addresses;
    }

    private List<Phone> buildPhone(List<PhoneEntity> phoneEntities){
        List<Phone> phones = new ArrayList<>();
        phoneEntities.forEach(phoneEntity -> {
            Phone phone = new Phone();
            BeanUtils.copyProperties(phoneEntity, phone);
            phones.add(phone);
        });
        return phones;
    }
}
```

* **searchById** will receive a UUID as input and call the **userRepository** by passing this UUID as input and fetching the respective UUID User data.
    
* **buildAddress** and **buildPhone** methods are just a kind of helper methods to map the address and phone details to the final response.
    

> One thing to be noted here is we are executing **searchById** *method in @Transactional boundary. This is to avoid hibernating session close exceptions while fetching address and phone details.*

Next, let's implement our other use case fetch by **firstName** and **lastName**.

```java
 @Override
    public List<User> searchUser(SearchInput searchInput) {
        log.info("searchUser Starts");
        List<UserEntity> userAddressEntities = userRepository.findUserAddressByFirstNameAndLastName(searchInput.getFirstName(), searchInput.getLastName());
        List<UserEntity> userPhoneEntities = userRepository.findUserPhoneByFirstNameAndLastName(searchInput.getFirstName(), searchInput.getLastName());

        List<User> users = new ArrayList<>();

        if(!CollectionUtils.isEmpty(userAddressEntities)){

            users =  userAddressEntities.stream()
                            .map(userEntity -> {
                                User user = buildUser(userEntity);
                                user.setAddress(buildAddress(userEntity.getAddresses()));
                                UserEntity userPhoneEntity = getUserById(userPhoneEntities, userEntity.getId());
                                user.setPhone(buildPhone(userPhoneEntity.getPhones()));
                                return user;
                            }).collect(Collectors.toList());
        }
        return users;
    }

    private UserEntity getUserById(List<UserEntity> userEntities, UUID userId){
       return userEntities.stream()
                    .filter(userEntity -> userEntity.getId().equals(userId))
                .findFirst().get();
    }
```

* We are first fetching both address and phone details for the given **firstName** and **lastName** from that result filter our address and phone details along with user data.
    

> Here we are fetching both address and phone details using separate query execution this is due to avoid hibernate error and "**cannot simultaneously fetch multiple bags"** as our user entity has more than OneToMany mapping.

Finally, let's define data fetcher class which calls this service class.

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
        log.info("users() Starts");
        log.info("First Name {}", searchInput.getFirstName());
        log.info("Last Name {}", searchInput.getLastName());

        List<User> users = userSearchService.searchUser(searchInput);
        return Mono.just(users);
    }

    @DgsQuery
    public Mono<User> user(@InputArgument UUID id){
        log.info("user() Starts");
        User user = userSearchService.searchById(id);
        return Mono.just(user);
    }
}
```

### **Test the Application**

Now it's time to test our application. To test that our data is loaded properly during the start-up we need to introduce a special component. As we are using Spring WebFlux it won't support H2 DB console like Spring MVC.

So we have to configure a webserver and tcpServer to expose the H2 DB console in the browser.

```java
@Component
public class H2DBConfiguration {

    private org.h2.tools.Server webServer;

    private org.h2.tools.Server tcpServer;

    @EventListener(org.springframework.context.event.ContextRefreshedEvent.class)
    public void start() throws java.sql.SQLException {
        this.webServer = org.h2.tools.Server.createWebServer("-webPort", "8082", "-tcpAllowOthers").start();
        this.tcpServer = org.h2.tools.Server.createTcpServer("-tcpPort", "9092", "-tcpAllowOthers").start();
    }

    @EventListener(org.springframework.context.event.ContextClosedEvent.class)
    public void stop() {
        this.tcpServer.stop();
        this.webServer.stop();
    }
}
```

This will allow us to connect to H2 DB from the browser and view tables and data.

When we start the application we can see our **DataLoader** got invoked and load the data in DB.

As we defined our DB configuration using the spring profile we have to provide that information when starting the app.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702011247809/7211b5ad-3602-4abb-aca2-b6a5ee16d833.png align="center")

After a successful application start-up, we can see a record get inserted and the respective UUID will be printed.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702011445784/79169fd3-77a7-4242-9e4b-9f27cc7a8b75.png align="center")

Let's confirm the tables in H2 console as well. Browse with below URL.

*http://localhost:8082*

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702011523936/7ff18248-9f75-4ad3-9b60-f8a2883140df.png align="center")

When clicking connect and execute the query we can see the table data.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702011678535/24777259-e996-4f9b-bf08-8d3ad4809d4a.png align="center")

This will indicate our table is populated and now we can use this data to test our 2 use cases.

We can test our endpoint queries in any client.

This will conclude our article. Happy learning!!!