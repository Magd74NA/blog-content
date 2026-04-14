---
title: "Building dilates-education.com: a multi-tenant Pilates LMS in Phoenix and Ash"
date: "2026-04-09"
updated: "2026-04-09"
categories:
  - "elixir"
  - "phoenix"
  - "ash"
  - "projects"
excerpt: "Notes on architecting a multi-tenant learning management system for a Pilates studio in Elixir, Phoenix LiveView, and the Ash framework, including why Ash was the right call and where it pushed back."
coverImage: "/images/posts/dimalates-logo.png"
coverWidth: 500
coverHeight: 500
---

[dilates-education.com](https://dilates-education.com) (named "Dimalates" internally) is a learning management system I architected and built for a Pilates studio under MKA Cloud Studio, my consultancy. It hosts courses, lessons, and student enrollments, lets instructors run live chat with students, and supports multiple studios on a single deployment with proper data isolation.

This post is a tour of the stack and the architectural decisions that paid off.

## The stack at a glance

- **Elixir 1.19 / Phoenix 1.8 / Phoenix LiveView 1.1**
- **[Ash Framework 3.x](https://ash-hq.org/)** for resources, authorization, and code interfaces
- **PostgreSQL 16** via `ash_postgres` and `ecto_sql`
- **AshAuthentication** with password + Google OAuth
- **OpenTelemetry** + Sentry for observability
- **Tailwind + daisyUI** for styling: no custom CSS, no inline styles
- **Bandit** as the HTTP server
- **Fly.io** for deployment

No JavaScript framework on the frontend. LiveView handles every interactive surface, including the live chat between instructors and students.

## Why Ash

Phoenix has an enviable default stack (Ecto, contexts, LiveView) and for many apps that's enough. I picked Ash because the LMS has a few characteristics that punish the Phoenix-default approach:

1. **A lot of related resources with overlapping authorization rules.** Users belong to tenants. Tenants own courses. Courses contain lessons. Users enroll in courses, and their access depends on enrollment status, role within the tenant, and whether the lesson is published.
2. **Multiple read/write surfaces.** LiveView, JSON:API for an upcoming mobile companion, an admin UI, and a TypeScript RPC layer for some interactive components, all needing to share the same authorization rules and validations.
3. **A real preference for declarative business logic** that I can read in one place rather than scattered across context modules, controllers, and LiveView mounts.

Ash gives you all of that. A resource declares its attributes, relationships, actions, validations, changes, and policies in one file, and every entry point (LiveView, JSON API, Admin UI, code interface) goes through the same actions and the same policies.

## Domain-driven structure

The application is organized into three Ash domains:

```text
lib/dimalates/
├── accounts/        # User, Tenant, UserAccount, UserRole, Token
├── content/         # Course, Lesson, RenderedLesson
└── learning/        # Enrollment
```

**Accounts** owns identity and authorization. `User` is the global identity (one per email). `Tenant` represents a studio. `UserAccount` is the join table that lets a single user belong to multiple tenants with different roles in each, instructor in one studio, student in another. `Token` is managed by AshAuthentication.

**Content** owns the educational material. `Course` belongs to a tenant. `Lesson` belongs to a course. `RenderedLesson` is a cache of pre-rendered lesson HTML, invalidated by an Ash `change` hook when the underlying lesson is updated:

```elixir
# lib/dimalates/content/lesson/changes/invalidate_rendered_cache.ex
defmodule Dimalates.Content.Lesson.Changes.InvalidateRenderedCache do
  use Ash.Resource.Change
  # ... after_action hook that drops the cached render
end
```

**Learning** owns the relationship between users and content. `Enrollment` is the resource that says "user X is enrolled in course Y, and they're in state Z". `EnrollmentStatus` is an Ash enum (`pending | active | completed | cancelled`).

The domains import each other only through code interfaces. The web layer never calls `Ash.read!/2` or `Ash.get!/2` directly. It always goes through the domain function:

```elixir
# Good
Dimalates.Accounts.get_user_by_id!(id, load: [:tenants, :roles])

# Not in this codebase
Ash.get!(Dimalates.Accounts.User, id)
```

This is enforced by code review and by the Ash usage rules included in `CLAUDE.md`. The payoff is that domain functions are the *only* way into the data, which means policies can't be accidentally bypassed.

## Multi-tenancy via the attribute strategy

Ash supports two multi-tenancy strategies: schema-based (separate Postgres schemas per tenant) and attribute-based (a `tenant_id` column on every tenant-aware resource, automatically scoped on every query). I picked attribute-based.

Why:

- **One database, one connection pool.** Schema-per-tenant is great if your tenants are large and want hard isolation, but it complicates connection management and pgbouncer config.
- **Cheap onboarding.** New tenant = new row in `tenants`, no DDL.
- **Easier reporting** across tenants when needed (with explicit opt-out of the tenant scope).

The risk of attribute-based multi-tenancy is the classic one: forget the `tenant_id` filter on a query and you leak data between tenants. Ash handles this for you. Once a resource declares `multitenancy strategy: :attribute, attribute: :tenant_id`, every action requires a tenant context, and unscoped reads error out at the framework level. You can't accidentally write a leaky query because the framework refuses to execute it.

The tenant context is set in a LiveView mount hook (`DimalatesWeb.LiveUserAuth`) and in a JSON API plug, both of which read the active tenant off the user session.

## Authentication

Two flows: password and Google OAuth, both via AshAuthentication.

Password registration sends a confirmation email through Swoosh + the configured mailer adapter. The user clicks the link, the token is verified, and `confirmed_at` is stamped on the user.

Google OAuth was the more interesting integration. The redirect URI in Google Cloud Console has to match *exactly*, and I wanted dev, test, and prod to all work without manual config edits. The solution is:

- **Dev/test:** read `GOOGLE_REDIRECT_URI` from `envs/.dev.env` / `envs/.test.env` (Dotenvy), and require it to match what's registered in Google Cloud Console.
- **Prod:** never set `GOOGLE_REDIRECT_URI`. Instead, derive it from `PHX_HOST` at runtime in `Dimalates.GoogleOAuthConfig`. Production redirect URI is always `https://{PHX_HOST}/auth/user/google/callback`.

That way prod can't drift from the registered URI by accident, and dev can have its own.

When a brand-new user signs in with Google, a custom Ash `change` (`CreateOAuthUserAccount`) provisions their `User` and `UserAccount` rows in the same transaction as the OAuth identity creation, so the user lands on the dashboard already attached to a tenant.

## Authorization, the Ash way

Every resource has a `policies` block. A few examples of how that reads:

```elixir
# Pseudocode - actual policies live in each resource module.
# A user can read a lesson if they're enrolled in its course and the lesson is published.
policy action_type(:read) do
  authorize_if Dimalates.Learning.Checks.IsEnrolledInCourse
  authorize_if actor_attribute_equals(:role, :admin)
end
```

`Dimalates.Learning.Checks.IsEnrolledInCourse` is a small Ash check module. It runs on every read and write attempt that goes through the resource. There is no authorization logic in the LiveViews. There is none in the controllers. The web layer is "fetch the actor, fetch the tenant, call the domain function, render."

This is the part of Ash I'd most miss going back to plain Phoenix. The framework drags you toward putting auth at the resource level, which is the only place it can be checked consistently.

## Observability

The app exports OpenTelemetry traces from Phoenix, Bandit, Ecto, and Ash via the corresponding `opentelemetry_*` packages, and reports errors to Sentry with `sentry_before_send` filtering out the predictable noise. Every request gets a trace; every Ash action shows up as a span; every Ecto query is a child span underneath. When a slow request shows up in Sentry, the linked trace usually tells me exactly which Ash action and which query.

`logger_json` formats logs as JSON for Fly's log shipper, and `Dimalates.AuditLogNotifier` writes a separate audit trail for sensitive actions (role changes, tenant-level config edits) so admins can see who did what.

## Closing thought

Ash gets a reputation for being heavyweight, and on the surface it is: a lesson resource is 200 lines of declarative DSL. But the alternative isn't 50 lines of Ecto schema; it's 50 lines of schema, *plus* 200 lines of context module, *plus* authorization checks scattered across LiveViews and controllers, *plus* a JSON API serializer module, *plus* duplicate validation logic for the admin UI. Ash collapses all of that into one place, and the resulting code is the easiest Elixir codebase I've ever come back to after a month away.

If you're considering Ash for a multi-tenant SaaS with non-trivial authorization, my recommendation is: try it. The learning curve is real, but so is the payoff.
