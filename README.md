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
        - [SonaQube Cloud](#sonarqube-cloud)
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
#### SonaQube Cloud
#### Snyk
#### Git Guardian

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