# Dddit Server Report for Software Dependability Course

<p align="center"><img src='https://i.postimg.cc/QxnvK4LL/dddit-upscaled.png' alt="Quixel_Texel_Logo" height="200"></p>

## Authors

**Angelo Antonio Prisco** (0522501996) - [AngeloAntonioPrisco](https://github.com/AngeloAntonioPrisco)

**Pasquale Sorrentino** (0522501996) - [PasqualeSorrentino](https://github.com/PasqualeSorrentino)

## Professor 
**Prof. Dario Di Nucci** - [Dario Di Nucci](https://docenti.unisa.it/029186/home)

## Index

- [Introduction](#introduction)
    - [Document Goal](#document-goal)
    - [Project Description](#project-description)
- [Containerization with Docker](#containerization-with-docker)
- [Instruments and Techniques](#instruments-and-techniques)
    - [Security and Code Quality Analysis](#security-and-code-quality-analysis)
        - [SonarQube Cloud](#sonarqube-cloud)
        - [Snyk](#snyk)
        - [Git Guardian](#git-guardian)
    - [Testing and Code Coverage](#testing-and-code-coverage)
        - [Unit Tests with JUnit](#unit-tests-with-junit)
        - [Code Coverage with JaCoCo](#code-coverage-with-jacoco)
        - [Mutation Testing with pitest](#mutation-testing-with-pitest)
        - [Benchmarks with JMH](#benchmarks-with-jmh)
    - [CI/CD with GitHub Actions](#cicd-with-github-actions)
        - [Build, Test and Coverage](#build-test-and-coverage)
        - [Security Analysis](#security-analysis)
        - [Build and Deployment](#build-and-deployment)
    - [JML](#jml)
- [Conclusions](##conclusions)

## Introduction
### Document Goal
The purpose of this report is to describe the work carried out within the context of the Software Dependability course project. The report highlights how the techniques, methodologies, and tools presented during the course were applied and integrated into the development lifecycle of a web application.

In particular, the document illustrates the approaches adopted to improve the reliability, correctness, and maintainability of the system. It also discusses the processes used to analyze, test, and verify the software, demonstrating how dependability-oriented practices can be incorporated into a modern software development workflow.

Through this report, the project aims to provide a clear overview of the development process, the adopted technologies, and the strategies used to ensure that the application meets the expected dependability requirements.

### Project Description

The project focuses on the development of a system designed to manage and version 3D assets, specifically 3D models and their associated materials. In many 3D design and graphics workflows, models and materials evolve over time as they are modified, refined, or reused across different projects. The system aims to support this process by providing mechanisms to track changes, manage different versions of assets, and allow users to organize and retrieve previous states of 3D models and materials in a structured and reliable way.

To achieve this goal, the project implements a server-side web application responsible for handling asset storage, version management, and user interactions through a set of APIs. The system allows users to upload, update, and manage versions of 3D models and related materials, ensuring that modifications can be tracked and previous versions can be accessed when needed. This approach facilitates collaboration, reproducibility, and better management of complex 3D content.

The purpose of this report is to describe the work carried out within the context of the Software Dependability course project. In particular, the report highlights how the techniques, methodologies, and tools presented during the course were applied and integrated into the development lifecycle of the application. The document also discusses the strategies adopted to improve the reliability, correctness, and maintainability of the system, demonstrating how dependability-oriented practices can be incorporated into the development of modern software systems.

## Containerization with Docker
The system is designed as a containerized backend application composed
of multiple services that cooperate to provide versioning
functionalities for 3D assets. The infrastructure is orchestrated using
Docker and Docker Compose, allowing the application and all its
dependencies to run in a reproducible and isolated environment.

The core component of the system is a Java server application built
using the Spring Framework. The application exposes RESTful APIs
responsible for managing repositories, resources, versions, and
collaborative features such as repository invitations. The application
is built using Maven and packaged into a Docker image through a
multi-stage build process.

An excerpt of the Dockerfile used to build the application is shown
below:
```dockerfile
FROM maven:3.9.6-eclipse-temurin-21 AS build
WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn clean package
```

The final runtime image uses a lightweight Java Runtime Environment:
```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*-exec.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The application relies on three main external services: MongoDB,
JanusGraph, and MinIO. These services are orchestrated through Docker
Compose, which defines the containers, networking configuration, and
persistent storage volumes.

MongoDB is used as a document-oriented database to store version
metadata and authentication-related information. The following excerpt
from the Docker Compose configuration shows how the MongoDB container is
defined:
```dockerfile
mongodb:
  image: mongo:6
  container_name: mongodb
  ports:
    - "27017:27017"
  environment:
    MONGO_INITDB_ROOT_USERNAME: ${MONGODB_ROOT_USERNAME}
    MONGO_INITDB_ROOT_PASSWORD: ${MONGODB_ROOT_PASSWORD}
    MONGO_INITDB_DATABASE: ${MONGODB_NAME}
  volumes:
    - mongo-data:/data/db
```

JanusGraph is used as a graph database to model relationships between
entities such as users, repositories, and resources. This allows the
system to efficiently represent collaboration structures and repository
interactions.
```dockerfile
janusgraph:
  image: janusgraph/janusgraph:1.0
  container_name: janusgraph
  command:
    - bin/janusgraph-server.sh
    - /opt/janusgraph/conf/janusgraph-server.yaml
  ports:
    - "8182:8182"
  volumes:
    - janus-data:/var/lib/janusgraph
```

For storing large binary files such as FBX models and PNG textures
associated with materials, the system uses MinIO, an S3-compatible
object storage service:
```dockerfile
minio:
  image: minio/minio
  container_name: minio
  command: server /data --console-address ":9001"
  ports:
    - "9000:9000"
    - "9001:9001"
  volumes:
    - minio-data:/data
```

The application container connects to these services using environment
variables defined in the Docker Compose configuration. This approach
allows the same container image to be deployed in different environments
by simply changing the configuration parameters.
```dockerfile
environment:
  MINIO_HOST: minio
  MONGO_HOST: mongodb
  JANUS_HOST: janusgraph
  JWT_SECRET: ${JWT_SECRET}
```

Persistent volumes are defined for all storage services to ensure that
data remains available even if containers are restarted or recreated.
This architecture separates metadata management from binary asset
storage: structured data and relationships are stored in MongoDB and
JanusGraph, while large files are handled by MinIO.

In addition to the local development configuration, the project also
includes a separate Docker Compose configuration used for automated
environments such as CI pipelines. In this configuration, the
application container is pulled directly from Docker Hub rather than
built locally, allowing automated systems to deploy and test the
application in a consistent environment.

## Instruments and Techniques

### Security and Code Quality Analysis
#### SonarQube Cloud
The SonarQube Cloud static analysis session processed the project's source code, revealing a total of 187 issues (~40 Blocker, ~107 High, ~38 Medium, ~2 Low/Info) across 65 files. The distribution of these findings suggests that while the codebase is functional, it carries significant technical debt related to maintainability, architectural consistency, and robust error handling.

*Architectural Integrity & Dependency Injection* -
The most frequent architectural violation is the use of Field Injection (via `@Autowired` or similar annotations). This pattern is prevalent in almost all Service and Repository implementations (e.g., `BranchServiceImpl`, `ResourceServiceImpl`, `VersionServiceImpl`).

*Code Maintainability & String Constants* -
The analysis detected high levels of literal duplication. Strings such as "error", "details", "message", "repositoryName", and "username" are repeated up to 9 times in single files like `AuthControllerImpl` and `GremlinRepositoryImpl`.

*API Design & Type Safety* -
In the Controller layer (e.g., `AuthControllerImpl`, `BranchControllerImpl`, `ResourceControllerImpl`), SonarQube flagged the usage of generic wildcard types (`?`) in return signatures. This obscures the API contract and reduces type safety for the consumers of the REST services.

*Exception Handling & Reliability* -
Two critical issues were identified regarding how the application handles failures:
- **InterruptedExceptions:** In the Gremlin repository suite, several methods catch `InterruptedException` without re-interrupting the thread or re-throwing it. This can lead to "swallowed" interrupts, preventing the JVM from shutting down threads correctly.
- **Exception Hierarchy:** Multiple custom exception classes (e.g., `InvalidBranchNameException`, `RepositoryNotFoundException`) exceed the authorized inheritance depth of 5 levels, indicating an overly complex class hierarchy.
- **Generic Exceptions:** Usage of `java.lang.Exception` in method signatures (e.g., in `SecurityConfig`) was flagged as too broad.

*Code Quality & Performance Smells* -
Several issues were identified regarding the application code quality and performance smells:
- **Cognitive Complexity:** High complexity was noted in `VersionValidatorImpl`, where logic needs to be broken down into smaller, more readable methods.
- **RegEx Efficiency:** Several validators were advised to use more concise character class syntax (e.g., `\w` instead of `[a-zA-Z0-9_]`).
- **Resource Management:** A try-with-resources violation was found in `TagClassificationServiceImpl`, which could lead to potential resource leaks.
- **Unused Code:** Numerous files contain unused imports and commented-out code that clutter the codebase.

*Summary Table* -
Following is a summary table of the issues identified in the SonarQube analysis:

<div align="center">

| Category | Primary Files Affected                                     |
| --- |------------------------------------------------------------|
| Dependency Injection | All `*ServiceImpl` & `*RepositoryImpl`                     |
| Literal Duplication | `AuthControllerImpl` and `GremlinInvitationRepositoryImpl` |
| API Type Safety | All `*ControllerImpl` classes                              |
| Thread Safety | `Gremlin*RepositoryImpl`                                   |
| Inheritance Depth | All `*Exception` classes                                   |

</div>

After a thorough refactoring, the number of issues was significantly reduced to **36 issues** (~40 Blocker resolved, ~107 High resolved, ~2 Medium resolved, ~2 Low/Info ignored), consisting of **34 medium** and **2 low severity** issues. The remaining medium issues mostly concern the **inheritance depth of custom exceptions exceeding 5 levels**, while the two low issues are related to the **naming of two specific classes**.
<div align="center">

[![postqube-png.png](https://i.postimg.cc/HkPczVMF/postqube-png.png)](https://postimg.cc/750ZYPmN)
</div>

#### Snyk
This detailed report summarizes the security analysis performed by **Snyk**, focusing on both open-source dependencies and code-level vulnerabilities within the project.

*Open Source Vulnerabilities (SCA)* -
The analysis of the project's dependencies initially revealed **24 unique vulnerabilities** with the following severity breakdown:

- **1 Critical (C):** Authentication Bypass in `spring-security-crypto`
- **13 High (H):** Including Path Traversal, Uncontrolled Recursion, and HTTP Request Smuggling
- **7 Medium (M):** CRLF Injection and Authorization Bypass
- **3 Low (L):** Information Exposure

Following is a breakdown of the vulnerabilities by their primary source:
- **Critical - Authentication Bypass:** The `spring-security-crypto@5.7.0` library contains a primary weakness that could allow attackers to bypass authentication mechanisms.
- **Infrastructure Risks:** Vulnerabilities in `tomcat-embed-core` and `netty-codec-http2` (Data Amplification, Request Smuggling) pose risks to the underlying server stability and request integrity.
- **Logic Risks:** `commons-lang` and `commons-lang3` are susceptible to Uncontrolled Recursion, which can lead to Denial of Service (DoS) through stack exhaustion.
- **Path Traversal:** Multiple libraries (`spring-beans`, `tomcat-embed-core`) were flagged for Relative Path Traversal, potentially allowing unauthorized file access.

After initial remediation steps, the Open Source vulnerabilities were reduced to **2 total issues** (1 High: `commons-lang` recursion; 1 Medium: `commons-configuration` resource allocation).

*Code Security (SAST)* -
The Static Application Security Testing (SAST) identified **26 vulnerabilities** within the custom codebase, all classified as **Low Severity**.

The majority of these findings relate to Spring CSRF protections, identified across almost all Controller and Controller Implementation classes, including:
    - `InvitationController` / `InvitationControllerImpl`
    - `BranchController` / `BranchControllerImpl`
    - `RepositoryController` / `RepositoryControllerImpl`
    - `ResourceController` / `ResourceControllerImpl`
    - `AuthController` / `AuthControllerImpl`

Specific endpoints or configurations lack adequate CSRF tokens or protection mechanisms, which could allow a malicious site to perform actions on behalf of an authenticated user.

*Summary of Risk Levels* -
The following table summarizes the overall risk level of the project's dependencies and vulnerabilities:

| Severity | Count | Primary Source | Impact |
|----------|-------|----------------|--------|
| Critical | 1     | spring-security-crypto | High: Risk of complete system compromise via auth bypass |
| High     | 13    | Various Libraries      | Significant: Potential DoS, data leaks, and server instability |
| Medium   | 7     | Netty, Spring, Commons | Moderate: Limited injection risks and resource throttling issues |
| Low      | 29    | Custom Code (CSRF)     | Minor: Localized security gaps in web request handling |

*Remediation Strategy*
Following a remediation strategy, the project's dependencies were updated to the latest versions, and the following vulnerabilities were addressed: 

- **Dependency Updates:** 18 of the 24 vulnerabilities are automatically fixable by upgrading the versions in the `pom.xml`. This should be the immediate priority.
- **CSRF Hardening:** Review the `WebSecurityConfigurerAdapter` (or `SecurityFilterChain`) to ensure CSRF protection is globally enabled for all POST, PUT, and DELETE methods.
- **Resource Management:** For "Allocation of Resources Without Limits" flags, ensure that timeouts and maximum request sizes are explicitly defined.



*Snyk Policy File (.snyk)* -
The project also uses a `.snyk` policy file to manage ignored vulnerabilities and document justifications:

```yaml
# Snyk (https://snyk.io) policy file
version: v1.25.0
ignore:
  SNYK-JAVA-COMFASTERXMLJACKSONCORE-15365924:
    - '*':
        reason: "This risk is managed by an internal validator that makes sure that the solution is not vulnerable"
        expires: 2026-06-01T00:00:00.000Z
  SNYK-JAVA-COMMONSLANG-10734077:
    - '*':
        reason: "Legacy dependency of JanusGraph that can't be updated, the solution should be to change to a newer technology"
        expires: 2026-06-01T00:00:00.000Z
  SNYK-JAVA-TOOLSJACKSONCORE-15365915:
    - '*':
        reason: "This risk has been evaluated as acceptable"
        expires: 2026-06-01T00:00:00.000Z
  SNYK-JAVA-TOOLSJACKSONCORE-15371178:
      - '*':
          reason: "Transitive dependency from Spring Boot 4.0.3 (Jackson 3.0.4). Resource limit risk is currently acceptable for this environment."
          expires: 2026-06-01T00:00:00.000Z
```

This policy ensures that known, acceptable risks are documented, while remaining vulnerabilities are actively managed and mitigated.

<div align="center">

[![Catturasnykpng.png](https://i.postimg.cc/Nj7b0hyG/Catturasnykpng.png)](https://postimg.cc/FYz0PBgw)
</div>

#### Git Guardian
During the initial scan of the project using GitGuardian, it was noted that the project requires numerous secrets, as exemplified by files such as:
 ```yaml
 MINIO_HOST=minio_host
 MINIO_PORT=minio_port
 MINIO_ROOT_USERNAME=minio_root_username
 MINIO_ROOT_PASSWORD=minio_root_password
 BLOB_STORAGE_BUCKET_MESHES=blob_storage_bucket_meshes
 BLOB_STORAGE_BUCKET_MATERIALS=blob_storage_bucket_materials
 
 MONGO_HOST=mongo_host
 MONGO_PORT=mongo_port
 MONGODB_ROOT_USERNAME=mongo_root_username
 MONGODB_ROOT_PASSWORD=mongo_root_password
 MONGODB_NAME=mongo_db_name
 MONGODB_COLLECTION_VERSIONS=mongo_db_collection_versions
 MONGODB_COLLECTION_TOKEN_BLACKLIST=mongo_db_collection_token_blacklist
 
 JANUS_HOST=janus_host
 JANUS_PORT=janus_port
 JANUS_USERNAME=janus_username
 JANUS_PASSWORD=janus_password
 
 MODELS_FOLDER_PATH=models_folder_path
 JWT_SECRET=jwt_secret
 FROM_EMAIL=from_email@example.com
 TO_EMAIL=to_email@example.com
 APP_PASSWORD=app_password
 ```

However, GitGuardian reported only a single instance: a dummy value embedded within a test file, which was subsequently removed. This outcome demonstrates that secret management within the project is handled effectively, with sensitive information properly safeguarded and no actual credentials exposed in the repository.

<div align="center">

[![Catturaguardianpng.png](https://i.postimg.cc/x8DbHC76/Catturaguardianpng.png)](https://postimg.cc/k6s4Pn7S)
</div>

### Testing and Code Coverage
#### Unit Tests with JUnit
#### Code Coverage with JaCoCo
#### Mutation Testing with pitest
#### Benchmarks with JMH

### CI/CD with GitHub Actions
#### Build, Test and Coverage
#### Security Analysis
#### Build and Deployment

### JML

## Conclusions