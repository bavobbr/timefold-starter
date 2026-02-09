# CLAUDE.md — Timefold Solver Scheduling & Rostering (Java/Quarkus)

> You are helping build scheduling and rostering applications using **Timefold Solver**
> (the open-source continuation of OptaPlanner) with **Quarkus** and **Java 17+**.
> The examples and reference docs cover employee scheduling, timetabling, crew scheduling,
> conference/meeting scheduling, bed allocation, task assignment, and tournament scheduling.
>
> This file is the entry point. Read the relevant reference docs below before writing code.
> When unsure about a pattern, `cat` the ground truth examples in `references/examples/` to verify.

---

## Quick Start

When the user asks you to create a new project from scratch:

1. **First**, `cat` the employee scheduling example's `pom.xml` and `application.properties` to see the real structure:
   - `cat references/examples/employee-scheduling/pom.xml`
   - `cat references/examples/employee-scheduling/src/main/resources/application.properties`
2. Copy `references/docs/pom-template.xml` to `pom.xml`.
3. Replace `{{GROUP_ID}}`, `{{ARTIFACT_ID}}`, `{{PACKAGE}}` with user's values (or sensible defaults like `com.example` / `employee-scheduling`).
4. Create `src/main/resources/application.properties` with at minimum:
   ```properties
   quarkus.timefold.solver.termination.spent-limit=5m
   %dev.quarkus.timefold.solver.termination.spent-limit=30s
   %test.quarkus.timefold.solver.termination.spent-limit=5s
   quarkus.log.category."ai.timefold.solver".level=INFO
   ```
5. Create domain classes following the modeling guide (and `cat` the example domain classes to verify patterns).
6. Create the ConstraintProvider following the constraint streams guide.
7. Create a REST endpoint following the Quarkus integration guide.
8. Create constraint unit tests.

---

## Constraint Design (Initial Modeling Phase)

During the initial domain and constraint design phase — before writing code — act as a co-designer, not just an implementer. Help the user think through their constraints systematically.

### What to do

1. **When the user describes their scheduling problem**, review the constraint catalog (`references/docs/constraint-catalog.md`) and suggest constraints they likely need but haven't mentioned. Common ones that are easy to overlook:
   - No overlapping shifts/assignments for the same person
   - Required skills or qualifications
   - Minimum rest time between consecutive shifts (often a legal requirement)
   - Maximum shifts per day/week/period
   - Availability or unavailability windows
   - Fairness / workload balancing

2. **Categorize constraints with the user** before implementing. Walk through each proposed constraint and agree on:
   - **Hard** — must never be violated (e.g., no overlapping shifts, required skills)
   - **Medium** — used for overconstrained problems (e.g., assign as many shifts as possible)
   - **Soft** — preferences to optimize (e.g., preferred shifts, balanced workload)

3. **Warn if a clearly critical constraint is missing.** If the user's design has no overlap prevention, no skill matching, or no termination condition — flag it explicitly. These are almost always bugs, not intentional omissions.

4. **Point to the relevant example** for each suggested constraint so the user can see a working implementation. Use the "Which example to consult" and "Common Use Case Quick Reference" tables below.

### When NOT to do this

- When the user asks to add a specific, well-defined constraint — just implement it.
- When the user is debugging or modifying existing constraints — focus on the issue at hand.
- When the user explicitly says they want a minimal setup and will add constraints later.

---

## Reference Documentation

Read these BEFORE writing code. They are in `references/docs/`:

