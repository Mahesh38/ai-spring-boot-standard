# Java Spring Boot coding standard

This repository is the coding standard for our Spring Boot microservices. People and AI agents follow it before they generate or change code.

The standard is [CODING_STANDARD.md](CODING_STANDARD.md). It covers naming shapes, package grouping and the order inside a source file, Palantir formatting, JDK-only utilities and MapStruct, design principles with accepted and rejected Java examples, the rule-list shape for adding a condition to an existing flow, the service-layer and configuration practices, design patterns, layer boundaries, the REST API profile in section 11, errors, persistence, zero-trust security and compliance in section 17, distributed logging in section 18, testing, and the Spring Boot 4.1 baseline (Java 25).

Cursor loads `.cursor/rules/java-spring-boot-standard.mdc` on every session and points agents at the standard. Copy both files into a service repository when you adopt the standard. Record any temporary local exception in that service's README under "Standard exceptions". Change the rules here, then update the copies. Do not edit a service's copy in place.
