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
The unit testing strategy for the project underwent a significant transformation, evolving from a minimal set of checks to a robust, professional-grade test suite.

*Initial State: Minimal Happy Path Coverage* -
Originally, the project suffered from a severe lack of automated verification. Testing was restricted to a few rudimentary cases targeting only the "happy paths" of a handful of classes within the Repository layer. These tests merely confirmed that the system worked under ideal conditions, leaving the application highly vulnerable to edge cases, invalid inputs, and unexpected runtime failures. Large sections of the Business Logic and Validator layers remained entirely untested, meaning that even minor modifications could introduce undetected regressions.

*LLM-Assisted Expansion* -
To bridge this gap, an **LLM-driven testing strategy** was implemented. By providing the AI with the source code of the classes and the original repository tests as structural references, the LLM was tasked to act as a professional tester. It applied the **Category Partition Method**, a formal specification-based testing technique. This approach systematically identified all possible input categories and boundary conditions for each method. The LLM generated comprehensive test suites for Services, Validators, and Repositories, ensuring that not only the happy paths were covered, but also error states and edge cases that had been previously ignored. This process enabled the automated discovery of unusual input combinations, missing null checks, and potential concurrency issues.

*Post-Expansion Results: Robust Verification* -
The integration of these LLM-generated tests led to a dramatic increase in the project's reliability. The final execution report shows a total of **285 unit tests** across **22 test classes**, all passing successfully. Notable coverage highlights include:

- **BlobStorageVersionRepositoryImplTest:** 43 tests
- **GremlinRepositoryRepositoryImplTest:** 31 tests
- **UserValidatorImplTest:** 18 tests
- **Service Layer:** Comprehensive coverage across Auth, Invitation, and Versioning services (averaging 10-12 tests per service)

This rigorous suite provides a safety net that confirms the internal consistency of the codebase and ensures that new refactoring efforts—such as those required by the SonarQube analysis—do not introduce regressions. It also facilitates future maintainability by providing clear, structured tests that can be reused or extended as the application evolves.

*Bug Identification through Rigorous Testing* -
Despite the high pass rate, the transition to more granular testing successfully unmasked critical defects that had stayed hidden during manual development. Examples include:

- **Validation Failure:** A specific bug was identified in `testShowVersionMetadata`. The test failed when provided with null values for both User and Comment. This revealed that the service layer lacked necessary null-checks or default value handling for metadata retrieval, which would have caused a `NullPointerException` in production.

- **Operational Resilience:** By simulating `DBError: true` scenarios in parameterized tests like `testCreateVersion`, the team verified how the system handles persistence failures, ensuring that exceptions are propagated correctly or handled gracefully rather than failing silently.

These discoveries validate the necessity of the **Category Partition approach**, as it forced the application to confront "unhappy" scenarios that the original minimal testing would never have caught. The strategy significantly strengthened confidence in the codebase and provided a foundation for continued professional software engineering practices.

#### Code Coverage with JaCoCo
The implementation of the professional test suite, combined with a refined **JaCoCo** configuration, resulted in a dramatic improvement of the project's code coverage metrics. This transition moved the project from a "blind" development state to a **data-driven quality assurance model**, enabling developers to make informed decisions based on measurable code quality.

*Initial Coverage State: High Risk and "Red" Packages* -
Before the expansion of the test suite, the JaCoCo report reflected a critical lack of automated testing. Core packages—including **Validators**, **Service Implementations**, and **Repositories**—displayed 0% or near-zero coverage. Key metrics included:

- **Total Missed Instructions:** Over 7,300
- **Critical Gaps:** Strategic components such as `VersionServiceImpl`, `GremlinAuthRepository`, and nearly all Validator classes were completely untested, highlighted in red.
- **Single Points of Success:** The `BlobStorage` implementation had 87% coverage, but this was isolated and did not reflect the overall system health.

This situation posed a high risk of undetected defects and regression errors, making the project vulnerable to failures in production.

*Strategic JaCoCo Configuration* -
To obtain realistic insights into the health of the business logic, the `jacoco-maven-plugin` was configured to **exclude boilerplate and infrastructure code**, such as:
- DTOs
- Configuration classes
- Generated JMH benchmarks
- Controllers (usually handled via integration tests)

