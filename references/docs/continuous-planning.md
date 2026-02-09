# Continuous & Real-Time Planning Reference — Timefold Solver

> Patterns for planning that evolves over time: continuous planning windows,
> pinning, real-time problem changes, non-disruptive replanning,
> and the Assignment Recommendation API.
>
> This is the most advanced reference file. Read `design-patterns.md` first.

## Table of Contents

1. [Continuous Planning Overview](#1-continuous-planning-overview)
2. [Planning Window Stages](#2-planning-window-stages)
3. [Pinning Entities (Freezing the Past)](#3-pinning-entities)
4. [Implementing a Planning Window](#4-implementing-a-planning-window)
5. [Real-Time Planning with ProblemChange](#5-real-time-planning-with-problemchange)
6. [Non-Disruptive Replanning](#6-non-disruptive-replanning)
7. [Daemon Mode](#7-daemon-mode)
8. [Assignment Recommendation API](#8-assignment-recommendation-api)
9. [Backup Planning](#9-backup-planning)
10. [Complete Employee Scheduling Example](#10-complete-employee-scheduling-example)

---

## 1. Continuous Planning Overview

Continuous planning is replanning periodically (daily, weekly) with a rolling window
that covers the near future. You don't plan all time forever — you plan a window
and repeat.

**Why?** Because:
- Far-future problem facts are unreliable (employees might change availability).
- You need to publish schedules with advance notice (e.g., 2 weeks ahead).
- You must not change already-published or already-worked shifts.

**Typical cadence for employee scheduling:**
- Replan every week.
- Planning window: 4 weeks (1 published + 3 draft).
- Publish the first week of the draft.

---

## 2. Planning Window Stages

When you replan, the data divides into four stages:

| Stage | Description | Solver behavior |
|---|---|---|
| **History** | Past shifts, already worked. | **Pinned.** Include recent history only (affects constraints like "max consecutive days"). |
| **Published** | Upcoming shifts already shared with employees. | **Pinned** (or semi-movable with disruption penalty). |
| **Draft** | Upcoming shifts not yet shared. | **Movable.** Solver freely reassigns employees. |
| **Unplanned** | Future shifts beyond the planning window. | **Not loaded.** Not in the solution at all. |

```
   ◄─── History ───►◄── Published ──►◄───── Draft ─────►◄─ Unplanned ─►
   [===pinned===][===pinned===][========movable========]  [not loaded]
                                ▲
                        "publish line"
                        (moves forward each cycle)
```

**Key rule:** The draft must extend BEYOND the next publish cycle. Otherwise you risk
"painting yourself into a corner" — making the next publish infeasible because all
rare-skilled employees were used in the published period.

---

## 3. Pinning Entities

Pinned entities are frozen — the solver cannot change them.

### Using @PlanningPin

```java
@PlanningEntity
public class Shift {

    @PlanningPin
    private boolean pinned;

    @PlanningVariable
    private Employee employee;

    // ...
}
```

Set `pinned = true` for all history and published shifts before submitting to the solver.

### Pinning rules

- A pinned entity with a null variable and `allowsUnassigned = false` throws an exception (Timefold 1.10+).
- `@PlanningPin` value must NOT change during solving. Use `ProblemChange` to modify it.
- Pinned entities still affect constraint scores (e.g., a pinned Monday shift counts toward "max consecutive days" for a movable Tuesday shift).

### Loading history efficiently

Don't load ALL historical data — just enough to affect current constraints:
- "Max 5 consecutive days" → load last 5 days of history
- "Max 40 hours/week" → load current week's history
- "Max 2 consecutive weekends" → load last 2 weekends

Add a safety margin (e.g., load 2 extra days beyond what constraints need).

---

## 4. Implementing a Planning Window

### ScheduleState helper class

```java
public class ScheduleState {
    private LocalDate historyStart;    // oldest loaded history date
    private LocalDate publishStart;    // first published date
    private LocalDate draftStart;      // first draft date (= publish end)
    private LocalDate draftEnd;        // last planned date

    public boolean isHistory(Shift shift) {
        return shift.getDate().isBefore(publishStart);
    }

    public boolean isPublished(Shift shift) {
        return !shift.getDate().isBefore(publishStart)
            && shift.getDate().isBefore(draftStart);
    }

    public boolean isDraft(Shift shift) {
        return !shift.getDate().isBefore(draftStart)
            && !shift.getDate().isAfter(draftEnd);
    }
}
```

### Setting the pin flag before solving

```java
// Before submitting to the solver
for (Shift shift : schedule.getShifts()) {
    shift.setPinned(!scheduleState.isDraft(shift));
}
```

### Include ScheduleState in the solution

```java
@PlanningSolution
public class EmployeeSchedule {

    @ProblemFactProperty
    private ScheduleState scheduleState;

    // ...
}
```

---

## 5. Real-Time Planning with ProblemChange

When the problem changes WHILE the solver is running (employee calls in sick,
new shift added), use `ProblemChange`.

### ProblemChange interface

```java
public interface ProblemChange<Solution_> {
    void doChange(Solution_ workingSolution,
                  ProblemChangeDirector problemChangeDirector);
}
```

### Example: Employee calls in sick

```java
public class EmployeeCallsInSickChange implements ProblemChange<EmployeeSchedule> {

    private final Employee sickEmployee;

    public EmployeeCallsInSickChange(Employee sickEmployee) {
        this.sickEmployee = sickEmployee;
    }

    @Override
    public void doChange(EmployeeSchedule workingSolution,
                         ProblemChangeDirector director) {
        // Look up the working copy of the employee
        Employee workingEmployee = director.lookUpWorkingObject(sickEmployee);

        // Unassign all shifts for this employee
        for (Shift shift : workingSolution.getShifts()) {
            if (shift.getEmployee() == workingEmployee) {
                director.changeVariable(shift, "employee",
                        s -> s.setEmployee(null));
            }
        }

        // Remove employee from the value range
        director.removeProblemFact(workingEmployee, 
                workingSolution.getEmployees()::remove);
    }
}
```

### Submitting the change

```java
// Via SolverManager (async)
solverManager.addProblemChange(jobId, new EmployeeCallsInSickChange(carl));

// Via SolverJob
solverJob.addProblemChange(new EmployeeCallsInSickChange(carl));
```

### Critical rules for ProblemChange

1. **Always use `director.lookUpWorkingObject()`** to get the working copy. The solver uses a planning clone — your original objects are different instances.
2. **Use director methods** for all mutations: `addProblemFact`, `removeProblemFact`, `changeProblemProperty`, `addEntity`, `removeEntity`, `changeVariable`.
3. **Never modify the working solution directly** without going through the director.
4. **Requires `@PlanningId`** on all domain classes you look up.
5. **Batch changes** with `addProblemChanges(List<ProblemChange>)` for performance.

### What happens after a ProblemChange

1. Solver stops current solving.
2. Applies the change to the working solution.
3. Restarts (warm start) — construction heuristics fills any new gaps, then local search optimizes.
4. Termination timers reset (but `terminateEarly()` is NOT undone).

---

## 6. Non-Disruptive Replanning

When replanning published shifts is necessary (e.g., sick employee), minimize disruption
by penalizing changes to already-published assignments.

### Store the original assignment

```java
@PlanningEntity
public class Shift {

    @PlanningVariable(allowsUnassigned = true)
    private Employee employee;

    // Set before solving to the currently published value
    private Employee publishedEmployee;

    // ...
}
```

### Penalize changes

```java
Constraint minimizeDisruption(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .filter(shift -> shift.getPublishedEmployee() != null
                    && shift.getEmployee() != shift.getPublishedEmployee())
            .penalize(HardSoftScore.ofSoft(1000))
            .asConstraint("Minimize disruption");
}
```

The high weight (1000) means the solver only changes a published assignment when it improves
the total score by more than 1000 per change — effectively requiring a very good reason
(like resolving a hard constraint violation from a sick employee).

---

## 7. Daemon Mode

For always-on real-time planning, use daemon mode. The solver blocks when idle
and resumes immediately when a `ProblemChange` arrives.

### Configuration

```properties
# In application.properties (Quarkus)
quarkus.timefold.solver.daemon=true
quarkus.timefold.solver.termination.unimproved-spent-limit=30s
```

Or in XML:
```xml
<solver>
    <daemon>true</daemon>
    <termination>
        <unimprovedSpentLimit>30s</unimprovedSpentLimit>
    </termination>
</solver>
```

### Behavior

- Solver runs until termination triggers (e.g., 30s without improvement), then **blocks** instead of returning.
- When a `ProblemChange` arrives, it unblocks, applies the change, and resumes solving.
- `terminateEarly()` is the only way to make `solve()` return — use this for graceful shutdown.

### Processing best solutions

```java
solverManager.solveBuilder()
    .withProblemId(jobId)
    .withProblem(schedule)
    .withBestSolutionConsumer(bestSolution -> {
        if (bestSolution.getScore().isFeasible()) {
            saveToDB(bestSolution);
        }
    })
    .run();
```

**Check `isFeasible()` or `isEveryProblemChangeProcessed()`** — intermediate solutions
during a ProblemChange may be infeasible or uninitialized.

---

## 8. Assignment Recommendation API

For ad-hoc requests ("Can we fit a new shift on Thursday?"), the Recommendation API
gives instant answers without full re-solving.

```java
@Inject
SolutionManager<EmployeeSchedule, HardSoftScore> solutionManager;

// Add the unassigned shift to the solution
Shift newShift = new Shift("new-1", thursday6am, thursday2pm, "Cook", null);
schedule.getShifts().add(newShift);

// Get recommendations
List<RecommendedAssignment<Employee, HardSoftScore>> recommendations =
    solutionManager.recommendAssignment(schedule, newShift, Shift::getEmployee);

// Each recommendation contains:
//   .proposition()   → the Employee
//   .scoreDiff()     → the score impact of this assignment
for (var rec : recommendations) {
    System.out.println(rec.proposition().getName()
            + " → score impact: " + rec.scoreDiff());
}
```

**How it works:** Uses a greedy algorithm with incremental score calculation — millisecond response
time even for large problems. Does NOT run local search.

**After the user accepts a recommendation**, apply it and optionally run the full solver
to optimize around the change.

---

## 9. Backup Planning

Build slack into the schedule for unforeseen events:

```java
// Reward having spare capacity
Constraint spareCapacity(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getTimePeriod, countDistinct(Shift::getEmployee))
            .filter((period, count) -> count < period.getTotalEmployeeCount())
            .reward(HardSoftScore.ONE_SOFT,
                    (period, count) -> period.getTotalEmployeeCount() - count)
            .asConstraint("Spare employee capacity");
}
```

When someone calls in sick:
1. Remove the sick employee (via ProblemChange or resubmit).
2. Unpin their shifts.
3. Restart — the solver uses the spare capacity to reassign.

---

## 10. Complete Employee Scheduling Example

Putting it all together for a weekly replanning cycle:

```java
@ApplicationScoped
public class SchedulingService {

    @Inject
    SolverManager<EmployeeSchedule, String> solverManager;

    public void weeklyReplan() {
        LocalDate today = LocalDate.now();
        ScheduleState state = new ScheduleState(
            today.minusDays(7),       // history: last 7 days
            today,                     // publish start
            today.plusWeeks(1),        // draft start (publish 1 week)
            today.plusWeeks(4)         // draft end (plan 4 weeks ahead)
        );

        // Load shifts from DB for the planning window
        List<Shift> shifts = shiftRepo.findBetween(
                state.getHistoryStart(), state.getDraftEnd());
        List<Employee> employees = employeeRepo.findAll();

        // Pin history and published shifts
        for (Shift shift : shifts) {
            shift.setPinned(!state.isDraft(shift));
        }

        EmployeeSchedule schedule = new EmployeeSchedule(
                state, employees, shifts);

        // Solve async
        solverManager.solveBuilder()
            .withProblemId("weekly-" + today)
            .withProblem(schedule)
            .withBestSolutionConsumer(best -> {
                if (best.getScore().isFeasible()) {
                    saveAndPublishFirstWeek(best, state);
                }
            })
            .run();
    }

    private void saveAndPublishFirstWeek(EmployeeSchedule solution,
                                          ScheduleState state) {
        for (Shift shift : solution.getShifts()) {
            if (state.isDraft(shift)) {
                shiftRepo.save(shift);
                // Publish only the first week of draft
                if (shift.getDate().isBefore(
                        state.getDraftStart().plusWeeks(1))) {
                    notifyEmployee(shift);
                }
            }
        }
    }
}
```

### application.properties for continuous planning

```properties
# Longer termination for weekly batch planning
quarkus.timefold.solver.termination.spent-limit=10m
quarkus.timefold.solver.termination.unimproved-spent-limit=2m

# Use FULL_ASSERT during development
%dev.quarkus.timefold.solver.environment-mode=FULL_ASSERT
%dev.quarkus.timefold.solver.termination.spent-limit=30s
```
