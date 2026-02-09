# Quarkus Integration Reference — Timefold Solver

> How to set up, configure, and run Timefold Solver in a Quarkus application.
> Covers Maven setup, application.properties, REST endpoint patterns, and testing.

## Table of Contents

1. [Maven Dependencies](#1-maven-dependencies)
2. [application.properties](#2-applicationproperties)
3. [Auto-Discovery (Zero XML)](#3-auto-discovery-zero-xml)
4. [Injectable Beans](#4-injectable-beans)
5. [REST Endpoint Pattern](#5-rest-endpoint-pattern)
6. [SolverManager API](#6-solvermanager-api)
7. [SolutionManager / ScoreManager](#7-solutionmanager--scoremanager)
8. [Testing](#8-testing)
9. [Logging](#9-logging)
10. [JSON Serialization (Jackson)](#10-json-serialization-jackson)
11. [Native Build Considerations](#11-native-build-considerations)
12. [Complete Project Skeleton](#12-complete-project-skeleton)

---

## 1. Maven Dependencies

### Minimal (Quarkus + Timefold)

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>ai.timefold.solver</groupId>
            <artifactId>timefold-solver-bom</artifactId>
            <version>1.28.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Timefold Quarkus extension (includes solver core) -->
    <dependency>
        <groupId>ai.timefold.solver</groupId>
        <artifactId>timefold-solver-quarkus</artifactId>
    </dependency>

    <!-- JSON support for REST endpoints -->
    <dependency>
        <groupId>ai.timefold.solver</groupId>
        <artifactId>timefold-solver-quarkus-jackson</artifactId>
    </dependency>

    <!-- REST framework -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>

    <!-- Test dependencies -->
    <dependency>
        <groupId>ai.timefold.solver</groupId>
        <artifactId>timefold-solver-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-junit5</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Key points:**
- Use the BOM to ensure version alignment across all Timefold artifacts.
- `timefold-solver-quarkus` is the extension — NOT `timefold-solver-core` (that's for non-Quarkus).
- `timefold-solver-quarkus-jackson` enables proper JSON serialization of `HardSoftScore` and other Timefold types.
- **No solverConfig.xml is needed** — Quarkus auto-discovers everything.

---

## 2. application.properties

### Essential properties

```properties
# How long the solver runs (required — no default termination!)
quarkus.timefold.solver.termination.spent-limit=5m

# Alternative: stop after no improvement for N seconds
# quarkus.timefold.solver.termination.unimproved-spent-limit=30s

# Alternative: stop when score reaches this threshold
# quarkus.timefold.solver.termination.best-score-limit=0hard/*soft
```

### Common optional properties

```properties
# Environment mode: REPRODUCIBLE (default), FAST_ASSERT, STEP_ASSERT, FULL_ASSERT
# Use FULL_ASSERT during development to catch score corruption
quarkus.timefold.solver.environment-mode=FULL_ASSERT

# Domain access type: REFLECTION (default) or GIZMO (faster, Quarkus default)
# quarkus.timefold.solver.domain-access-type=GIZMO

# Multi-threaded solving (Enterprise Edition only)
# quarkus.timefold.solver.move-thread-count=AUTO

# Automatic node sharing for constraint streams (Enterprise Edition)
# quarkus.timefold.solver.constraint-stream-automatic-node-sharing=true
```

### Logging properties

```properties
# Show solver progress (recommended during development)
quarkus.log.category."ai.timefold.solver".level=INFO

# Detailed step-by-step logging
# quarkus.log.category."ai.timefold.solver".level=DEBUG

# Show every move (very verbose)
# quarkus.log.category."ai.timefold.solver".level=TRACE
```

**Critical:** You MUST set a termination condition. Without one, the solver runs forever.

---

## 3. Auto-Discovery (Zero XML)

With the Quarkus extension, Timefold automatically discovers:

| What | How |
|---|---|
| `@PlanningSolution` class | Classpath scan |
| `@PlanningEntity` classes | Classpath scan |
| `ConstraintProvider` implementation | Classpath scan (exactly one) |
| Planning variables, value ranges | Annotations on entity/solution |

**No `solverConfig.xml` is needed.** The only required config is the termination condition in `application.properties`.

If you need XML config (advanced), set:
```properties
quarkus.timefold.solver.solver-config-xml=solverConfig.xml
```

---

## 4. Injectable Beans

Quarkus auto-creates and manages these beans. Inject with `@Inject`:

```java
import jakarta.inject.Inject;

@Inject
SolverManager<EmployeeSchedule, String> solverManager;

@Inject
SolutionManager<EmployeeSchedule, HardSoftScore> solutionManager;

@Inject
SolverFactory<EmployeeSchedule> solverFactory;     // rarely needed directly

@Inject  // in test classes
ConstraintVerifier<EmployeeSchedulingConstraintProvider, EmployeeSchedule>
        constraintVerifier;
```

| Bean | Purpose | When to use |
|---|---|---|
| `SolverManager<Solution, ProblemId>` | Async solving, manages thread pool | **Always use this in REST endpoints** |
| `SolutionManager<Solution, Score>` | Score analysis, explanation | Score breakdown, constraint analysis |
| `SolverFactory<Solution>` | Create Solver instances manually | Rarely — only for custom thread management |
| `ConstraintVerifier<Provider, Solution>` | Unit test constraints | Test classes only |

---

## 5. REST Endpoint Pattern

### Simple synchronous (blocks until solved)

```java
@Path("/schedule")
public class ScheduleResource {

    @Inject
    SolverManager<EmployeeSchedule, String> solverManager;

    @POST
    @Path("/solve")
    public EmployeeSchedule solve(EmployeeSchedule problem) {
        String problemId = UUID.randomUUID().toString();
        SolverJob<EmployeeSchedule, String> solverJob =
                solverManager.solve(problemId, problem);
        try {
            return solverJob.getFinalBestSolution();
        } catch (InterruptedException | ExecutionException e) {
            throw new IllegalStateException("Solving failed.", e);
        }
    }
}
```

### Async (recommended for production)

```java
@Path("/schedule")
@ApplicationScoped
public class ScheduleResource {

    @Inject
    SolverManager<EmployeeSchedule, String> solverManager;

    @Inject
    SolutionManager<EmployeeSchedule, HardSoftScore> solutionManager;

    // In-memory store (replace with database in production)
    private final ConcurrentHashMap<String, EmployeeSchedule> solutionMap =
            new ConcurrentHashMap<>();

    @POST
    @Path("/solve")
    public String solve(EmployeeSchedule problem) {
        String jobId = UUID.randomUUID().toString();
        solutionMap.put(jobId, problem);

        solverManager.solveBuilder()
                .withProblemId(jobId)
                .withProblem(problem)
                .withBestSolutionConsumer(solution ->
                        solutionMap.put(jobId, solution))
                .run();
        return jobId;
    }

    @GET
    @Path("/solution/{jobId}")
    public EmployeeSchedule getSolution(@PathParam("jobId") String jobId) {
        return solutionMap.get(jobId);
    }

    @GET
    @Path("/status/{jobId}")
    public SolverStatus getStatus(@PathParam("jobId") String jobId) {
        return solverManager.getSolverStatus(jobId);
    }

    @DELETE
    @Path("/solve/{jobId}")
    public void stopSolving(@PathParam("jobId") String jobId) {
        solverManager.terminateEarly(jobId);
    }
}
```

**Key points:**
- `solverManager.solve()` returns immediately — solving happens in a background thread.
- `withBestSolutionConsumer` gets called whenever a better solution is found.
- `getSolverStatus()` returns `NOT_SOLVING`, `SOLVING_ACTIVE`, or `SOLVING_SCHEDULED`.
- `terminateEarly()` stops the solver gracefully and triggers the final best solution.

---

## 6. SolverManager API

```java
// Simple solve (returns SolverJob)
SolverJob<Solution, ProblemId> job = solverManager.solve(problemId, problem);
Solution best = job.getFinalBestSolution();  // blocks until done

// Builder pattern (recommended)
solverManager.solveBuilder()
    .withProblemId(jobId)
    .withProblem(problem)
    .withBestSolutionConsumer(this::saveSolution)    // called on improvement
    .withExceptionHandler(this::handleError)         // called on failure
    .run();

// Check status
SolverStatus status = solverManager.getSolverStatus(problemId);

// Stop early
solverManager.terminateEarly(problemId);
```

The problem ID (`String`, `Long`, `UUID`) uniquely identifies a solving job. Use it for all subsequent operations.

---

## 7. SolutionManager / ScoreManager

Use `SolutionManager` to analyze scores without solving:

```java
@Inject
SolutionManager<EmployeeSchedule, HardSoftScore> solutionManager;

// Get score analysis (which constraints are broken and by how much)
ScoreAnalysis<HardSoftScore> analysis = solutionManager.analyze(solution);

// Print summary
analysis.constraintMap().forEach((constraintRef, constraintAnalysis) -> {
    System.out.println(constraintRef.constraintName()
            + ": " + constraintAnalysis.score());
});
```

This is useful for:
- Showing a score breakdown in the UI
- Debugging why a particular solution has a certain score
- Comparing two solutions

---

## 8. Testing

### Constraint tests (unit tests)

```java
@QuarkusTest
class EmployeeSchedulingConstraintProviderTest {

    @Inject
    ConstraintVerifier<EmployeeSchedulingConstraintProvider, EmployeeSchedule>
            constraintVerifier;

    @Test
    void requiredSkill() {
        Employee ann = new Employee("1", "Ann", Set.of("Waiter"));
        constraintVerifier
            .verifyThat(EmployeeSchedulingConstraintProvider::requiredSkill)
            .given(new Shift("s1", MONDAY_6AM, MONDAY_2PM, "Cook", ann))
            .penalizesBy(1);
    }
}
```

### Integration test (full solve)

```java
@QuarkusTest
class ScheduleResourceTest {

    @Test
    void solveAndVerifyScore() {
        EmployeeSchedule problem = generateTestProblem();

        EmployeeSchedule solution = given()
            .contentType(ContentType.JSON)
            .body(problem)
            .when().post("/schedule/solve")
            .then().statusCode(200)
            .extract().as(EmployeeSchedule.class);

        assertNotNull(solution.getScore());
        assertTrue(solution.getScore().isFeasible());
    }
}
```

**Tip:** For integration tests, set a short termination time:
```properties
%test.quarkus.timefold.solver.termination.spent-limit=5s
```

---

## 9. Logging

| Level | application.properties | What you see |
|---|---|---|
| INFO | `quarkus.log.category."ai.timefold.solver".level=INFO` | Start/end, best score |
| DEBUG | `...level=DEBUG` | Every step, score changes |
| TRACE | `...level=TRACE` | Every move evaluated |

**Recommended:** Use `INFO` in production, `DEBUG` during development, `TRACE` only for deep debugging.

Log output example at DEBUG:
```
Solving started: time spent (67), best score (0hard/0soft), environment mode (REPRODUCIBLE)
CH step (0), time spent (128), score (-1hard/0soft), selected move count (15), picked move (...)
LS step (112), time spent (5023), score (0hard/-3soft), new best score (0hard/-3soft)
Solving ended: time spent (300000), best score (0hard/-2soft), score calc speed (45000/sec)
```

---

## 10. JSON Serialization (Jackson)

The `timefold-solver-quarkus-jackson` dependency auto-registers serializers for:
- `HardSoftScore`, `HardMediumSoftScore`, `SimpleScore`, `BendableScore`, etc.
- `SolverStatus`

A `HardSoftScore` serializes as:
```json
{
  "score": "0hard/-5soft"
}
```

**Important:** If `@PlanningScore` is on a field called `score`, Jackson will serialize it automatically. Make sure your `@PlanningSolution` class has a getter and setter for the score field.

If using the REST endpoint to accept problems as JSON, your `@PlanningSolution` must have a no-arg constructor, and all domain objects need no-arg constructors (Jackson requirement, same as Timefold requirement).

---

## 11. Native Build Considerations

Timefold works with Quarkus native builds (`-Dnative`):

```bash
./mvnw package -Dnative
```

Key notes:
- The default `GIZMO` domain access type is required for native builds (no reflection).
- All domain classes must be registered for reflection or use Gizmo.
- The Quarkus extension handles this automatically for annotated classes.
- Custom `VariableListener` or `Comparator` classes used in annotations must be accessible at build time.

---

## 12. Complete Project Skeleton

### Directory structure

```
src/main/java/com/example/scheduling/
├── domain/
│   ├── Employee.java            // Problem fact
│   ├── Availability.java        // Problem fact
│   ├── AvailabilityType.java    // Enum
│   ├── Shift.java               // @PlanningEntity
│   └── EmployeeSchedule.java    // @PlanningSolution
├── solver/
│   └── EmployeeSchedulingConstraintProvider.java
└── rest/
    └── ScheduleResource.java

src/main/resources/
└── application.properties

src/test/java/com/example/scheduling/solver/
└── EmployeeSchedulingConstraintProviderTest.java
```

### application.properties (complete)

```properties
# Solver termination
quarkus.timefold.solver.termination.spent-limit=5m

# Dev/test override (shorter for faster feedback)
%dev.quarkus.timefold.solver.termination.spent-limit=30s
%test.quarkus.timefold.solver.termination.spent-limit=5s

# Logging
quarkus.log.category."ai.timefold.solver".level=INFO
%dev.quarkus.log.category."ai.timefold.solver".level=DEBUG

# REST
quarkus.http.port=8080

# CORS (if frontend on different origin)
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000
```
