# ADR 0004: Scenario files as the primary specification

Status: proposed

## Context

We expect AI coding agents to write much of the code and many of the tests. Agent-written tests tend to be long, coupled to implementation, mock-heavy, and changed whenever code is refactored. That erodes trust in the suite. The ["Tests as data"](https://martinfowler.com/articles/tests-as-data.html) article describes a way to counter this: customer tests, black-box testing, tests as data, and a small test runner protected from casual change.

## Decision

- Most behavior is specified in scenario files (`specs/**/*.scenario.md`) and tables (`specs/**/*.table.csv`) in an external DSL with a strict parser.
- The DSL requires an `id`, a `covers` list, and at least one non-empty `THEN`. It rejects `skip`, `todo`, and `only`.
- Test code that is a better fit for code (pure domain rules, properties) stays in `test/unit/`.
- There are three runners (in-process, HTTP, oracle) reading the same files.
- CI checks which paths a PR touches against the PR's declared type.

Details are in [04-testing-strategy.md](../04-testing-strategy.md).

## Consequences

- We maintain a parser, printers, and runners. This is code to review and test, and it is guarded by CODEOWNERS and its own unit tests.
- Test review becomes reading short scenario diffs instead of long test code.
- The same scenarios can validate a future rewrite or a different deployment target.
- Cheating (an assertion-free test) is blocked by the parser rather than by reviewer attention.

## Risks

- The DSL could grow into a language. Mitigation: keep keywords few; add a printer or parser only when a scenario cannot be written clearly without it.
- Expected values could be wrong. Mitigation: oracle runner, mutation testing, property-based tests, human exploratory testing.

## Alternatives considered

- **Tests as code only (Vitest `it` blocks):** fastest to start, but the weaknesses above apply.
- **Cucumber or Gherkin:** familiar, but adds an interpretation layer that still lets assertion-free steps pass, and the article points out maintenance cost. A small custom DSL is cheaper now that agents can write the runner.