| File | When to read | Lines |
|---|---|---|
| `modeling-guide.md` | Creating/modifying domain classes (`@PlanningEntity`, `@PlanningSolution`, etc.) | 653 |
| `constraint-streams-guide.md` | Writing or modifying constraints (ConstraintStream API) | 624 |
| `constraint-catalog.md` | Need a specific constraint (copy-paste ready implementations) | 502 |
| `quarkus-integration.md` | Maven setup, application.properties, REST endpoints, testing | 469 |
| `design-patterns.md` | Architectural decisions (overconstrained, pinning, many-to-many) | 291 |
| `continuous-planning.md` | Continuous/real-time planning, ProblemChange, replanning | 445 |
| `pom-template.xml` | Starting a new project | — |
| `references/examples/employee-scheduling/` | Verify any pattern against working code (primary ground truth) | — |
| `references/examples/school-timetabling/` | Simpler reference for basic patterns, multi-variable entities | — |
| `references/examples/flight-crew-scheduling/` | Rest time constraints, home airport, date-spanning shifts | — |
| `references/examples/conference-scheduling/` | Multi-variable, speaker conflicts, preference constraints | — |
| `references/examples/meeting-scheduling/` | Attendee conflicts, time grain modeling | — |
| `references/examples/bed-allocation/` | Overconstrained planning, date-range overlaps | — |
| `references/examples/task-assigning/` | Skill matching, priority weights, affinity | — |
| `references/examples/tournament-scheduling/` | Sports scheduling, load-balanced fairness, overconstrained | — |

**Read order for a new project:** `modeling-guide.md` → `quarkus-integration.md` → `constraint-streams-guide.md`

**Read order for adding constraints:** `constraint-catalog.md` → `constraint-streams-guide.md`

**Read order for advanced patterns:** `design-patterns.md` → `continuous-planning.md`

---

## Ground Truth Examples

Eight complete working quickstarts are in `references/examples/`. **When unsure about annotation placement, domain modeling, constraint syntax, or REST wiring — `cat` the relevant file below before generating code.** These are canonical Timefold patterns.

### Employee Scheduling (primary reference)

Full Quarkus app with multiple constraint types, availability handling, and REST API. **Start here for any employee-to-shift pattern.**

| What | Path |
|---|---|
| Planning entity | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/domain/Shift.java` |
| Problem facts | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/domain/Employee.java` |
| Availability fact | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/domain/Availability.java` |
| Planning solution | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/domain/EmployeeSchedule.java` |
| ConstraintProvider | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/solver/EmployeeSchedulingConstraintProvider.java` |
| REST endpoint | `references/examples/employee-scheduling/src/main/java/org/acme/employeescheduling/rest/EmployeeScheduleResource.java` |
| Constraint tests | `references/examples/employee-scheduling/src/test/java/org/acme/employeescheduling/solver/EmployeeSchedulingConstraintProviderTest.java` |
| application.properties | `references/examples/employee-scheduling/src/main/resources/application.properties` |
| pom.xml | `references/examples/employee-scheduling/pom.xml` |

### School Timetabling (simpler reference)

Simpler domain with fewer constraints — good for understanding the minimal pattern. Two planning variables (timeslot + room) on one entity.

| What | Path |
|---|---|
| Planning entity | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/domain/Lesson.java` |
| Problem facts | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/domain/Room.java` |
| Problem facts | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/domain/Timeslot.java` |
| Planning solution | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/domain/Timetable.java` |
| ConstraintProvider | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/solver/TimetableConstraintProvider.java` |
| REST endpoint | `references/examples/school-timetabling/src/main/java/org/acme/schooltimetabling/rest/TimetableResource.java` |
| Constraint tests | `references/examples/school-timetabling/src/test/java/org/acme/schooltimetabling/solver/TimetableConstraintProviderTest.java` |

### Flight Crew Scheduling

Crew-to-flight assignment with rest time between flights, home airport constraints, and date-spanning shifts. **Consult for rest/gap-between-shifts constraints.**

| What | Path |
|---|---|
| Planning entity | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/domain/FlightAssignment.java` |
| Problem facts | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/domain/Flight.java` |
| Problem facts | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/domain/Employee.java` |
| Planning solution | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/domain/FlightCrewSchedule.java` |
| ConstraintProvider | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/solver/FlightCrewSchedulingConstraintProvider.java` |
| REST endpoint | `references/examples/flight-crew-scheduling/flight-crew-scheduling/src/main/java/org/acme/flighcrewscheduling/rest/FlightCrewScheduleResource.java` |

### Conference Scheduling

Talks → timeslots + rooms with speaker conflicts, theme tracks, required/preferred timeslots. **Consult for multi-variable entities with complex preference constraints.**

| What | Path |
|---|---|
| Planning entity | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/domain/Talk.java` |
| Problem facts | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/domain/Room.java` |
| Problem facts | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/domain/Timeslot.java` |
| Problem facts | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/domain/Speaker.java` |
| Planning solution | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/domain/ConferenceSchedule.java` |
| ConstraintProvider | `references/examples/conference-scheduling/src/main/java/org/acme/conferencescheduling/solver/ConferenceSchedulingConstraintProvider.java` |

