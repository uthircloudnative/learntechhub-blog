---
title: "How to containarize a spring boot application with Jib plugin"
datePublished: Thu Dec 28 2023 19:12:26 GMT+0000 (Coordinated Universal Time)
cuid: clqpl0pup000208l0almo2rzn
slug: how-to-containarize-a-spring-boot-application-with-jib-plugin
tags: devops, docker-images, jib

---

## Introduction

This article will discuss on how to containerize spring boot application using Google Jib plugin. To containerize an application we have multiple options available. In this article we will explore Jib plugin.

Jib is a plugin which is developed by Google will help to convert a given source code to docker images. This will support different leading languages. We can find more information on this in below link.

[Jib plugin for Java](https://github.com/GoogleContainerTools/jib/blob/master/jib-gradle-plugin/README.md)

## Tech Stack

Java 17

Gradle

Spring boot

Jib Gradle plugin

## Implementation

Let's create a very basic simple spring boot application and expose a single REST endpoint.

First let's choose gradle is our build tool and include following dependencies.

```java
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.2.0'
	id 'io.spring.dependency-management' version '1.1.4'
	id 'com.google.cloud.tools.jib' version '3.4.0'
}

group = 'com.learntech'
version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '17'
}

jib {
	container {
		mainClass = 'com.learntech.k8slearning.K8sLearningApplication'
		ports = ['8080']
	}
	to {
		image = 'registry-1.docker.io/<RegistryUserName>/k8s-learning:latest'
		auth {
			username = 'RegistryUSERNAME'
			password = 'RegistryPASSWORD'
		}
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}
```

* In the above configuration under plugin section we defined 4 plugins in which our focus would be the last **com.google.cloud.tools.jib.** This plugin is used to create docker image while build the application.
    
* In **Jib** section we need to provide the informations to create images which is then used by the plugin while building the application. In **To** section we need to provide a registry details. This is where we need to save the image once its build.
    
* In **Container** section provide application main class so the image will use point to this class. Also provide a port-8080 in which this container to be accessed.
    
* Here we have provided DockerHub as registry URL and in auth section we need to provide our DockerHub userName and password as it required to authenticate and push the image once its build successfully.
    

> This type of defining UserName and Password information in source code is not advisable for Production it's only for learning purpose. For actual production usage there are different options available.

## The Code

Now let's take look at the actual application code. Just define a RestController and expose a GET endpoint. This is just to test the application.

```java
package com.learntech.k8slearning.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * ServerController
 *
 * @author Uthiraraj Saminathan
 */
@RestController
public class ServerController {

    @GetMapping("/servercheck")
    public String getNodeData(){
       return "Reached Server Controller";
    }
}
```

With all this configuration run below command which will create a docker image and push the image to configured docker hub registry.

> ./gradlew jib