# Constraint Streams API Reference — Timefold Solver (Java)

> **Imports used throughout this file:**
> ```java
> import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;
> import ai.timefold.solver.core.api.score.stream.Constraint;
> import ai.timefold.solver.core.api.score.stream.ConstraintFactory;
> import ai.timefold.solver.core.api.score.stream.ConstraintProvider;
> import ai.timefold.solver.core.api.score.stream.ConstraintCollectors;
> import ai.timefold.solver.core.api.score.stream.Joiners;
> ```

## Table of Contents

1. [ConstraintProvider Interface](#1-constraintprovider-interface)
2. [The Constraint Pipeline](#2-the-constraint-pipeline)
3. [Stream Sources: forEach](#3-stream-sources-foreach)
4. [Filtering](#4-filtering)
5. [Joining](#5-joining)
6. [forEachUniquePair](#6-foreachuniquepair)
7. [Conditional Propagation: ifExists / ifNotExists](#7-conditional-propagation-ifexists--ifnotexists)
8. [Grouping and Collectors](#8-grouping-and-collectors)
9. [Mapping, Flattening, Expanding, Complement](#9-mapping-flattening-expanding-complement)
10. [Penalizing and Rewarding](#10-penalizing-and-rewarding)
11. [Joiners Reference](#11-joiners-reference)
12. [ConstraintCollectors Reference](#12-constraintcollectors-reference)
13. [Load Balancing and Fairness](#13-load-balancing-and-fairness)
14. [Score Types](#14-score-types)
15. [ConstraintVerifier Testing API](#15-constraintverifier-testing-api)
16. [Performance Tips](#16-performance-tips)
17. [Complete Employee Scheduling ConstraintProvider Example](#17-complete-employee-scheduling-constraintprovider-example)

---

## 1. ConstraintProvider Interface

Every Timefold project needs exactly one class implementing `ConstraintProvider`:

```java
public class EmployeeSchedulingConstraintProvider implements ConstraintProvider {

    @Override
    public Constraint[] defineConstraints(ConstraintFactory factory) {
        return new Constraint[] {
            requiredSkill(factory),
            noOverlappingShifts(factory),
            unavailableEmployee(factory),
            balanceShifts(factory)
        };
    }

    // Each constraint is a separate method returning Constraint
    Constraint requiredSkill(ConstraintFactory factory) {
        return factory.forEach(Shift.class)
                .filter(shift -> !shift.getEmployee().getSkills()
                        .contains(shift.getRequiredSkill()))
                .penalize(HardSoftScore.ONE_HARD)
                .asConstraint("Required skill");
    }
    // ... more constraints
}
```

Rules:
- Constraint names must be unique across the entire `ConstraintProvider`.
- Each constraint method must accept `ConstraintFactory` as its only parameter (for testability with `ConstraintVerifier`).
- The method must return `Constraint` (the result of `.asConstraint("name")`).
- Quarkus auto-discovers the `ConstraintProvider` — no XML config needed.

---

## 2. The Constraint Pipeline

Every constraint stream follows this pipeline:

```
Source → [Filter] → [Join/IfExists] → [GroupBy] → [Map/Flatten] → Penalize/Reward → asConstraint
```

Minimum viable constraint:
```java
factory.forEach(Shift.class)
    .penalize(HardSoftScore.ONE_SOFT)
    .asConstraint("Penalize every shift");
```

Stream cardinalities:

| Cardinality | Prefix | Type |
|---|---|---|
| 1 | Uni | `UniConstraintStream<A>` |
| 2 | Bi | `BiConstraintStream<A, B>` |
| 3 | Tri | `TriConstraintStream<A, B, C>` |
| 4 | Quad | `QuadConstraintStream<A, B, C, D>` |

Max cardinality is 4. For higher, use `map()` to reduce to a tuple object first.

---

## 3. Stream Sources: forEach

| Method | Behavior |
|---|---|
| `forEach(Class)` | Selects all instances with non-null genuine planning variables. **Default choice.** |
| `forEachIncludingUnassigned(Class)` | Also includes entities whose planning variable is null (for `allowsUnassigned = true`). |
| `forEachUnfiltered(Class)` | Includes all instances regardless of variable state or consistency. |
| `forEachUniquePair(Class, Joiner...)` | Selects all unique pairs `(A, B)` where `A != B`, no duplicates. See §6. |

`forEach` works on both `@PlanningEntity` classes and problem fact classes registered via `@ProblemFactCollectionProperty`.

---

## 4. Filtering

```java
// Uni filter
factory.forEach(Shift.class)
    .filter(shift -> shift.getEmployee().getName().equals("Ann"))

// Bi filter (after join)
factory.forEach(Shift.class)
    .join(DayOff.class)
    .filter((shift, dayOff) -> shift.getDate().equals(dayOff.getDate()))
```

**Important:** Prefer Joiners over post-join filters for performance. Joiners use indexes; filters evaluate every combination.

---

## 5. Joining

`join()` creates a cartesian product between two streams (inner join). Always use Joiners to restrict matches:

```java
Constraint shiftOnDayOff(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .join(DayOff.class,
                Joiners.equal(Shift::getDate, DayOff::getDate),
                Joiners.equal(Shift::getEmployee, DayOff::getEmployee))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Shift on an off-day");
}
```

A join increases cardinality: Uni + join = Bi, Bi + join = Tri, Tri + join = Quad.

**Joiner evaluation caveat:** Multiple Joiners' mapping functions are evaluated independently — a later Joiner's lambda may run even when an earlier Joiner wouldn't match. Don't assume null-safety from earlier Joiners. Use `Joiners.filtering()` as a safe fallback if needed.

---

## 6. forEachUniquePair

Shorthand for joining a class with itself, producing only unique pairs (no `(A,A)`, no `(B,A)` if `(A,B)` exists):

```java
Constraint noOverlappingShifts(ConstraintFactory factory) {
    return factory.forEachUniquePair(Shift.class,
                Joiners.equal(Shift::getEmployee),
                Joiners.overlapping(Shift::getStart, Shift::getEnd))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("No overlapping shifts");
}
```

This is **much better** than `forEach(Shift.class).join(Shift.class)` + filtering out duplicates manually. Always use `forEachUniquePair` when matching pairs of the same entity type.

---

## 7. Conditional Propagation: ifExists / ifNotExists

Use when you only need to check whether a matching object exists, without needing the matched object in the stream:

```java
// Penalize shifts where an unavailable employee is assigned
Constraint unavailableEmployee(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .ifExists(Availability.class,
                Joiners.equal(Shift::getEmployee, Availability::getEmployee),
                Joiners.equal(Shift::getDate, Availability::getDate),
                Joiners.filtering((shift, avail) ->
                    avail.getType() == AvailabilityType.UNAVAILABLE))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Unavailable employee");
}
```

`ifExists` does NOT increase cardinality (unlike `join`). The stream stays Uni.
`ifNotExists` is the inverse — keeps elements where no matching object exists.

---

## 8. Grouping and Collectors

`groupBy()` groups stream elements by a key and applies collectors. It can increase or decrease cardinality.

```java
// Count shifts per employee
factory.forEach(Shift.class)
    .groupBy(Shift::getEmployee, ConstraintCollectors.count())
    // BiConstraintStream<Employee, Integer>
    .filter((employee, count) -> count > 5)
    .penalize(HardSoftScore.ONE_SOFT,
            (employee, count) -> count - 5)
    .asConstraint("Too many shifts");
```

groupBy supports 0-2 group keys and 0-4 collectors:

```java
// 0 keys + 1 collector → Uni result
.groupBy(count())

// 1 key + 0 collectors → Uni result (distinct keys)
.groupBy(Shift::getEmployee)

// 1 key + 1 collector → Bi result
.groupBy(Shift::getEmployee, count())

// 1 key + 2 collectors → Tri result
.groupBy(Shift::getEmployee, count(), sum(Shift::getHours))

// 2 keys + 1 collector → Tri result
.groupBy(Shift::getEmployee, Shift::getDate, count())
```

**Warning:** After `groupBy`, you lose access to the original objects. Only the keys and collector results are available.

---

## 9. Mapping, Flattening, Expanding, Complement

### map()
Transforms stream elements. Can decrease cardinality:

```java
// Bi → Uni (extract one field)
.join(Employee.class)
.groupBy((shift, employee) -> employee)  // equivalent to map

// Bi → Uni (create tuple)
factory.forEachUniquePair(Visit.class)
    .map((v1, v2) -> Pair.of(v1.getName(), v2.getName()))
```

### flattenLast()
Transforms a collection in the last position into individual stream elements:

```java
factory.forEach(Match.class)
    .groupBy(Match::getHomeTeam,
        ConstraintCollectors.toConsecutiveSequences(Match::getRoundId))
    .flattenLast(SequenceChain::getConsecutiveSequences)
    // Each consecutive sequence is now a separate tuple
```

### expand()
Adds computed values to each tuple, increasing cardinality:

```java
factory.forEach(Shift.class)
    .expand(Shift::getDurationInHours)
    // Uni<Shift> → Bi<Shift, Integer>
```

### complement()
Adds missing problem facts to a grouped stream (critical for fairness/load balancing):

```java
factory.forEach(Shift.class)
    .groupBy(Shift::getEmployee, ConstraintCollectors.count())
    // BiStream<Employee, Integer> — only employees WITH shifts
    .complement(Employee.class, employee -> 0)
    // Now includes ALL employees; those without shifts get count = 0
```

---

## 10. Penalizing and Rewarding

Every constraint stream must end with a penalty or reward, then `.asConstraint("name")`:

```java
// Static weight — each match penalizes by exactly ONE_HARD
.penalize(HardSoftScore.ONE_HARD)
.asConstraint("Name");

// Dynamic weight — match weigher multiplies the base weight
.penalize(HardSoftScore.ONE_SOFT, shift -> shift.getHours())
.asConstraint("Name");

// Reward (improves score)
.reward(HardSoftScore.ONE_SOFT)
.asConstraint("Name");

// Configurable weight (from ConstraintConfiguration)
.penalizeConfigurable()
.asConstraint("Name");
```

**Critical:** Every chain MUST end with `.asConstraint("unique name")`. Without it, the constraint silently doesn't exist.

---

## 11. Joiners Reference

`import static ai.timefold.solver.core.api.score.stream.Joiners.*;`

| Joiner | Description | Example |
|---|---|---|
| `equal(Function)` | Same property on both sides | `equal(Shift::getEmployee)` |
| `equal(FunctionA, FunctionB)` | Different properties that must be equal | `equal(Shift::getDate, DayOff::getDate)` |
| `lessThan(FunctionA, FunctionB)` | A's value < B's value (Comparable) | `lessThan(Shift::getEnd, Shift::getStart)` |
| `lessThanOrEqual(...)` | A's value <= B's value | |
| `greaterThan(...)` | A's value > B's value | |
| `greaterThanOrEqual(...)` | A's value >= B's value | |
| `overlapping(startA, endA, startB, endB)` | Two intervals overlap | `overlapping(Shift::getStart, Shift::getEnd)` |
| `overlapping(startA, endA)` | Same-type shorthand (both sides use same getters) | Used with `forEachUniquePair` |
| `filtering(BiPredicate)` | Custom filter (slower — no indexing) | `filtering((a, b) -> a.overlaps(b))` |

**Never use `==` or `.equals()` in a post-join filter to match entities — always use `Joiners.equal()` for performance.**

---

## 12. ConstraintCollectors Reference

`import static ai.timefold.solver.core.api.score.stream.ConstraintCollectors.*;`

| Collector | Return type | Description |
|---|---|---|
| `count()` | `int` | Count elements. Use `countBi()`, `countTri()`, `countQuad()` for higher cardinality. |
| `countLong()` | `long` | Long variant of count. |
| `countDistinct()` | `int` | Count unique elements. |
| `sum(ToIntFunction)` | `int` | Sum an int property. Variants: `sumLong()`, `sumBigDecimal()`, `sumDuration()`. |
| `average(ToIntFunction)` | `double` | Average of int property. Returns `null` for empty groups. |
| `min(Function)` | `T` | Minimum by Comparable property. |
| `max(Function)` | `T` | Maximum by Comparable property. |
| `toList()` | `List<T>` | Collect to list. **Disables incremental calculation — use sparingly.** |
| `toSet()` | `Set<T>` | Collect to set. Same performance caveat. |
| `toSortedSet()` | `SortedSet<T>` | Sorted set variant. |
| `toMap(keyFn, valueFn)` | `Map<K,V>` | Collect to map. |
| `loadBalance(Function)` | `LoadBalance` | Fairness metric. See §13. |
| `toConsecutiveSequences(Function)` | `SequenceChain` | Group into consecutive sequences. Use with `flattenLast()`. |
| `toConnectedTemporalRanges(startFn, endFn)` | `ConnectedRangeChain` | Group overlapping time ranges. |
| `toConnectedRanges(startFn, endFn, diffFn)` | `ConnectedRangeChain` | Non-temporal variant. |
| `conditionally(Predicate, collector)` | varies | Only applies collector when predicate is true. |
| `compose(collector1, collector2, mergerFn)` | varies | Combine 2-4 collectors into one. |

---

## 13. Load Balancing and Fairness

For fair distribution of work across employees, use the `loadBalance` collector:

```java
Constraint fairShiftDistribution(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(ConstraintCollectors.loadBalance(Shift::getEmployee))
            .penalizeBigDecimal(HardSoftBigDecimalScore.ONE_SOFT,
                    LoadBalance::unfairness)
            .asConstraint("Fair shift distribution");
}
```

To include employees with zero assignments (critical for fairness), use `complement()`:

```java
Constraint fairShiftDistribution(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee, ConstraintCollectors.count())
            .complement(Employee.class, employee -> 0)
            // Now all employees are in the stream, even those with 0 shifts
            .groupBy(ConstraintCollectors.loadBalance(
                    (employee, count) -> employee, (employee, count) -> count))
            .penalizeBigDecimal(HardSoftBigDecimalScore.ONE_SOFT,
                    LoadBalance::unfairness)
            .asConstraint("Fair shift distribution");
}
```

**Note:** For load balancing, the Timefold docs recommend `BigDecimal`-based score types (e.g., `HardSoftBigDecimalScore`) because the unfairness value is a rational number. Rounding to `int` can create score traps. If you must use `HardSoftScore`, multiply unfairness by a scaling factor (e.g., 1000) before converting to int.

---

## 14. Score Types

| Type | Levels | Use when |
|---|---|---|
| `SimpleScore` | 1 | Only one priority level |
| `HardSoftScore` | 2 | **Default for most problems.** Hard = must satisfy, Soft = optimize. |
| `HardMediumSoftScore` | 3 | Need medium level (e.g., "assign as many shifts as possible") |
| `HardSoftBigDecimalScore` | 2 | Need decimal precision (fairness/load balancing) |
| `BendableScore` | N | Configurable number of hard/soft levels |

Common score constants:
```java
HardSoftScore.ZERO
HardSoftScore.ONE_HARD          // -1hard/0soft per match
HardSoftScore.ONE_SOFT          // 0hard/-1soft per match
HardSoftScore.of(0, -1)         // equivalent to ONE_SOFT
HardSoftScore.ofHard(2)         // -2hard penalty
HardSoftScore.ofSoft(3)         // -3soft penalty
```

---

## 15. ConstraintVerifier Testing API

The `ConstraintVerifier` tests each constraint in isolation.

### Maven dependency

```xml
<dependency>
    <groupId>ai.timefold.solver</groupId>
    <artifactId>timefold-solver-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Quarkus injection (recommended)

```java
@QuarkusTest
class EmployeeSchedulingConstraintProviderTest {

    @Inject
    ConstraintVerifier<EmployeeSchedulingConstraintProvider, EmployeeSchedule>
            constraintVerifier;
```

### Manual construction (non-Quarkus)

```java
ConstraintVerifier<EmployeeSchedulingConstraintProvider, EmployeeSchedule>
    constraintVerifier = ConstraintVerifier.build(
        new EmployeeSchedulingConstraintProvider(),
        EmployeeSchedule.class,
        Shift.class);         // list ALL entity classes
```

### Testing a single constraint

```java
@Test
void whenEmployeeLacksSkill_thenPenalize() {
    Employee ann = new Employee("Ann", Set.of("Waiter"));

    constraintVerifier.verifyThat(
            EmployeeSchedulingConstraintProvider::requiredSkill)
        .given(
            new Shift("1", MONDAY_6AM, MONDAY_2PM, "Cook", ann))
        .penalizesBy(1);
}

@Test
void whenEmployeeHasSkill_thenNoPenalty() {
    Employee ann = new Employee("Ann", Set.of("Waiter"));

    constraintVerifier.verifyThat(
            EmployeeSchedulingConstraintProvider::requiredSkill)
        .given(
            new Shift("1", MONDAY_6AM, MONDAY_2PM, "Waiter", ann))
        .penalizesBy(0);
}
```

### Testing all constraints together

```java
@Test
void givenSolution_thenCheckScore() {
    constraintVerifier.verifyThat()
        .given(shift1, shift2, employee1)
        .scores(HardSoftScore.ofSoft(-2));
}
```

### Key methods

| Method | Description |
|---|---|
| `verifyThat(ConstraintProvider::method)` | Test a single constraint method |
| `verifyThat()` | Test the entire ConstraintProvider |
| `.given(Object...)` | Provide facts/entities directly (no solution needed) |
| `.givenSolution(solution)` | Provide a full PlanningSolution |
| `.settingAllShadowVariables()` | Compute shadow variables before testing (use with `givenSolution`) |
| `.penalizesBy(int)` | Assert total penalty match weight |
| `.penalizes(int)` | Assert number of penalty matches |
| `.rewardsWith(int)` | Assert total reward match weight |
| `.rewards(int)` | Assert number of reward matches |
| `.scores(Score)` | Assert exact score (only with `verifyThat()` no-arg) |

**Important:** `given()` does NOT update shadow variables. If your constraint depends on shadow variables, either:
- Set them manually in your test data, or
- Use `.givenSolution(solution).settingAllShadowVariables()`

---

## 16. Performance Tips

1. **Use Joiners, not post-join filters.** Joiners use hash indexes; filters evaluate every pair.

   ```java
   // BAD — creates full cartesian product, then filters
   factory.forEach(Shift.class)
       .join(DayOff.class)
       .filter((s, d) -> s.getEmployee().equals(d.getEmployee()))

   // GOOD — indexed join
   factory.forEach(Shift.class)
       .join(DayOff.class,
           Joiners.equal(Shift::getEmployee, DayOff::getEmployee))
   ```

2. **Use `ifExists`/`ifNotExists` instead of `join` when you don't need the joined object.** Join increases cardinality; ifExists does not.

3. **Use `forEachUniquePair` for same-entity pairs.** Never `forEach(X).join(X)` + manual dedup.

4. **Filter early.** Apply `filter()` as close to `forEach()` as possible to reduce downstream combinations.

5. **Avoid `toList()` and `toSet()` collectors when possible.** They disable incremental calculation for that group. Use `count()`, `sum()`, `min()`, `max()` instead.

6. **Don't use mutable objects as group keys.** Keys must have stable `hashCode()`. Planning entities are safe as keys only if their `hashCode` doesn't depend on planning variables.

7. **Precompute when possible.** If a calculation in `penalize(score, weigher)` doesn't depend on planning variables, compute it once in the domain model and cache it.

---

## 17. Complete Employee Scheduling ConstraintProvider Example

```java
package com.example.scheduling.solver;

import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;
import ai.timefold.solver.core.api.score.stream.Constraint;
import ai.timefold.solver.core.api.score.stream.ConstraintFactory;
import ai.timefold.solver.core.api.score.stream.ConstraintProvider;
import ai.timefold.solver.core.api.score.stream.Joiners;

import com.example.scheduling.domain.Availability;
import com.example.scheduling.domain.AvailabilityType;
import com.example.scheduling.domain.Shift;

import java.time.Duration;

public class EmployeeSchedulingConstraintProvider implements ConstraintProvider {

    @Override
    public Constraint[] defineConstraints(ConstraintFactory factory) {
        return new Constraint[] {
            // Hard constraints
            requiredSkill(factory),
            noOverlappingShifts(factory),
            unavailableEmployee(factory),
            // Soft constraints
            preferredEmployee(factory),
            minimumRestBetweenShifts(factory)
        };
    }

    // --- HARD CONSTRAINTS ---

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

    Constraint unavailableEmployee(ConstraintFactory factory) {
        return factory.forEach(Shift.class)
                .join(Availability.class,
                    Joiners.equal(Shift::getEmployee, Availability::getEmployee),
                    Joiners.equal(Shift::getDate, Availability::getDate))
                .filter((shift, avail) ->
                        avail.getType() == AvailabilityType.UNAVAILABLE)
                .penalize(HardSoftScore.ONE_HARD)
                .asConstraint("Unavailable employee");
    }

    // --- SOFT CONSTRAINTS ---

    Constraint preferredEmployee(ConstraintFactory factory) {
        return factory.forEach(Shift.class)
                .join(Availability.class,
                    Joiners.equal(Shift::getEmployee, Availability::getEmployee),
                    Joiners.equal(Shift::getDate, Availability::getDate))
                .filter((shift, avail) ->
                        avail.getType() == AvailabilityType.PREFERRED)
                .reward(HardSoftScore.ONE_SOFT)
                .asConstraint("Preferred employee");
    }

    Constraint minimumRestBetweenShifts(ConstraintFactory factory) {
        return factory.forEachUniquePair(Shift.class,
                    Joiners.equal(Shift::getEmployee))
                .filter((s1, s2) -> {
                    Duration gap = Duration.between(s1.getEnd(), s2.getStart());
                    if (gap.isNegative()) {
                        gap = Duration.between(s2.getEnd(), s1.getStart());
                    }
                    return !gap.isNegative() && gap.toHours() < 12;
                })
                .penalize(HardSoftScore.ONE_SOFT,
                        (s1, s2) -> {
                            Duration gap = Duration.between(s1.getEnd(), s2.getStart());
                            if (gap.isNegative()) {
                                gap = Duration.between(s2.getEnd(), s1.getStart());
                            }
                            return (int) (12 - gap.toHours());
                        })
                .asConstraint("Minimum 12h rest between shifts");
    }
}
```
