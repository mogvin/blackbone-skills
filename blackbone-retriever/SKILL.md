---
name: blackbone_retriever
description: "Retrieves, validates and normalizes data for Hermes."
v: 1.0.0
platforms: [linux, macos, windows]
metadata:
  tags:
    - blackbone
    - retriever
    - market-data
    - trading
    - data
---

# Blackbone Retriever Skill

## Skill Name

Blackbone Retriever

Version: 1.0.0

---

# Purpose

The Blackbone Retriever is the centralized data acquisition layer for the Blackbone ecosystem.

Its responsibility is to discover, collect, normalize, validate, cache, and distribute information from local applications, external APIs, local files, databases, and operating system resources.

The Retriever does not perform AI reasoning.

It supplies clean, structured, and reliable information to Hermes Universal Gateway and Hermes Agent, allowing the AI to reason using the most current data available.

The Retriever acts as the system's eyes and ears.

---

# Project Goal

Provide one universal retrieval engine capable of supplying data from every supported source.

Instead of every application implementing its own data retrieval:

Applications

↓

Retriever

↓

Universal Objects

↓

Gateway

↓

Hermes

↓

Applications

The Retriever becomes the single source of truth for all runtime information.

---

# Design Principles

The Retriever follows several core principles.

• Read-only by default

• Source independent

• Application independent

• Local-first architecture

• Modular connectors

• Real-time capable

• Event driven

• Highly scalable

• Fault tolerant

• Cache aware

• API independent

• Extensible

---

# Primary Responsibilities

The Retriever is responsible for:

Discovering applications

Reading application data

Watching local files

Reading configuration files

Monitoring logs

Collecting API data

Receiving WebSocket events

Reading databases

Normalizing data

Validating retrieved information

Caching data

Detecting changes

Publishing updates

Supplying Hermes with structured information

Maintaining source health

Monitoring connector status

Managing retrieval schedules

---

# What the Retriever Does NOT Do

The Retriever never:

Perform AI reasoning

Generate responses

Modify application data

Execute trades

Change databases

Overwrite files

Communicate directly with language models

Replace Hermes

Replace the Gateway

Its responsibility ends after delivering validated data.

---

# Supported Data Sources

Current applications

BB Terminals

Osiris

Trading Agent

Obsidian

Future applications

Up to twenty local applications.

Future connectors may include

TradingView

MetaTrader

Exness

Deriv

Binance

Bybit

Interactive Brokers

Custom desktop applications

Web applications

Python services

Node.js services

Windows services

---

# Supported Resource Types

Local folders

JSON files

CSV files

TXT files

XML files

YAML files

SQLite databases

MongoDB

PostgreSQL

MySQL

REST APIs

WebSocket APIs

Named Pipes

Windows Registry

Environment Variables

Shared Memory

Process Memory

System Performance Counters

GPU information

CPU information

RAM statistics

Disk statistics

Network statistics

---

# Overall Architecture

Applications

↓

Application Connectors

↓

Collectors

↓

Parsers

↓

Validators

↓

Normalizers

↓

Universal Objects

↓

Cache

↓

Distribution Layer

↓

Gateway

↓

Hermes

Each stage has a single responsibility.

---

# Retriever Components

## processManager.js

Discovers and monitors running applications.

Responsibilities

Process discovery

Process status

PID management

Application availability

Automatic reconnect

---

## connectorManager.js

Maintains every connector.

Responsibilities

Start connectors

Stop connectors

Restart connectors

Health monitoring

Connection pooling

---

## connectorRegistry.js

Stores every available connector.

Each connector includes

Application

Capabilities

Priority

Connection method

Version

Status

---

## apiCollector.js

Reads REST APIs.

Supports

GET

POST

Authentication

Headers

Rate limits

Retries

Timeouts

---

## websocketCollector.js

Maintains persistent WebSocket connections.

Supports

Reconnect

Heartbeat

Subscriptions

Live event streams

---

## fileCollector.js

Reads local files.

Supports

JSON

CSV

TXT

XML

YAML

Binary files

Automatic file watching

---

## databaseCollector.js

Reads databases.

Supports

SQLite

MongoDB

PostgreSQL

MySQL

Read-only queries

Connection pools

---

## parser.js

Converts raw information into structured objects.

Every connector produces the same internal format.

---

## validator.js

Checks incoming information.

Removes invalid data.

