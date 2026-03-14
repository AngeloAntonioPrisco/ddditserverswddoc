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