This allowed the coverage metrics to focus strictly on **Service**, **Repository**, and **Validator layers**. Additionally, a strict quality gate was defined in the Maven lifecycle to enforce high standards:

```xml
<limit>
    <counter>LINE</counter>
    <value>COVEREDRATIO</value>
    <minimum>0.80</minimum>
</limit>
```

This ensured that any build dropping below **80% line coverage** would fail, institutionalizing high code quality as a mandatory requirement.

*Post-Remediation Results: Reaching the 80% Threshold* -
After integrating the LLM-generated tests using **Category Partitioning**, coverage metrics improved dramatically. The previously "red" areas were replaced by a "green" landscape, as summarized below:

| Metric | Initial State (Pre-Fix) | Final State (Post-Fix) | Improvement |
|--------|-------------------------|------------------------|------------|
| Total Instruction Coverage | 6% | 84% | +78% |
| Branch Coverage | 3% | 69% | +66% |
| Missed Instructions | 7,325 | 943 | -87% |
| Fully Covered Packages (100%) | 1 (Exceptions) | 12+ (Services/Validators) | Significant |

*Detailed Package Improvements:* -
The following areas of the codebase were significantly improved:

- **Service Layer:** Packages like `AuthServiceImpl` and `InvitationServiceImpl` jumped from 0% to 95-100% coverage.
- **Validator Layer:** Classes such as `InvitationValidator` and `ResourceValidator` reached 98-100% coverage.
- **Repository Layer:** Complex logic in Gremlin and Cosmos implementations, previously untested, now ranges between 78% and 100% coverage.

<div align="center">