### Meeting Scheduling

Meetings → timeslots + rooms with required/preferred attendees and overlap prevention. **Consult for attendee-based conflict constraints.**

| What | Path |
|---|---|
| Planning entity | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/domain/MeetingAssignment.java` |
| Problem facts | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/domain/Meeting.java` |
| Problem facts | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/domain/Room.java` |
| Problem facts | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/domain/TimeGrain.java` |
| Planning solution | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/domain/MeetingSchedule.java` |
| ConstraintProvider | `references/examples/meeting-scheduling/src/main/java/org/acme/meetingschedule/solver/MeetingSchedulingConstraintProvider.java` |

### Bed Allocation

Patients → beds with admission/discharge date ranges, department specialisms, and gender constraints. **Consult for overconstrained planning and date-range overlaps.**

| What | Path |
|---|---|
| Planning entity | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/domain/Stay.java` |
| Problem facts | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/domain/Bed.java` |
| Problem facts | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/domain/Room.java` |
| Problem facts | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/domain/Department.java` |
| Planning solution | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/domain/BedPlan.java` |
| ConstraintProvider | `references/examples/bed-allocation/src/main/java/org/acme/bedallocation/solver/BedAllocationConstraintProvider.java` |

### Task Assigning

Employees → tasks with skill matching, priority weighting, and affinity. **Consult for weighted soft constraints and skill-based assignment.**

| What | Path |
|---|---|
| Planning entities | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/domain/Employee.java` (chain anchor) |
| Planning entities | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/domain/Task.java` (chained entity) |
| Problem facts | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/domain/Customer.java` |
| Problem facts | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/domain/TaskType.java` |
| Planning solution | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/domain/TaskAssigningSolution.java` |
| ConstraintProvider | `references/examples/task-assigning/src/main/java/org/acme/taskassigning/solver/TaskAssigningConstraintProvider.java` |

### Tournament Scheduling

Teams → days/slots with fairness constraints and unavailability windows. **Consult for sports scheduling, load-balanced fairness, and `HardMediumSoftBigDecimalScore`.**

| What | Path |
|---|---|
| Planning entity | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/domain/TeamAssignment.java` |
| Problem facts | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/domain/Team.java` |
| Problem facts | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/domain/Day.java` |
| Problem facts | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/domain/UnavailabilityPenalty.java` |
| Planning solution | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/domain/TournamentSchedule.java` |
| ConstraintProvider | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/solver/TournamentScheduleConstraintProvider.java` |
| REST endpoint | `references/examples/tournament-scheduling/src/main/java/org/acme/tournamentschedule/rest/TournamentSchedulingResource.java` |

### Which example to consult

| If you need to... | First check |
|---|---|
| Basic employee-to-shift assignment | `employee-scheduling` |
| Rest time / gap between consecutive shifts | `flight-crew-scheduling` |
| Multiple planning variables on one entity | `school-timetabling` or `conference-scheduling` |
| Speaker/attendee conflict prevention | `conference-scheduling` or `meeting-scheduling` |
| Date-range overlap constraints | `bed-allocation` |
| Overconstrained planning (not enough resources) | `bed-allocation` |
| Skill matching with priority weights | `task-assigning` |
| Required vs preferred (soft) preferences | `conference-scheduling` |
| Sports/tournament scheduling with fairness | `tournament-scheduling` |
| Minimal "hello world" pattern | `school-timetabling` |
| REST / SolverManager wiring | `employee-scheduling` (most complete) |
| Constraint test structure | `employee-scheduling` (has full test suite) |
| pom.xml / dependencies | `cat references/examples/employee-scheduling/pom.xml` |

---

## Core Concepts (Quick Reference)

### Architecture

```
┌─────────────────────────────────────────────────┐
│ @PlanningSolution (EmployeeSchedule)            │
│                                                  │
│   @ProblemFactCollectionProperty                │
│   List<Employee> employees;        ← value range │
│                                                  │
│   @PlanningEntityCollectionProperty              │
│   List<Shift> shifts;              ← entities    │
│                                                  │
│   @PlanningScore                                 │
│   HardSoftScore score;                           │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ @PlanningEntity (Shift)                          │
│                                                  │
│   @PlanningVariable                              │
│   Employee employee;    ← solver assigns this    │
│                                                  │
│   // Problem properties (fixed during solving):  │
│   LocalDateTime start, end;                      │
│   String requiredSkill;                          │
└─────────────────────────────────────────────────┘
```

### Key Annotations

| Annotation | Goes on | Purpose |
|---|---|---|
| `@PlanningSolution` | Solution class | Wraps the entire problem + solution |
| `@PlanningEntity` | Entity class (Shift) | Class the solver modifies |
| `@PlanningVariable` | Field on entity | The field the solver changes |
| `@PlanningScore` | Score field on solution | Where solver writes the score |
| `@PlanningId` | ID field on entities & facts | Required for testing & ProblemChange |
| `@ProblemFactCollectionProperty` | List fields on solution | Collections of fixed facts |
| `@PlanningEntityCollectionProperty` | List field on solution | Collection of planning entities |
| `@ValueRangeProvider` | List field on solution | Possible values for a planning variable |
| `@PlanningPin` | Boolean field on entity | If true, solver won't change this entity |

### Score Types

| Type | Levels | When to use |
|---|---|---|
| `HardSoftScore` | 2 | **Default.** Hard = must not violate. Soft = preferences. |
| `HardMediumSoftScore` | 3 | Overconstrained (medium = "assign as many as possible"). |
| `HardSoftBigDecimalScore` | 2 | Fairness/load balancing with `loadBalance()`. |

### Constraint Stream Pipeline

```
forEach(Shift.class)          // Select all assigned Shifts
  → filter(...)               // Keep only matching
  → join(OtherFact.class)     // Combine with another fact
  → groupBy(...)              // Aggregate (count, sum, etc.)
  → penalize(Score) / reward(Score)
  → asConstraint("name")      // Every constraint needs a unique name
