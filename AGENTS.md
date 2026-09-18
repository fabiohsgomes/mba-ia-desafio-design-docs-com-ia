# Repository Guidelines

## Project Structure & Module Organization

This repository contains a Node.js 20, Express, TypeScript, Prisma, and MySQL order-management API that supports a design-documentation challenge. Application code lives in `src/`: domains are under `src/modules/<domain>/`, middleware in `src/middlewares/`, and common errors, logging, and HTTP helpers in `src/shared/`. Prisma files live in `prisma/`. Integration tests and helpers live in `tests/`.

## Build, Test, and Development Commands

- `npm ci`: install the locked dependency set.
- `docker compose up -d mysql`: start the local MySQL 8 service.
- `cp .env.example .env`: create local configuration; never commit `.env`.
- `npm run db:migrate && npm run db:seed`: prepare development data.
- `npm run dev`: run the API with watch mode.
- `npm test`: run the Vitest/Supertest integration suite once.
- `npm run lint`: check TypeScript with ESLint.
- `npm run format`: apply Prettier formatting.
- `npm run build`: type-check and compile into `dist/`.

## Coding Style & Naming Conventions

Use strict TypeScript, ES modules, two-space indentation, single quotes, semicolons, trailing commas, and a 100-character line width. Prefer type-only imports and avoid `any`. Follow existing domain filenames such as `order.service.ts`, `order.repository.ts`, and `order.schemas.ts`; use PascalCase for classes/types and camelCase for functions/variables. Name ADRs `ADR-NNN-kebab-case-title.md` and keep document links relative.

## Testing Guidelines

Vitest discovers `tests/**/*.test.ts`; Supertest exercises HTTP routes against a real test database. Write behavior-focused test names, reuse factories, and keep tests deterministic because files run serially. Run `npm test`, `npm run lint`, and `npm run build` before submission. No numeric coverage threshold is configured; cover success paths, authorization, validation, and failure cases for changed behavior.

## Commit & Pull Request Guidelines

Recent history uses short Conventional Commit-style subjects, especially `docs: ...`; use an imperative subject such as `docs: add webhook retry ADR`. Keep commits focused. Pull requests should summarize scope, identify sources, link issues, list validation commands, and call out unresolved assumptions. Include screenshots only for presentation changes.

## Documentation Integrity

Keep technical claims grounded in the implementation. When describing existing behavior, cite exact code paths and distinguish verified behavior from proposals or assumptions.