[![Catturapostjacoco.png](https://i.postimg.cc/02gDLtVf/Catturapostjacoco.png)](https://postimg.cc/w3V1Nk8y)
</div>

The remaining 16% of missed instructions are primarily concentrated in the `Gremlin.versioning.version` and `Validators.versioning.version` packages (approx. 51-68% coverage). These areas contain highly complex conditional logic and nested loops that require specialized test cases to exercise the remaining branches. Despite this, the overall project **comfortably exceeds the 80% line coverage threshold**, marking a successful transition to a professionally tested and maintainable codebase.

#### Mutation Testing with pitest
Following the substantial increase in code coverage, a **pitest (Mutation Testing)** analysis was conducted to evaluate the actual strength and effectiveness of the test suite. While standard code coverage (JaCoCo) measures which lines of code are executed, pitest measures the tests' ability to detect injected faults (mutations).

*Executive Summary of Metrics* -
The project summary reveals a robust testing foundation with room for refinement in specific complex logic areas:

- **Number of Classes Analyzed:** 24
- **Line Coverage:** 84% (1166/1385 lines), consistent with recent JaCoCo improvements.
- **Mutation Coverage:** 59% (304/518 mutations killed)
- **Test Strength:** 80% (304/382)

The 80% Test Strength indicates that when the tests actually cover a piece of code, they are highly effective at catching bugs. The lower overall Mutation Coverage (59%) is primarily due to mutations in code that is not yet fully reached by the test suite.

*Package-Level Performance Analysis* -
The following table summarizes the performance of each package in the project:

| Package / Layer | Line Coverage | Mutation Coverage | Test Strength |
|-----------------|---------------|-----------------|---------------|
| Service Layer (Invitation, Branch, Repo) | 94% - 100% | 73% - 100% | 73% - 100% |
| Validators (JWT, Invitation, User) | 79% - 100% | 69% - 100% | 80% - 100% |
| Persistence/DB (Cosmos, BlobStorage, Gremlin) | 55% - 100% | 42% - 88% | 63% - 88% |

*High-Performance Areas (Success Stories)* -
The following areas demonstrate the project's ability to maintain high levels of code quality and reliability:

- **Cosmos & JWT:** Both `cosmos.auth` and `validators.auth.JWT` achieved 100% Mutation Coverage and 100% Test Strength, demonstrating exhaustive tests for critical security components.
- **Invitation & Branch Services:** Near-perfect line coverage and high mutation scores (100% test strength) validate the stability of business logic for invitations and repository branching.

*Areas for Improvement (Legacy & Complex Logic)* -
The following areas highlight areas of potential improvement, particularly in the `Validators.versioning.version` and `Gremlin.versioning.version` packages:

- **Gremlin Versioning (version):** Lowest scores with 55% Line Coverage and 45% Mutation Coverage. Reflects the high complexity of JanusGraph-related logic where many branches remain unexercised.
- **Version Service Implementation:** 88% Line Coverage but only 33% Mutation Coverage. Indicates that lines are executed but assertions are not sensitive enough to detect logic changes.

*Strategic Recommendations* -
The following recommendations highlight areas where the project can be further refined to achieve higher levels of test coverage and reliability:

1. **Strengthen Version Service Assertions:** Add granular assertions to verify object state changes rather than just absence of exceptions.
2. **Focus on Gremlin Edge Cases:** Target surviving 45% of mutations in `it.unisa.ddditserver.db.gremlin.versioning.version` with new test cases.
3. **Monitor Mutation Expiration:** Use this pitest report as baseline; future migration from JanusGraph should aim to maintain or exceed 80% Test Strength.

<div align="center">

[![pites.png](https://i.postimg.cc/v82BxXg9/pites.png)](https://postimg.cc/BtDsdTCQ)
</div>

#### Benchmarks with JMH

This report presents an in-depth analysis of the JMH benchmark results conducted across the project's core services and validators. The goal is to identify performance bottlenecks, highlight areas for optimization, and guide development efforts towards classes that most impact execution time.

*Benchmark Scope and Configuration* -
The benchmarks cover a wide range of classes across **Service** and **Validator** layers, including:

- **Service Layer:** `BranchServiceImpl`, `RepositoryServiceImpl`, `ResourceServiceImpl`, `VersionServiceImpl`
- **Validator Layer:** `BranchValidatorImpl`, `RepositoryValidatorImpl`, `ResourceValidatorImpl`, `VersionValidatorImpl`

The JMH configuration included:

- **Mode:** Average Time (avgt)
- **Forks:** 2
- **Warmup Iterations:** 3
- **Measurement Iterations:** 10
- **Units:** Nanoseconds (ns) or microseconds (us) depending on method

This setup ensures a statistically reliable measurement of typical execution paths under realistic scenarios.

*Service Layer Observations* -
In the **Service Layer**, most methods execute efficiently, with some clear differences based on operational complexity:

- **BranchServiceImpl:** Both `benchmarkCreateBranch` and `benchmarkListBranchesByResource` average ~37 µs per operation. Performance is stable and suggests minimal optimization is needed for typical CRUD operations in branch management.
- **RepositoryServiceImpl:** Methods such as `benchmarkCreateRepository` (~26,645 ns), `benchmarkListRepositoriesContributed` (~15,857 ns), and `benchmarkListRepositoriesOwned` (~15,961 ns) indicate significantly higher execution costs. This is consistent with repository logic involving database lookups and validation sequences. Optimization here should focus on caching repeated queries and reducing redundant data retrieval.
- **ResourceServiceImpl:** Operations like `benchmarkCreateResource` and `benchmarkShowVersionTree` are similar to BranchServiceImpl (~37 µs), showing good efficiency.
- **VersionServiceImpl:** While `benchmarkPullVersion` and `benchmarkShowVersionMetadata` are ~32 µs, `benchmarkCreateVersion` rises to 54 µs, reflecting additional metadata computation and potential I/O interactions. Targeted optimization may include memoization and selective validation of unchanged fields.

Service methods with nested validations or repository interactions (RepositoryServiceImpl and VersionServiceImpl) are prime candidates for performance improvement.

*Validator Layer Analysis* -
Validator classes show higher variance in execution times, reflecting the complexity of validation logic:

- **BranchValidatorImpl:** `benchmarkValidateBranch` (~10,654 ns) is relatively lightweight, but `benchmarkValidateFullChain` (~25,179 ns) indicates significant computational overhead when chaining validations.
- **RepositoryValidatorImpl:** Simple name checks (`benchmarkIsValidRepositoryName` ~183 ns) are extremely fast, but `benchmarkValidateRepository` (~189 ns) and `benchmarkValidateFull` (~5,388 ns) highlight the cost of full DTO validation.
- **ResourceValidatorImpl:** Name validation (`benchmarkIsValidResourceName` ~49 ns) is negligible, while `benchmarkValidateExistence` (~15,483 ns) and `benchmarkValidateResource` (~5,302 ns) are heavier due to conditional logic and simulated repository interactions.
- **VersionValidatorImpl:** Exhibits the highest execution times in the validator layer:
    - `benchmarkIsValidComment` (~364 ns) is lightweight
    - `benchmarkIsValidVersionName` (~56 ns) is extremely fast
    - `benchmarkValidateExistence` (~20,623 ns) and `benchmarkValidateVersionMesh` (~35,846 ns) indicate complex nested checks and branching.

The VersionValidator and BranchValidator full-chain methods are the most CPU-intensive, suggesting opportunities for refactoring, caching, or breaking down complex validation chains into smaller, benchmarked units.

| Class / Benchmark Method | Mode | Iterations | Score (µs/op) | Error (µs/op) |
|--------------------------|------|------------|---------------|---------------|
| BranchServiceImplBenchmark.benchmarkCreateBranch | avgt | 10 | 37,142 | ±0,386 |
| BranchServiceImplBenchmark.benchmarkListBranchesByResource | avgt | 10 | 37,371 | ±0,160 |
| RepositoryServiceImplBenchmark.benchmarkCreateRepository | avgt | 10 | 26645,772 | ±0,289 |
| RepositoryServiceImplBenchmark.benchmarkListRepositoriesContributed | avgt | 10 | 15857,198 | ±0,056 |
| RepositoryServiceImplBenchmark.benchmarkListRepositoriesOwned | avgt | 10 | 15961,212 | ±0,091 |
| ResourceServiceImplBenchmark.benchmarkCreateResource | avgt | 10 | 37,116 | ±0,574 |
| ResourceServiceImplBenchmark.benchmarkShowVersionTree | avgt | 10 | 37,212 | ±0,322 |
| VersionServiceImplBenchmark.benchmarkCreateVersion | avgt | 10 | 54,119 | ±0,530 |
| VersionServiceImplBenchmark.benchmarkPullVersion | avgt | 10 | 32,284 | ±0,185 |
| VersionServiceImplBenchmark.benchmarkShowVersionMetadata | avgt | 10 | 32,374 | ±0,259 |
| BranchValidatorImplBenchmark.benchmarkValidateBranch | avgt | 10 | 10654,682 | ±0,047 |
| BranchValidatorImplBenchmark.benchmarkValidateFullChain | avgt | 10 | 25179,768 | ±0,214 |
| RepositoryValidatorImplBenchmark.benchmarkIsValidRepositoryName | avgt | 10 | 0,183 | ±0,001 |
| RepositoryValidatorImplBenchmark.benchmarkValidateRepository | avgt | 10 | 0,189 | ±0,001 |
| RepositoryValidatorImplBenchmark.benchmarkValidateFull | avgt | 10 | 5388,817 | ±0,051 |
| ResourceValidatorImplBenchmark.benchmarkIsValidResourceName | avgt | 10 | 0,049 | ±0,0003 |
| ResourceValidatorImplBenchmark.benchmarkValidateExistence | avgt | 10 | 15483,676 | ±0,094 |
| ResourceValidatorImplBenchmark.benchmarkValidateResource | avgt | 10 | 5302,181 | ±0,025 |
| VersionValidatorImplBenchmark.benchmarkIsValidComment | avgt | 10 | 0,365 | ±0,002 |
| VersionValidatorImplBenchmark.benchmarkIsValidVersionName | avgt | 10 | 0,057 | ±0,0002 |
| VersionValidatorImplBenchmark.benchmarkValidateExistence | avgt | 10 | 20623,618 | ±0,247 |
| VersionValidatorImplBenchmark.benchmarkValidateVersionMesh | avgt | 10 | 35846,745 | ±0,959 |

*Optimization Recommendations* - Based on the benchmark analysis:
1. **Target High-Impact Methods:** Focus on `VersionValidatorImpl.validateVersionMesh`, `BranchValidatorImpl.validateFullChain`, and repository creation/listing methods.
2. **Caching and Memoization:** Store intermediate results for repeated validation checks, especially for nested repository and version validations.
3. **Refactor Complex Chains:** Break down full-chain validations into modular, testable units to reduce per-operation cost.
4. **Parallelize Independent Checks:** In validators with multiple independent field validations, consider parallel execution to leverage multicore CPUs.
5. **Monitor Execution in CI/CD:** Include these JMH benchmarks in automated pipelines to detect regressions and measure impact of refactorings.

The JMH benchmark results demonstrate that while many operations are highly efficient, the **Validator and Repository layers** contain methods that dominate execution time. Focusing on these high-impact methods, refactoring nested validations, introducing caching, and leveraging parallelism will provide the greatest improvements in overall system performance. By following these recommendations, the project can achieve a more predictable and scalable performance profile.

### CI/CD with GitHub Actions
#### Build, Test and Coverage

The build and test jobs form the backbone of the CI/CD process. The pipeline leverages Maven with JDK 21 to compile the project and manage dependencies. The `./mvnw install -DskipTests` step ensures that all dependencies are resolved and that the project is structurally sound before running any tests or analysis.

Unit tests are executed automatically in the pipeline to verify functional correctness. By combining standard unit tests with advanced LLM-assisted test generation and category partitioning, the project ensures comprehensive coverage of edge cases, error handling, and core business logic.

**Code coverage metrics** are gathered using JaCoCo, with a strict quality gate configured to require at least 80% line coverage. The pipeline focuses on the Service, Repository, and Validator layers while excluding DTOs, configuration classes, controllers, and boilerplate code, ensuring that coverage reflects true business logic verification. This configuration prevents regressions in critical areas and highlights classes that need additional testing or refactoring.

The use of caching for Maven dependencies (`~/.m2/repository`) and Buildx for Docker builds optimizes performance and minimizes redundant downloads, improving build times for repeated runs and allowing fast feedback loops for developers.

#### Security Analysis

Security is deeply integrated into the CI/CD workflow through three complementary mechanisms: GitGuardian, Snyk, and SonarQube.

*GitGuardian Scan* - GitGuardian is employed to detect any secrets accidentally committed into the repository, such as API keys or credentials. The scan runs on all commits and pull requests to the `localenv` branch.

*Snyk Security Scan* - Snyk performs static and dynamic analysis of both dependencies and custom code. The workflow installs the Snyk CLI, sets the `SNYK_TOKEN`, and runs `snyk test` with a severity threshold of high. Vulnerabilities are automatically detected and classified into Critical, High, Medium, and Low. For dependencies flagged with unresolved issues, Snyk provides remediation guidance, highlighting upgrade paths for open-source libraries and potential code fixes. This ensures the project minimizes risks from authentication bypass, uncontrolled recursion, HTTP request smuggling, and other security flaws.

*SonarQube Analysis* - SonarQube provides a continuous assessment of code quality, maintainability, and potential bugs. The workflow sets up JDK 21, caches SonarCloud packages, and executes `mvn verify` with the `sonar-maven-plugin`. The analysis measures metrics such as code duplication, cyclomatic complexity, code smells, and potential vulnerabilities. By enforcing the Sonar quality gate, the pipeline prevents code that introduces maintainability or security issues from merging into the main branch.

#### Build and Deployment

The build-and-deploy jobs integrate Docker to package the application and ensure consistent runtime environments across development, staging, and production servers.

*Docker Build and Push* -
The pipeline sets up Docker Buildx for multi-platform builds and logs into Docker Hub using stored secrets.
Docker images are built with caching enabled to reduce build times and pushed with two tags: `latest` and the current Git commit SHA, ensuring traceability.
This approach guarantees reproducibility and supports rollback strategies if a deployment fails.

*Remote Deployment* - 
Deployment is handled securely using SSH and Tailscale VPN connectivity.
The workflow sets up the SSH key and known hosts, transfers the `docker-compose-actions.yml` file to the remote server,
pulls the latest Docker images, and launches all services with `docker compose up -d`. After deployment, old Docker images are pruned to maintain disk hygiene. This deployment strategy ensures that production systems are updated safely and consistently without manual intervention, reducing human error and downtime.

Overall, this CI/CD pipeline enforces high standards of quality, security, and operational efficiency. Every commit undergoes rigorous verification, ensuring that only tested, secure, and reliable code is delivered. The combination of automated builds, unit testing, code coverage, mutation testing, dependency scanning, and containerized deployment creates a robust foundation for continuous delivery and rapid iteration.

### JML
To further enhance the reliability and formal correctness of the codebase, the project adopted **JML (Java Modeling Language)** to implement a **Design by Contract (DbC)** paradigm. By using formal annotations within the source code (e.g., in `BranchServiceImpl`), the team moved beyond simple documentation to a system of rigorous behavioral specifications.

*Behavioral Constraints* -
Through the use of `/*@ public normal_behavior @*/` and `/*@ public exceptional_behavior @*/`, each method explicitly defines its logical boundaries. This includes:

- **Pre-conditions (`requires`)**: specify the valid state of inputs like tokens and DTOs.
- **Post-conditions (`ensures`)**: guarantee the state of the result or the system after execution.

*Data Integrity* -
Annotations such as `/*@ spec_public non_null @*/` ensure that critical internal dependencies—like repositories and validators—are never in an inconsistent or null state, effectively preventing `NullPointerException`s at the architectural level.

*Exception Transparency* -
JML was used to formalize exactly which exceptions should be thrown under specific conditions (e.g., `signals_only NotLoggedUserException`), providing a clear roadmap for the `BranchServiceImplTest` suite to verify error-handling logic.

*Challenges and Technical Management of OpenJML* -
Integrating formal verification into a modern Java stack presents significant technical hurdles, particularly when using frameworks like Spring and libraries like Lombok. While JML provides a powerful framework for mathematical correctness, it is natively designed for "plain" Java, creating a gap when faced with automated bytecode generation and Dependency Injection (DI).
The primary challenge lies in the fact that **OpenJML** operates on source code, but:

- **Lombok** generates significant portions of code (getters, setters, constructors) only during compilation.
- **Spring** manages bean lifecycles and dependency injection at runtime.

For OpenJML, these components appear as "missing" or "undefined," leading to false positives and verification stalls.

*The Custom Automation Solution* -
To bridge this gap, a custom automation script, `jml-check.sh`, was developed to orchestrate a sophisticated multi-stage analysis pipeline. The script automates the following critical steps:

- **Delombok Pre-processing**: Transforms the source code into "standard" Java before analysis, so OpenJML can see all generated methods and constructors required by JML contracts.
- **Dynamic Classpath Resolution**: Builds a comprehensive classpath, manually including essential JARs (Spring Core, Web, Beans) to ensure all external type definitions are available.
- **Optimized Parallel Execution**: Splits the source file list into chunks and executes OpenJML across all CPU threads, reducing analysis time from tens of minutes to a manageable duration.

*Handling Analysis Warnings and Limitations* -
Even with a refined pipeline, several types of warnings are reported, which are considered **known limitations** rather than logic errors:

- **NullField & Invariant Violations**: OpenJML may flag potential nullity or state inconsistencies due to limited understanding of Spring’s `@Autowired` lifecycle or internal library invariants (e.g., `CharSequence`).
- **Arithmetic Range Warnings**: OpenJML is conservative about potential overflows, often flagging standard arithmetic operations that are safe within the project's context.

## Conclusions

In conclusion, the project underwent a rigorous transformation that elevated the codebase from a functional prototype to a more professionally engineered system.
By integrating SonarQube and Snyk, we successfully mitigated critical security vulnerabilities and architectural technical debt, moving toward a more secure and maintainable infrastructure.
The testing strategy evolved significantly: unit testing was expanded through Category Partitioning, achieving over 80% coverage verified by JaCoCo, while pitest confirmed high test strength of 80%, ensuring that the suite is capable of detecting real-world bugs.
Furthermore, the adoption of JML and formal verification scripts institutionalized the Design by Contract paradigm, providing mathematical grounding for code correctness despite the complexities of Spring and Lombok.
This holistic approach—combining automated security, formal specifications, and mutation testing—has established a robust foundation for long-term scalability and software quality.