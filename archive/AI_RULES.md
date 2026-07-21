# AI_RULES.md

# AI Development Rules

These rules apply to every AI coding session.

## General

-   Never modify ERPNext core.
-   All customizations must stay inside the `mobile_shop` app.
-   Follow Frappe best practices.
-   Use Python for business logic.
-   Use JavaScript only for UI enhancements.

## Development Process

-   One feature per prompt.
-   Finish one feature before starting another.
-   Show changed files before explaining implementation.
-   Do not introduce unrelated refactoring.

## Architecture

-   Reuse ERPNext features whenever possible.
-   Prefer configuration over hardcoded values.
-   Never duplicate existing ERPNext functionality.

## Business Logic

-   All VAT calculations must run server-side.
-   Never trust client-side calculations.
-   Validate all user input.
-   Validate IMEI uniqueness before save.

## Database

-   Use DocTypes correctly.
-   Avoid unnecessary custom tables.
-   Keep naming consistent.

## UI

-   Mobile-first.
-   Responsive.
-   Minimal typing.
-   Large touch targets.

## Code Quality

-   Type hints where appropriate.
-   Clear comments for complex logic.
-   Keep functions small.
-   Reuse utilities.

## Git Workflow

For every completed feature:

1.  Review code.
2.  Run:
    -   bench migrate
    -   bench restart
3.  Test manually.
4.  Commit with a meaningful message.

## When Unsure

The AI should stop and ask instead of guessing requirements or
introducing new dependencies.
