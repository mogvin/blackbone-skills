---
name: blackbone_gateway
description: "Communication layer for the Blackbone ecosystem."
v: 1.0.0
platforms: [linux, macos, windows]
metadata:
  tags:
    - blackbone
    - gateway
    - communication
    - routing
    - orchestration
---
# Blackbone Gateway
# Hermes Universal Gateway Skill

## Skill Name

Hermes Universal Gateway

Version: 3.0.0

---

## Hermes Operating Rules

When interacting with the Blackbone ecosystem:

- Use the Blackbone Gateway for all communication with Blackbone applications.
- Do not communicate directly with OpenErii, Alice, CIEN, Trading Agent, or future Blackbone applications.
- The Gateway is the communication authority.
- The Gateway determines routing, authentication, provider selection, and response formatting.
- If application data is required, request it through the Gateway.
- Do not bypass the Gateway.
- Perform reasoning only after the Gateway returns the requested information.

# Purpose

The Hermes Universal Gateway is the centralized AI communication layer for the Blackbone ecosystem.

Its primary responsibility is to receive requests from local applications, determine how those requests should be processed, translate them into a universal format, forward them to Hermes Agent, and return provider-compatible responses.

Applications never communicate directly with OpenAI, Anthropic, OpenRouter, Ollama, Claude, Gemini, or any other model provider.

Instead, every AI request flows through this gateway.

The gateway acts as a protocol translator, application router, request manager, and response formatter.

---

# Project Goal

Create one AI communication layer capable of serving every local application.

Instead of building AI support into every application separately:

OpenErii

↓

Gateway

↓

Hermes

↓

Provider

↓

Gateway

↓

Application

The gateway isolates applications from provider changes.

If a provider changes its API, only the gateway requires modification.

Applications remain unchanged.

---

# Design Principles

The gateway follows several core principles.

• One communication layer

• Provider independent

• Application independent

• Modular architecture

• High concurrency

• Fault tolerant

• Easy to extend

• Local-first design

• API compatible

---

# Primary Responsibilities

The gateway is responsible for:

Receiving application requests

Authenticating applications

Detecting providers

Translating requests

Managing concurrent execution

Queueing requests

Executing Hermes

Receiving Hermes output

Building provider responses

Returning responses

Logging requests

Health monitoring

Model discovery

Application discovery

Error handling

Streaming support

---

# Supported Providers

Current providers

OpenAI

Anthropic

Future providers

OpenRouter

Ollama

Claude

Gemini

Groq

DeepSeek

Mistral

LM Studio

Local Python Models

Additional providers can be added without modifying connected applications.

---

# Supported Applications

Current

OpenErii

Alice

CIEN


Future

Up to twenty local applications.

Every application receives an independent configuration.

Applications do not know about each other.

---

# Request Flow

Incoming Request

↓

Authentication

↓

Provider Detection

↓

Translator

↓

Queue

↓

Hermes

↓

Translator

↓

Response Builder

↓

Outgoing Response

Every request follows this exact pipeline.

---

# Gateway Components

server.js

Runs the HTTP server.

Exposes every endpoint.

Coordinates the gateway.

---

translator.js

Converts every provider into one universal format.

Converts Hermes output back into provider format.

No provider communicates directly with Hermes.

---

responseBuilder.js

Creates responses compatible with:

OpenAI Chat Completions

OpenAI Responses API

Anthropic Messages API

Streaming

Errors

Health

Models

---

hermes.js

Communicates directly with Hermes CLI.

Executes requests.

Handles timeouts.

Manages running processes.

Supports cancellation.

Parses responses.

---

config.js

Contains every configurable setting.

Gateway

Providers

Applications

Limits

Logging

Streaming

Security

Timeouts

---

queue.js

Manages execution.

Responsibilities

Concurrency

Retries

Timeouts

Scheduling

Queue management

Burst handling

---

providerDetector.js

Determines which provider is being used.

Checks

Endpoint

Headers

Request body

Returns

OpenAI

Anthropic

Future providers

---

router.js

Routes requests internally.

Maps endpoints to the correct execution path.

---

appRegistry.js

Maintains registered applications.

Each application stores

Provider

Default model

API key

Permissions

Capabilities

---

models.js

Maintains available models.

Supports

Default model

Allowed models

Provider

Capabilities

Dynamic model loading

---

auth.js

Authenticates applications.

Supports

Bearer Tokens

API Keys

Application IDs

Permissions

Rate limits

---

Gateway Endpoints

/

Gateway information

/health

Gateway health

/apps

Registered applications

/v1/models

Model discovery

/v1/chat/completions

OpenAI Chat Completions

/v1/responses

OpenAI Responses API

/v1/messages

Anthropic Messages API

---

Universal Translation

Every provider is translated into one internal object.

OpenAI

↓

Universal Request

↓

Hermes

↓

Universal Response

↓

OpenAI

Anthropic follows the exact same process.

This removes provider-specific logic from Hermes.

Hermes only understands one internal format.

---

Application Registration

Every application registers itself with the gateway.

Each registration includes

Application name

Provider

Model

API key

Permissions

Capabilities

Maximum concurrency

Applications are isolated.

One application cannot interfere with another.

---

Concurrency

The gateway is designed for multiple simultaneous applications.

Target

20 Applications

↓

200 Concurrent Requests

↓

Queue

↓

Hermes

↓

Responses

Requests never execute directly.

Everything passes through the queue.

---

Streaming

Supported

OpenAI Streaming

Anthropic Streaming

Future SSE endpoints

Streaming responses are generated by Response Builder.

---

Logging

Gateway records

Requests

Responses

Execution time

Provider selection

Application routing

Errors

Queue statistics

Health information

Logs are intended for debugging and monitoring.

---

Health Monitoring

The gateway continuously exposes

Running processes

Queue size

Registered applications

Hermes health

Gateway uptime

Current version

Provider availability

---

Error Handling

Every failure is normalized.

Examples

Authentication failures

Provider failures

Timeouts

Queue overflow

Hermes execution failures

Translation failures

Unknown exceptions

Applications receive consistent error objects regardless of provider.

---

Future Expansion

The gateway has been designed to support

Additional providers

Plugin system

Remote gateways

Distributed execution

Multiple Hermes instances

Load balancing

Retriever integration

Memory integration

Tool execution

Windows services

Linux services

Cloud deployment

No architectural redesign should be required.

---

Relationship with Retriever

Retriever is not part of the gateway.

Retriever gathers information.

Gateway communicates with AI.

Retriever

↓

Universal Data

↓

Gateway

↓

Hermes

↓

Reasoning

↓

Gateway

↓

Application

Retriever and Gateway are separate systems connected through well-defined interfaces.

---

Relationship with Hermes

Hermes is the reasoning engine.

The gateway never performs reasoning.

Responsibilities

Gateway

Communication

Translation

Authentication

Routing

Queueing

Formatting

Hermes

Reasoning

Tool execution

Model interaction

Decision making

Keeping these responsibilities separate improves maintainability and scalability.

---

Long-Term Vision

The Hermes Universal Gateway is intended to become the permanent AI communication layer for the entire Blackbone ecosystem.

Every current and future application should interact with AI exclusively through this gateway.

By separating communication, translation, routing, authentication, and response formatting from AI reasoning, the system remains modular, provider-independent, and scalable.

The gateway is designed to support twenty or more applications, hundreds of concurrent requests, multiple AI providers, and future Blackbone services without requiring changes to individual applications.

Its role is to provide a stable, extensible, and centralized interface between applications and intelligent systems while allowing Hermes to focus solely on reasoning and execution.