```

### Quarkus Beans (auto-injected)

```java
@Inject SolverManager<EmployeeSchedule, String> solverManager;  // async solving
@Inject SolutionManager<EmployeeSchedule, HardSoftScore> solutionManager;  // score analysis
@Inject ConstraintVerifier<..., EmployeeSchedule> constraintVerifier;  // tests only
```

---

## Common Patterns

### Minimum Viable Domain Model

```java
// Employee.java — Problem fact
public class Employee {
    @PlanningId private String id;
    private String name;
    private Set<String> skills;
    // no-arg constructor + getters + setters
}

// Shift.java — Planning entity
@PlanningEntity
public class Shift {
    @PlanningId private String id;
    private LocalDateTime start;
    private LocalDateTime end;
    private String requiredSkill;

    @PlanningVariable
    private Employee employee;
    // no-arg constructor + getters + setters
}

// EmployeeSchedule.java — Planning solution
@PlanningSolution
public class EmployeeSchedule {
    @ProblemFactCollectionProperty
    @ValueRangeProvider
    private List<Employee> employees;

    @PlanningEntityCollectionProperty
    private List<Shift> shifts;

    @PlanningScore
    private HardSoftScore score;
    // no-arg constructor + getters + setters
}
```

### Minimum Viable ConstraintProvider

```java
public class EmployeeSchedulingConstraintProvider implements ConstraintProvider {
    @Override
    public Constraint[] defineConstraints(ConstraintFactory factory) {
        return new Constraint[] {
            requiredSkill(factory),
            noOverlappingShifts(factory),
        };
    }

    Constraint requiredSkill(ConstraintFactory factory) {
        return factory.forEach(Shift.class)
            .filter(shift -> !shift.getEmployee().getSkills()
                .contains(shift.getRequiredSkill()))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Required skill");
    }

    Constraint noOverlappingShifts(ConstraintFactory factory) {
        return factory.forEachUniquePair(Shift.class,
                Joiners.equal(Shift::getEmployee),
                Joiners.overlapping(Shift::getStart, Shift::getEnd))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("No overlapping shifts");
    }
}
```

### Minimum Viable REST Endpoint

```java
@Path("/schedule")
@ApplicationScoped
public class ScheduleResource {
    @Inject SolverManager<EmployeeSchedule, String> solverManager;