Detects corrupted information.

Verifies required fields.

---

## normalizer.js

Transforms every source into one universal structure.

Applications never need to understand source-specific formats.

---

## cacheManager.js

Stores recently retrieved information.

Supports

Memory cache

Disk cache

Expiration

Refresh

Version tracking

---

## eventBus.js

Distributes updates throughout the Retriever.

Allows components to communicate without direct dependencies.

---

## scheduler.js

Controls retrieval frequency.

Supports

Real-time

Polling

Scheduled tasks

Manual refresh

Priority scheduling

---

## resourceManager.js

Tracks every monitored resource.

Applications

Files

APIs

Databases

Processes

System resources

---

## healthMonitor.js

Continuously checks

Connector health

Application status

API availability

Database availability

Cache health

Retriever uptime

---

## logger.js

Records

Errors

Warnings

Retrieval time

Connection failures

Application status

Performance metrics

---

# Retrieval Pipeline

Application

↓

Connector

↓

Collector

↓

Parser

↓

Validator

↓

Normalizer

↓

Cache

↓

Publisher

↓

Gateway

↓

Hermes

Every connector follows the same pipeline.

---

# Universal Object

Every retrieved item is converted into a universal object.

Example fields

Source

Application

Category

Timestamp

Identifier

Payload

Metadata

Health

Version

This removes application-specific formats from the rest of the system.

---

# Application Discovery

The Retriever automatically detects supported applications.

Discovery methods include

Running processes

Configuration files

Known installation paths

Executable scanning

Manual registration

Applications can reconnect automatically after restarting.

---

# Retrieval Modes

Manual retrieval

Scheduled retrieval

Real-time retrieval

Event-driven retrieval

On-demand retrieval

Background retrieval

Bulk synchronization

Incremental synchronization

---

# Caching

The Retriever maintains intelligent caches.

Benefits

Lower API usage

Faster responses

Reduced disk access

Offline support

Improved reliability

Cached data always includes timestamps and freshness information.

---

# Health Monitoring

Every connector reports

Running

Disconnected

Paused

Failed

Recovering

Health information is exposed to the Gateway.

---

# Error Handling

Every connector handles failures independently.

Examples

Missing files

Disconnected APIs

Database failures

Application shutdown

Network interruption

Invalid responses

Authentication failures

Automatic retry logic minimizes downtime.

---

# Performance Goals

Support

20 Applications

Hundreds of files

Multiple databases

Numerous APIs

Persistent WebSocket connections

Continuous monitoring

Without interrupting Hermes execution.

---

# Relationship with Hermes Universal Gateway

The Retriever does not communicate with applications through AI.

Instead

Applications

↓

Retriever

↓

Structured Data

↓

Gateway

↓

Hermes

↓

Gateway

↓

Applications

The Gateway handles communication.

The Retriever handles information.

---

# Relationship with Hermes

Hermes never retrieves data directly.

Hermes requests information.

Retriever supplies information.

Hermes reasons over that information.

Responsibilities

Retriever

Collection

Normalization

Validation

Caching

Distribution

Hermes

Reasoning

Planning

Decision making

Tool execution

Natural language understanding

This separation keeps both systems independent.

---

# Security

The Retriever is designed as a read-only subsystem by default.

It should never modify application data unless explicitly extended with write-enabled modules.

Every connector operates with the minimum permissions necessary.

Sensitive information should remain inside connector boundaries whenever possible.

---

# Future Expansion

The Retriever architecture supports future additions without structural redesign.

Planned enhancements include

Plugin-based connectors

Cloud resource collection

Distributed retrievers

Remote agents

Container monitoring

Virtual machine monitoring

Windows services

Linux services

macOS support

Shared retrieval clusters

Enterprise deployments

Automatic connector discovery

AI-assisted connector generation

---

# Long-Term Vision

The Blackbone Retriever is intended to become the permanent data acquisition layer for the Blackbone ecosystem.

Every application, service, and resource should expose its information through the Retriever rather than implementing independent retrieval logic.

By separating data acquisition from AI reasoning and communication, the Blackbone architecture remains modular, scalable, maintainable, and provider-independent.

The Retriever continuously supplies reliable, normalized, and validated information, allowing Hermes Universal Gateway and Hermes Agent to operate with complete awareness of the surrounding environment while remaining focused solely on communication and intelligent reasoning.