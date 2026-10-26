# Java Spring Boot coding standard

This repository is the coding standard for our Spring Boot microservices. People and AI agents follow it before they generate or change code.

The standard is [CODING_STANDARD.md](CODING_STANDARD.md). It covers naming shapes, package grouping and the order inside a source file, Palantir formatting, design principles with accepted and rejected Java examples, the service-layer and configuration practices, design patterns, layer boundaries, the REST API profile in section 11, errors, persistence, security, testing, and the Spring Boot 4.1 baseline (Java 25).

Cursor loads `.cursor/rules/java-spring-boot-standard.mdc` on every session and points agents at the standard. Copy both files into a service repository when you adopt the standard. Record any temporary local exception in that service's README under "Standard exceptions". Change the rules here, then update the copies. Do not edit a service's copy in place.