    @POST @Path("/solve")
    public EmployeeSchedule solve(EmployeeSchedule problem) {
        String id = UUID.randomUUID().toString();
        SolverJob<EmployeeSchedule, String> job = solverManager.solve(id, problem);
        try { return job.getFinalBestSolution(); }
        catch (InterruptedException | ExecutionException e) {
            throw new IllegalStateException("Solving failed.", e);
        }
    }
}
```

### Minimum Viable Constraint Test

```java
@QuarkusTest
class ConstraintProviderTest {
    @Inject ConstraintVerifier<EmployeeSchedulingConstraintProvider, EmployeeSchedule>
            constraintVerifier;

    @Test void requiredSkill() {
        Employee ann = new Employee("1", "Ann", Set.of("Waiter"));
        Shift shift = new Shift("s1", MONDAY_6AM, MONDAY_2PM, "Cook", ann);
        constraintVerifier
            .verifyThat(EmployeeSchedulingConstraintProvider::requiredSkill)
            .given(shift)
            .penalizesBy(1);
    }
}
```

---

## Critical Rules

1. **Every domain class needs a no-arg constructor** (required by both Timefold and Jackson).
2. **Every domain class needs `@PlanningId`** on a unique identifier field.
3. **`@ValueRangeProvider` and `@ProblemFactCollectionProperty` go on the SAME list** when that list provides values for a planning variable.
4. **Constraint names must be unique** across the entire `ConstraintProvider`.
5. **Always set a termination condition** in `application.properties` — without one, the solver runs forever.
6. **Use `SolverManager` for REST** — never call `solver.solve()` on a REST thread (causes HTTP timeouts).
7. **Use Joiners, not filters, for join conditions** — Joiners use hash indexes (fast), filters do linear scans (slow).
8. **`forEach()` excludes unassigned (null) entities by default.** Use `forEachIncludingUnassigned()` only when you need to check for unassigned entities.
9. **Use `forEachUniquePair()` for same-class pairs** — avoids duplicate `(A,B)` / `(B,A)` and self-pairs `(A,A)`.
10. **Never use `float`/`double` in scores** — they cause score corruption. Use `int`, `long`, or `BigDecimal`.

---

## Common Mistakes to Avoid

| Mistake | Fix |
|---|---|
| Forgetting no-arg constructor | Add `public Employee() {}` to every domain class |
| Missing `@PlanningId` | Add `@PlanningId` to a unique field on every entity and fact |
| `@ValueRangeProvider` on entity class | Put it on the `@PlanningSolution` class, not the entity |
| Using `join()` + `filter()` instead of `Joiners` | Move conditions into `Joiners.equal()`, `Joiners.overlapping()`, etc. |
| Duplicate constraint names | Each `asConstraint("...")` string must be globally unique |
| No termination configured | Add `quarkus.timefold.solver.termination.spent-limit=5m` |
| Calling `solver.solve()` on REST thread | Use `SolverManager.solve()` instead |
| Using `toList()`/`toSet()` collectors | These disable incremental calculation — use `count()`, `sum()` instead |
| Score with `float`/`double` | Use `HardSoftScore` (int), `HardSoftLongScore`, or `HardSoftBigDecimalScore` |
| Modifying problem facts during solving | Use `ProblemChange` interface — never modify directly |

---

## Import Cheat Sheet

```java
// Annotations
import ai.timefold.solver.core.api.domain.entity.PlanningEntity;
import ai.timefold.solver.core.api.domain.lookup.PlanningId;
import ai.timefold.solver.core.api.domain.solution.PlanningEntityCollectionProperty;
import ai.timefold.solver.core.api.domain.solution.PlanningScore;
import ai.timefold.solver.core.api.domain.solution.PlanningSolution;
import ai.timefold.solver.core.api.domain.solution.ProblemFactCollectionProperty;
import ai.timefold.solver.core.api.domain.variable.PlanningVariable;
import ai.timefold.solver.core.api.domain.entity.PlanningPin;
import ai.timefold.solver.core.api.domain.valuerange.ValueRangeProvider;

