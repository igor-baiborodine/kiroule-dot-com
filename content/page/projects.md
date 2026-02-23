---
title: "Projects"
date: 2020-04-25T21:25:07-04:00
menu: "topnav"
showDate: false
showComments: false
weight: 20
---

{{< toc >}}

### 2026

##### Insurance Hub - Migration to Go

Technologies used: [Go](https://github.com/golang/go), [gRPC](https://github.com/grpc/grpc-go), [PostgreSQL](https://www.postgresql.org/), [Goose](https://pressly.github.io/goose/), [Docker](https://www.docker.com/), [Kubernetes](https://kubernetes.io/)\
[See GitHub repository](https://github.com/igor-baiborodine/insurance-hub)

### 2025

##### Insurance Hub - Migration to Go

Technologies used: [Go](https://github.com/golang/go), [gRPC](https://github.com/grpc/grpc-go), [PostgreSQL](https://www.postgresql.org/), [Goose](https://pressly.github.io/goose/), [Docker](https://www.docker.com/), [Kubernetes](https://kubernetes.io/)\
[See GitHub repository](https://github.com/igor-baiborodine/insurance-hub)

### 2024

##### Campsite Booking API (Go)

Technologies used: [Go](https://github.com/golang/go), [gRPC](https://github.com/grpc/grpc-go), [PostgreSQL](https://www.postgresql.org/), [Goose](https://pressly.github.io/goose/), [Docker](https://www.docker.com/), [Kubernetes](https://kubernetes.io/)\
[See GitHub repository](https://github.com/igor-baiborodine/campsite-booking-go)

##### Mastering Go

1. [Event-Driven Architecture in Golang](https://github.com/igor-baiborodine/event-driven-architecture-in-golang-workshop)

### 2023

##### Campsite Booking API (Java, 4th iteration)

1. Upgraded to Spring Boot 3. 
2. Added entity classes for the DB layer.
3. Reworked unit and integration tests using explicit `given-when-then` pattern.
4. Introduced test containers for integration tests.
5. Implemented database versioning using Flyway migrations.
6. Migrated to multi-module Maven project structure.
7. Applied the API-first design approach.

Read this [article](https://www.kiroule.com/article/campsite-booking-api-revisited-3/) for more details.

Technologies used: [Java 17](https://www.oracle.com/java/technologies/downloads/#java17), [Spring Boot 3](http://projects.spring.io/spring-boot/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/campsite-booking)

##### Bilberry Hugo Theme

1. Defined the [scope](https://github.com/Lednerb/bilberry-hugo-theme/projects/2) of the `v4`, a new major release, and implemented it. 

Technologies used: [Hugo](https://gohugo.io/)\
[See GitHub Repository](https://github.com/Lednerb/bilberry-hugo-theme)

##### Mastering Go

1. [Distributed Services with Go Workshop](https://github.com/igor-baiborodine/distributed-services-with-go-workshop)
2. [Elements of Programming Interviews in Go](https://github.com/igor-baiborodine/epi-go)

### 2022

##### Bilberry Hugo Theme

Made the following [contributions](https://github.com/Lednerb/bilberry-hugo-theme/commits?author=igor-baiborodine):
1. Added support for Hugo modules.
2. Implemented various new features, fixed bugs, reviewed pull requests, and updated documentation.

Technologies used: [Hugo](https://gohugo.io/)\
[See GitHub Repository](https://github.com/Lednerb/bilberry-hugo-theme)

##### Campsite Booking API (Java, 3rd iteration)

1. Re-wrote unit and integration tests with JUnit 5 in BDD style.
2. Introduced `var` syntax for local variables.
3. Upgraded to Java **17** LTS.
4. Implemented `Campsite` domain class/table and corresponding service and repository classes; upgraded API's contract
   to `v2`.
5. Added the `findForDateRange` method without pessimistic read locking.
6. Re-implemented the `findForDateRangeWithPessimisticWriteLocking` method, added integration tests for concurrent
   bookings create/update.
7. Added TOC Generator GitHub Actions workflow.

Read this [article](https://www.kiroule.com/article/campsite-booking-api-revisited-2/) for more details.

Technologies used: [Java 17](https://www.oracle.com/java/technologies/downloads/#java17), [Spring Boot 2](http://projects.spring.io/spring-boot/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/campsite-booking)

##### Mastering Go

[Exercises](https://github.com/igor-baiborodine/coding-challenges/blob/master/exercism/go-exercises.md) related to the [Exercism's Go track syllabus](https://exercism.org/tracks/go/concepts).

##### Company Website

Developed company website at https://www.projetsrios.com/.

Technologies used: [Hugo Terrassa Theme](https://github.com/igor-baiborodine/hugo-terrassa-theme), [Git](https://git-scm.com/), [Hugo](https://gohugo.io/), [GitHub](https://github.com/), [Netlify](https://www.netlify.com/)\
[See GitHub Repository](https://github.com/projetsrios/projetsrios.com)

##### CI/CD with GitHub Actions

Automated the following software development workflows:
1. [Workflow](https://github.com/igor-baiborodine/bilberry-hugo-theme-sandbox/blob/master/.github/workflows/upload-data-to-algolia-index.yml) for uploading data to Algolia index for the [Bilberry Sandbox/Netlify](https://www.bilberry-sandbox.kiroule.com/).
2. [Workflow](https://github.com/igor-baiborodine/bilberry-hugo-theme-sandbox/blob/master/.github/workflows/gh-pages.yml) for updating GitHub Pages when using the Bilberry theme as a Hugo module for the [Bilberry Sandbox/GitHub Pages](https://igor-baiborodine.github.io/bilberry-hugo-theme-sandbox/).

Technologies used: [GitHub Actions](https://github.com/features/actions)

##### Bilberry Hugo Theme Test Website

Created the [Bilberry Sandbox](https://www.bilberry-sandbox.kiroule.com/), which helps me develop, test, and maintain
the Bilberry theme.

Read this [article](https://www.kiroule.com/article/simplify-development-and-testing-with-bilberry-sandbox/) for more details.

Technologies used: [Bilberry Hugo Theme](https://github.com/Lednerb/bilberry-hugo-theme), [Git](https://git-scm.com/), [Hugo](https://gohugo.io/), [GitHub](https://github.com/), [Netlify](https://www.netlify.com/)\
[See GitHub Repository](https://github.com/igor-baiborodine/bilberry-hugo-theme-sandbox)

### 2021

##### Bilberry Hugo Theme
Took the role of the official maintainer; made the following [contributions](https://github.com/Lednerb/bilberry-hugo-theme/commits?author=igor-baiborodine):
1. Automated data upload to Algolia index using JavaScript API client.
2. Implemented support for custom audio files; co-authored the archive functionality; updated documentation.
3. Reviewed pull requests.

Technologies used: [Hugo](https://gohugo.io/)\
[See GitHub Repository](https://github.com/Lednerb/bilberry-hugo-theme)

##### Campsite Booking API (Java, 2nd iteration)

Read this [article](https://www.kiroule.com/article/campsite-booking-api-revisited/) for more details.

Technologies used: [Java 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html), [Spring Boot 2](http://projects.spring.io/spring-boot/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/campsite-booking)

##### CI/CD with GitHub Actions
Automated software development workflows for the following personal repositories:
1. [Docker Liferay Portal CE](https://github.com/igor-baiborodine/docker-liferay-portal-ce/tree/master/.github/workflows)
2. [Campsite Booking API](https://github.com/igor-baiborodine/campsite-booking/tree/master/.github/workflows)
3. [Vaadin Demo Bakery App](https://github.com/igor-baiborodine/vaadin-demo-bakery-app/tree/master/.github/workflows)
4. [Vaadin Demo Business App](https://github.com/igor-baiborodine/vaadin-demo-business-app/tree/master/.github/workflows)

Read this [article](https://www.kiroule.com/article/github-actions-in-action/) for more details.

Technologies used: [GitHub Actions](https://github.com/features/actions)

### 2020

##### Bilberry Hugo Theme

Made the following [contributions](https://github.com/Lednerb/bilberry-hugo-theme/commits?author=igor-baiborodine):
1. Implemented the series taxonomy and table of contents features.
2. Automated data upload to Algolia index using Python API client.
3. Fixed bugs.
4. Updated documentation. 

Technologies used: [Hugo](https://gohugo.io/)\
[See GitHub Repository](https://github.com/Lednerb/bilberry-hugo-theme)

##### Personal Website
Developed personal website at https://kiroule.com

Read this [series](https://www.kiroule.com/series/building-your-blog-the-geeky-way/) for more details.

Technologies used: [Bilberry Hugo Theme](https://github.com/Lednerb/bilberry-hugo-theme), [Git](https://git-scm.com/), [Hugo](https://gohugo.io/), [GitHub](https://github.com/), [Netlify](https://www.netlify.com/)\
[See GitHub Repository](https://github.com/igor-baiborodine/kiroule.com)

### 2019

##### Multi-Variant Docker Images for Liferay Portal CE
*Updated in 2020, 2021*

Read this [article](https://www.kiroule.com/article/multi-variant-docker-images-for-liferay-portal-ce/) for more details.

**500K+ pulls**\
Technologies used: [Docker](https://www.docker.com/), [Debian](https://www.debian.org/), [Alpine Linux](https://alpinelinux.org/)\
[See Docker Hub Repository](https://hub.docker.com/r/ibaiborodine/liferay-portal-ce)\
[See GitHub Repository](https://github.com/igor-baiborodine/docker-liferay-portal-ce)

### 2018

##### Campsite Booking API (Java)

Technologies used: [Java 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html), [Spring Boot 2](http://projects.spring.io/spring-boot/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/campsite-booking)

##### Coding Challenges from CodeWars, HackerRank, Codility

Technologies used: Java, Python\
[See GitHub repository](https://github.com/igor-baiborodine/coding-challenges)

### 2016

##### MyBatis JPetStore (MongoDB)
Exercise to port [MyBatis JPetStore](https://github.com/mybatis/jpetstore-6) from RDBMS/MyBatis to NoSQL/MongoDB and Spring Boot

Technologies used: [Java 8](https://www.oracle.com/java/technologies/javase-jdk8-downloads.html), [Lombok](https://projectlombok.org/), [Stripes](https://stripesframework.atlassian.net/wiki/display/STRIPES/Home), [Spring Boot](http://projects.spring.io/spring-boot/), [Spring Data MongoDB](http://projects.spring.io/spring-data-mongodb/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/jpetstore-6-spring-data-mongodb)

### 2015

##### MyBatis JPetStore (Vaadin)
Exercise to port [MyBatis JPetStore](https://github.com/mybatis/jpetstore-6) to Vaadin 7|8 and Spring Boot

Technologies used: [Java 11](https://www.oracle.com/technetwork/java/javase/downloads/jdk11-downloads-5066655.html), [Vaadin 8](https://vaadin.com/home), [Spring Boot 2](http://projects.spring.io/spring-boot/), [MyBatis 3](http://mybatis.org/mybatis-3/), [Maven 3](http://maven.apache.org/)\
[See GitHub repository](https://github.com/igor-baiborodine/jpetstore-6-vaadin-spring-boot)

This project was mentioned in [2016 January's edition of Vaadin's Community Spotlight](https://vaadin.com/blog/community-spotlight-january-2016):

![Vaadin Community Spotlight](/img/content/page/projects/vaadin-community-spotlight.png)
