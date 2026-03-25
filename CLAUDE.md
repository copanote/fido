# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **FIDO authentication knowledge repository and business planning project** written in Korean. It documents FIDO passwordless authentication technology and outlines a roadmap for expanding an existing UAF 1.1 FIDO server (built for Samsung Pay) into a B2B service.

There is no executable code — the repository is entirely Markdown documentation.

## Repository Structure

- **`/doc/`** — Reference material: FIDO definitions, standards (UAF, U2F, FIDO2, WebAuthn), and history
- **`/study/`** — 6-module educational curriculum covering crypto fundamentals through security threats
- **`/step1/`** — Business planning: current service capabilities and implementation roadmap for B2B expansion

## Content Navigation

- New to FIDO → start at `study/00-roadmap.md`, follow modules 01–06
- Current capabilities → `step1/01-service-list.md`
- What needs to be built → `step1/02-implementation-list.md`
- Server design reference → `study/04-server-architecture.md` (includes DB schema and API design)

## Current Project State

- UAF 1.1 FIDO server exists and is used by Samsung Pay
- No SDKs, portals, or FIDO2/WebAuthn server have been built yet
- Planned phases: Android/iOS SDKs → developer portal → FIDO2/WebAuthn support