// Constraints
import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;
import ai.timefold.solver.core.api.score.stream.Constraint;
import ai.timefold.solver.core.api.score.stream.ConstraintFactory;
import ai.timefold.solver.core.api.score.stream.ConstraintProvider;
import ai.timefold.solver.core.api.score.stream.Joiners;
import ai.timefold.solver.core.api.score.stream.ConstraintCollectors;

// Solver (REST / service layer)
import ai.timefold.solver.core.api.solver.SolverManager;
import ai.timefold.solver.core.api.solver.SolverJob;
import ai.timefold.solver.core.api.solver.SolverStatus;
import ai.timefold.solver.core.api.solver.SolutionManager;

// Testing
import ai.timefold.solver.test.api.score.stream.ConstraintVerifier;
```

---

## Common Use Case Quick Reference

When you need to implement a specific feature, consult this table to find the right pattern and example:

| Want to... | Pattern to use | Example reference | Key API |
|---|---|---|---|
| Prevent overlapping shifts for same person | `forEachUniquePair()` + `Joiners.overlapping()` | `employee-scheduling` | `Joiners.overlapping(startFunction, endFunction)` |
| Ensure minimum gap between consecutive shifts | `forEachUniquePair()` + custom time comparison | `flight-crew-scheduling` | Filter with time arithmetic |
| Limit max shifts per person per day/week | `groupBy()` + `count()` | `employee-scheduling` | `ConstraintCollectors.count()` |
| Balance workload fairly across employees | `groupBy()` + `loadBalance()` | Use `HardSoftBigDecimalScore` | `ConstraintCollectors.loadBalance()` |
| Match required skills | `filter()` on skill sets | `employee-scheduling`, `task-assigning` | `!employee.getSkills().contains(required)` |
| Prefer certain assignments (soft preference) | Soft constraint with `reward()` | `conference-scheduling` | `.reward(HardSoftScore.ONE_SOFT)` |
| Assign consecutive tasks in sequence | Chained planning variables | `task-assigning` | `@PlanningVariable(graphType = CHAINED)` |
| Handle availability windows | `filter()` with time range check | `employee-scheduling` | Check against `Availability` facts |
| Prevent double-booking resources (rooms) | `forEachUniquePair()` + `Joiners.equal(resource)` | `school-timetabling`, `meeting-scheduling` | `Joiners.equal()` + `Joiners.overlapping()` |
| Assign multiple variables (time + room) | Multiple `@PlanningVariable` on one entity | `school-timetabling`, `conference-scheduling` | Two `@PlanningVariable` fields |
| Handle speaker/attendee conflicts | `join()` with person + `Joiners.overlapping()` | `conference-scheduling`, `meeting-scheduling` | Join talks by speaker |
| Limit field/resource capacity | `groupBy()` + `sum()` of sizes | `tournament-scheduling` | `ConstraintCollectors.sum()` |
| Date-range overlaps (multi-day stays) | `forEachUniquePair()` + date range check | `bed-allocation` | Custom date overlap logic |
| Overconstrained (not enough resources) | `HardMediumSoftScore` + medium penalties | `bed-allocation` | Medium = "assign as many as possible" |
| Home location preferences | Soft constraint rewarding proximity | `flight-crew-scheduling` | `.reward()` based on distance |
| Fairness (equal distribution) | `loadBalance()` or `sum()` variance | Use `BigDecimalScore` | `ConstraintCollectors.loadBalance()` |

### How to use this table
1. Find your use case in the "Want to..." column
2. Check the example reference to see a working implementation
3. Use the Key API pattern as a starting point
4. `cat` the relevant file from the example to see the full constraint

---

## Development Workflow Commands

Essential commands for building, testing, and running Timefold applications.

### Starting a new project
```bash
# Run the example to see it work
cd references/examples/employee-scheduling
mvn quarkus:dev

# Access the UI (if available)
open http://localhost:8080
```

### Development mode
```bash
# Run with live reload (code changes apply automatically)
mvn quarkus:dev

# Run on a different port
mvn quarkus:dev -Dquarkus.http.port=8081

# Enable debug logging for solver
mvn quarkus:dev -Dquarkus.log.category."ai.timefold.solver".level=DEBUG
```

### Running tests
```bash
# Run all tests
mvn test

