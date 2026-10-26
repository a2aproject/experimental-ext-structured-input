# Structured Input Extension

> **Experimental:** This extension currently has experimental status — breaking changes are possible and community feedback is much appreciated. To learn more about the extension lifecycle, please refer to [A2A Extension and Protocol Binding Governance](https://a2a-protocol.org/latest/topics/extension-and-binding-governance/).

This repository contains the specification for the **Structured Input Extension** for the Agent2Agent (A2A) protocol.

## Purpose

The Structured Input extension standardizes how agents use the `input-required` task state to request structured input from clients.

An agent attaches a JSON Schema to its input request, and the task resumes only when the client provides data satisfying it. Invalid input re-enters the `input-required` state with validation errors attached, so clients can correct it without canceling the task.

The extension enables Human-In-The-Loop flows, such as tool call confirmations, to work across A2A SDKs and agent frameworks. Extension-unaware clients can still participate by responding with a plain `DataPart` matching the schema.

## Specification

The full v1 specification can be found [here](./v1/spec.md).
