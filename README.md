# Timefold Starter

A starter template for building scheduling and rostering applications with [Timefold Solver](https://timefold.ai/), Quarkus, and Java 17+. This project is designed to be used with [Claude Code](https://claude.ai/claude-code) as an AI-assisted development environment.

## What is this?

This repository provides Claude Code with the context it needs to generate correct, idiomatic Timefold Solver code. It contains:

- **8 complete working examples** covering common scheduling domains
- **Reference documentation** distilled from the official Timefold docs
- **A detailed `CLAUDE.md`** that instructs Claude Code on Timefold patterns, conventions, and pitfalls

When you ask Claude Code to build a scheduling application in this project, it will consult these references to produce code that follows Timefold best practices — correct annotations, proper constraint streams, appropriate score types, and working REST endpoints.

## What can you build with it?

Any constraint-based scheduling or rostering problem, including:

| Use case | Example reference |
|---|---|
| Employee shift scheduling | `employee-scheduling` |
| School/university timetabling | `school-timetabling` |
| Flight crew assignment | `flight-crew-scheduling` |
| Conference talk scheduling | `conference-scheduling` |
| Meeting room booking | `meeting-scheduling` |
| Hospital bed allocation | `bed-allocation` |
| Task assignment with skills | `task-assigning` |
| Sports tournament scheduling | `tournament-scheduling` |

These cover the most common patterns. You can combine and adapt them for other domains like vehicle routing, workforce planning, or resource allocation.

## Getting started

1. Open this project in Claude Code
2. Describe your scheduling problem — what needs to be assigned to what, and what rules apply
3. Claude Code will help you design constraints (hard vs. soft), then generate the domain model, constraint provider, REST API, and tests

Example prompt:

> Create a nurse rostering application. Nurses have skills (ICU, ER, General). Shifts need specific skills. No nurse should work two shifts in a row without 12 hours rest. Balance the workload fairly.

Claude Code will consult the reference examples and docs to produce a working Quarkus application.

## Project structure

```
timefold-starter/
  CLAUDE.md                     # Instructions for Claude Code
  references/
    docs/
      modeling-guide.md         # Domain class design
      constraint-streams-guide.md  # Writing constraints
      constraint-catalog.md     # Copy-paste constraint recipes
      quarkus-integration.md    # REST, Maven, testing setup
      design-patterns.md        # Overconstrained, pinning, many-to-many
      continuous-planning.md    # Real-time replanning
      pom-template.xml          # Maven template for new projects
    examples/
      employee-scheduling/      # Primary reference (most complete)
      school-timetabling/       # Simplest reference
      flight-crew-scheduling/   # Rest time, travel constraints
      conference-scheduling/    # Multi-variable, preferences
      meeting-scheduling/       # Attendee conflicts
      bed-allocation/           # Overconstrained planning
      task-assigning/           # Chained, skill matching
      tournament-scheduling/    # Fairness, load balancing
```

## How `CLAUDE.md` works

The `CLAUDE.md` file is automatically loaded by Claude Code when working in this project. It provides:

- **Step-by-step instructions** for creating new Timefold projects from scratch
- **Constraint design guidance** — Claude Code will suggest constraints you may have overlooked and help categorize them as hard, medium, or soft
- **A lookup table** mapping common scheduling needs to the right example and API pattern
- **Critical rules** like "every domain class needs a no-arg constructor" and "never use float/double in scores"
- **Troubleshooting** for common compilation, runtime, and constraint issues

You don't need to read or memorize `CLAUDE.md` yourself — it's written for Claude Code to follow.

## Prerequisites

- Java 17+
- Maven 3.8+
- Claude Code CLI

## Running the examples

You can run any example to see Timefold in action:

```bash
cd references/examples/employee-scheduling
mvn quarkus:dev
# Open http://localhost:8080
```

## Technology stack

- **[Timefold Solver](https://timefold.ai/)** — open-source constraint solver (successor to OptaPlanner)
- **[Quarkus](https://quarkus.io/)** — Java framework for cloud-native applications
- **Java 17+** — language runtime