# Run a specific test class
mvn test -Dtest=EmployeeSchedulingConstraintProviderTest

# Run a specific test method
mvn test -Dtest=EmployeeSchedulingConstraintProviderTest#requiredSkill

# Run tests with solver debug output
mvn test -Dquarkus.log.category."ai.timefold.solver".level=DEBUG

# Skip tests during build
mvn package -DskipTests
```

### Building for production
```bash
# Package as a JAR
mvn package

# Run the packaged JAR
java -jar target/quarkus-app/quarkus-run.jar

# Build native executable (requires GraalVM)
mvn package -Dnative

# Run native executable
./target/*-runner
```

### Debugging solver behavior
```bash
# Enable detailed solver logging
# Add to application.properties:
quarkus.log.category."ai.timefold.solver".level=DEBUG

# Enable constraint match logging to see which constraints fire
quarkus.log.category."ai.timefold.solver.core.impl.score.stream".level=TRACE

# Check solver configuration
mvn quarkus:dev
# Then visit http://localhost:8080/q/dev (Quarkus Dev UI)
```

### Verifying setup
```bash
# Check Timefold version
mvn dependency:tree | grep timefold

# Check Quarkus version
mvn dependency:tree | grep quarkus-core

# List all dependencies
mvn dependency:list

# Validate pom.xml
mvn validate
```

### Working with examples
```bash
# Clone/update examples repository
cd references/examples

# Run specific example
cd employee-scheduling
mvn quarkus:dev

# Run tests for an example
mvn test

# Compare your code to the example
diff -u my-file.java references/examples/employee-scheduling/.../TheirFile.java
```

---

## Troubleshooting

Common errors and their solutions.

### Compilation / Build Errors

**Error:** `java: cannot find symbol` for Timefold classes
**Cause:** Missing dependency or wrong version
**Fix:** Check `pom.xml` has `quarkus-timefold-solver` dependency and that Timefold and Quarkus versions are compatible (check `references/docs/pom-template.xml` for current versions)

**Error:** `No planning entity found`
**Cause:** Missing `@PlanningEntity` annotation
**Fix:** Add `@PlanningEntity` to the class that the solver should modify

**Error:** `No @PlanningVariable found`
**Cause:** Missing `@PlanningVariable` annotation
**Fix:** Add `@PlanningVariable` to the field the solver should assign (e.g., `Employee employee`)

**Error:** `No @PlanningSolution found`
**Cause:** Missing `@PlanningSolution` annotation
**Fix:** Add `@PlanningSolution` to your solution wrapper class

**Error:** Type inference fails with `ConstraintCollectors.min()` / `max()`
**Cause:** Java can't infer generic types with complex lambda expressions
**Fix:** Use explicit type arguments: `ConstraintCollectors.<Entity, Type>min(lambda)`

### Runtime / Solver Errors

**Error:** `Score corruption detected`
**Cause:** Using `float` or `double` in score calculations
**Fix:** Use `int`, `long`, or `BigDecimal`. Never use floating-point for scores.

**Error:** `IllegalStateException: The entity ... was never added to this ScoreDirector`
**Cause:** Missing `@PlanningId` on entity or problem fact
**Fix:** Add `@PlanningId` to a unique identifier field on ALL domain classes

**Error:** Solver runs forever / never terminates
**Cause:** No termination condition configured
**Fix:** Add to `application.properties`:
```properties
quarkus.timefold.solver.termination.spent-limit=5m
```

**Error:** `UnsupportedOperationException` when modifying solution during solving
**Cause:** Trying to modify problem facts directly while solver is running
**Fix:** Use `ProblemChange` API to modify the solution safely

**Error:** HTTP timeout on `/solve` endpoint
**Cause:** Calling `solver.solve()` directly on REST thread (blocks HTTP thread)
**Fix:** Use `SolverManager.solve()` for async solving in REST endpoints

**Error:** NullPointerException in constraint
**Cause:** Planning variable is `null` (unassigned entity)
**Fix:** Either use `forEachIncludingUnassigned()` and null-check, or ensure all entities are assigned before scoring

### Constraint / Scoring Issues

**Problem:** Constraint never fires / score is always 0
**Possible causes:**
1. Using `forEach()` but entities are unassigned (`null`) → Use `forEachIncludingUnassigned()` and null-check
2. Filter condition is too restrictive → Add logging to check if filter matches any entities
3. Wrong constraint name or disabled in config → Check constraint is in `defineConstraints()` array
4. Joiners don't match → Verify `Joiners.equal()` fields are actually equal

**Debug approach:**
```java
// Add logging to constraint
.forEach(Shift.class)
  .filter(shift -> {
    boolean matches = shift.getEmployee() != null;
    System.out.println("Shift " + shift.getId() + " matched: " + matches);
    return matches;
  })
```

**Problem:** Constraint fires too many times
**Possible causes:**
1. Using `forEach()` twice instead of `forEachUniquePair()` → Creates duplicate pairs
2. Missing `Joiners` conditions → Joins create Cartesian product

**Problem:** Wrong score weight
**Possible causes:**
1. Using `penalize(ONE_HARD)` instead of `penalize(ONE_HARD, weightFunction)` → No per-match weight
2. Weight function returns wrong type → Verify function returns `int` or `long`

**Problem:** Slow performance / solving takes forever
**Possible causes:**
1. Using `filter()` instead of `Joiners` → Filters are O(n²), Joiners use hash indexes
2. Using `toList()` or `toSet()` collectors → Disables incremental calculation
3. Too many entities → Consider using construction heuristics or adjust termination
4. Complex constraint logic → Profile and simplify

**Debug with logging:**
```properties
# Enable constraint stream logging
quarkus.log.category."ai.timefold.solver.core.impl.score.stream".level=DEBUG

# Enable solver phase logging
quarkus.log.category."ai.timefold.solver".level=DEBUG
```

### REST API Issues

**Error:** CORS errors in browser (fetch fails)
**Cause:** CORS not enabled or misconfigured in Quarkus 3.4+
**Fix:** Add to `application.properties`:
```properties
quarkus.http.cors.enabled=true
quarkus.http.cors.origins=http://localhost:5173
```

**Problem:** Can't get solver status
**Cause:** Job ID not tracked or solver already finished
**Fix:** Store job ID returned by `SolverManager.solve()` and poll with `getSolverStatus(jobId)`

**Problem:** Solution not updating during solving
**Cause:** Not fetching intermediate solutions
**Fix:** Poll `getSchedule(jobId)` periodically while status is `SOLVING_ACTIVE`

### Testing Issues

**Error:** `ConstraintVerifier` not injected
**Cause:** Missing `@QuarkusTest` annotation or wrong test scope
**Fix:** Add `@QuarkusTest` to test class

**Problem:** Test passes but constraint doesn't work in practice
**Cause:** Test data doesn't reflect real scenario
**Fix:** Test with realistic data including edge cases (null values, empty lists, boundary conditions)

**Problem:** `verifyThat()` fails with unexpected score
**Cause:** Multiple constraints affecting the same entity
**Fix:** Test constraints in isolation using `verifyThat(ClassName::constraintMethod)`

### Common Configuration Issues

**Problem:** Different behavior in dev vs test vs prod
**Cause:** Profile-specific configuration overriding defaults
**Fix:** Check `%dev`, `%test`, `%prod` prefixes in `application.properties`

**Problem:** Solver uses wrong termination time
**Cause:** Profile override not set
**Fix:** Set for each profile:
```properties
quarkus.timefold.solver.termination.spent-limit=5m
%dev.quarkus.timefold.solver.termination.spent-limit=30s
%test.quarkus.timefold.solver.termination.spent-limit=5s
```

---

## External Resources

**Local (always check first):**
- `references/examples/` — 8 working quickstarts (ground truth): employee-scheduling, school-timetabling, flight-crew-scheduling, conference-scheduling, meeting-scheduling, bed-allocation, task-assigning, tournament-scheduling
- `references/docs/` — condensed reference guides

**Remote (if local sources don't cover it):**
- Official docs: https://docs.timefold.ai/timefold-solver/latest/
- Quickstarts repo: https://github.com/TimefoldAI/timefold-quickstarts
- DeepWiki (AI-indexed docs): https://deepwiki.com/TimefoldAI/timefold-solver
