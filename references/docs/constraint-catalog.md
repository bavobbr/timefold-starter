# Employee Scheduling Constraint Catalog — Ready-Made Implementations

> Copy-paste-ready ConstraintStream implementations for common employee scheduling constraints.
> All examples assume the domain model from `modeling-guide.md` (Shift as `@PlanningEntity`, Employee as value).
>
> **Imports assumed throughout:**
> ```java
> import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;
> import ai.timefold.solver.core.api.score.stream.*;
> import static ai.timefold.solver.core.api.score.stream.Joiners.*;
> import static ai.timefold.solver.core.api.score.stream.ConstraintCollectors.*;
> import java.time.Duration;
> import java.time.LocalDate;
> ```

## Table of Contents

| # | Constraint | Level | Section |
|---|---|---|---|
| 1 | Required skill | HARD | [§1](#1-required-skill) |
| 2 | No overlapping shifts | HARD | [§2](#2-no-overlapping-shifts) |
| 3 | Employee unavailability | HARD | [§3](#3-employee-unavailability) |
| 4 | Minimum rest between shifts | HARD | [§4](#4-minimum-rest-between-shifts) |
| 5 | Maximum shifts per day | HARD | [§5](#5-maximum-shifts-per-day) |
| 6 | Maximum shifts per week | SOFT | [§6](#6-maximum-shifts-per-week) |
| 7 | Maximum consecutive working days | HARD | [§7](#7-maximum-consecutive-working-days) |
| 8 | Preferred/unpreferred availability | SOFT | [§8](#8-preferred-and-unpreferred-availability) |
| 9 | Fair shift distribution (load balance) | SOFT | [§9](#9-fair-shift-distribution) |
| 10 | Fair hours distribution | SOFT | [§10](#10-fair-hours-distribution) |
| 11 | Weekend balance | SOFT | [§11](#11-weekend-balance) |
| 12 | Minimum staffing per shift type | HARD | [§12](#12-minimum-staffing-per-shift-type) |
| 13 | No shift-on-requested-day-off | HARD | [§13](#13-no-shift-on-requested-day-off) |
| 14 | Undesired shift pattern (no back-to-back close-open) | SOFT | [§14](#14-no-back-to-back-close-open) |

---

## Domain Assumptions

These constraints assume the following domain classes. Adapt getter names to your model:

```java
// @PlanningEntity
Shift {
    Employee getEmployee();          // @PlanningVariable
    LocalDateTime getStart();
    LocalDateTime getEnd();
    LocalDate getDate();             // derived: getStart().toLocalDate()
    String getRequiredSkill();
    long getDurationInHours();       // derived: Duration.between(start, end).toHours()
}

// Problem fact
Employee {
    String getName();
    Set<String> getSkills();
}

// Problem fact
Availability {
    Employee getEmployee();
    LocalDate getDate();
    AvailabilityType getType();      // UNAVAILABLE, PREFERRED, UNPREFERRED
}
```

---

## 1. Required Skill

**Level:** HARD — employee must have the skill required by the shift.

```java
Constraint requiredSkill(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .filter(shift -> !shift.getEmployee().getSkills()
                    .contains(shift.getRequiredSkill()))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Required skill");
}
```

---

## 2. No Overlapping Shifts

**Level:** HARD — one employee cannot work two shifts that overlap in time.

```java
Constraint noOverlappingShifts(ConstraintFactory factory) {
    return factory.forEachUniquePair(Shift.class,
                equal(Shift::getEmployee),
                overlapping(Shift::getStart, Shift::getEnd))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("No overlapping shifts");
}
```

**Note:** `forEachUniquePair` avoids counting `(A,B)` and `(B,A)` as separate violations.

---

## 3. Employee Unavailability

**Level:** HARD — don't assign an employee to a shift on a day they're unavailable.

```java
Constraint unavailableEmployee(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .join(Availability.class,
                equal(Shift::getEmployee, Availability::getEmployee),
                equal(Shift::getDate, Availability::getDate))
            .filter((shift, avail) ->
                    avail.getType() == AvailabilityType.UNAVAILABLE)
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Unavailable employee");
}
```

**Alternative using ifExists** (slightly faster — doesn't need the Availability in downstream):

```java
Constraint unavailableEmployee(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .ifExists(Availability.class,
                equal(Shift::getEmployee, Availability::getEmployee),
                equal(Shift::getDate, Availability::getDate),
                filtering((shift, avail) ->
                        avail.getType() == AvailabilityType.UNAVAILABLE))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Unavailable employee");
}
```

---

## 4. Minimum Rest Between Shifts

**Level:** HARD — enforce a minimum gap (e.g., 10 hours) between consecutive shifts of the same employee.

```java
private static final int MIN_REST_HOURS = 10;

Constraint minimumRestBetweenShifts(ConstraintFactory factory) {
    return factory.forEachUniquePair(Shift.class,
                equal(Shift::getEmployee))
            .filter((s1, s2) -> {
                Duration gap = gapBetween(s1, s2);
                return !gap.isNegative() && gap.toHours() < MIN_REST_HOURS;
            })
            .penalize(HardSoftScore.ONE_HARD,
                    (s1, s2) -> (int) (MIN_REST_HOURS - gapBetween(s1, s2).toHours()))
            .asConstraint("Minimum rest between shifts");
}

// Helper — returns the non-overlapping gap between two shifts
private static Duration gapBetween(Shift s1, Shift s2) {
    if (s1.getEnd().isBefore(s2.getStart())) {
        return Duration.between(s1.getEnd(), s2.getStart());
    } else if (s2.getEnd().isBefore(s1.getStart())) {
        return Duration.between(s2.getEnd(), s1.getStart());
    }
    return Duration.ZERO; // overlapping — handled by noOverlappingShifts
}
```

**Tip:** If shifts never overlap (enforced by the hard constraint above), you can simplify the gap calculation.

---

## 5. Maximum Shifts Per Day

**Level:** HARD — an employee may work at most N shifts per calendar day.

```java
private static final int MAX_SHIFTS_PER_DAY = 1;

Constraint maxShiftsPerDay(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee, Shift::getDate, count())
            .filter((employee, date, shiftCount) ->
                    shiftCount > MAX_SHIFTS_PER_DAY)
            .penalize(HardSoftScore.ONE_HARD,
                    (employee, date, shiftCount) ->
                            shiftCount - MAX_SHIFTS_PER_DAY)
            .asConstraint("Max shifts per day");
}
```

**Alternative with forEachUniquePair (if max = 1):**

```java
Constraint atMostOneShiftPerDay(ConstraintFactory factory) {
    return factory.forEachUniquePair(Shift.class,
                equal(Shift::getEmployee),
                equal(Shift::getDate))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("At most one shift per day");
}
```

---

## 6. Maximum Shifts Per Week

**Level:** SOFT — prefer employees don't exceed a weekly shift limit.

```java
private static final int MAX_SHIFTS_PER_WEEK = 5;

Constraint maxShiftsPerWeek(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee,
                    shift -> shift.getDate().get(java.time.temporal.IsoFields.WEEK_OF_WEEK_BASED_YEAR),
                    count())
            .filter((employee, week, shiftCount) ->
                    shiftCount > MAX_SHIFTS_PER_WEEK)
            .penalize(HardSoftScore.ONE_SOFT,
                    (employee, week, shiftCount) ->
                            shiftCount - MAX_SHIFTS_PER_WEEK)
            .asConstraint("Max shifts per week");
}
```

---

## 7. Maximum Consecutive Working Days

**Level:** HARD — no employee should work more than N consecutive calendar days.

```java
private static final int MAX_CONSECUTIVE_DAYS = 5;

Constraint maxConsecutiveWorkingDays(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee,
                    toConsecutiveSequences(shift -> shift.getDate().toEpochDay()))
            .flattenLast(SequenceChain::getConsecutiveSequences)
            .filter((employee, seq) -> seq.getCount() > MAX_CONSECUTIVE_DAYS)
            .penalize(HardSoftScore.ONE_HARD,
                    (employee, seq) -> seq.getCount() - MAX_CONSECUTIVE_DAYS)
            .asConstraint("Max consecutive working days");
}
```

**Requires import:**
```java
import ai.timefold.solver.core.api.score.stream.common.SequenceChain;
```

**How it works:** `toConsecutiveSequences()` groups shifts by employee and finds runs of consecutive epoch days. A sequence of 6 consecutive days when the max is 5 gets penalized by 1.

**Important:** This counts unique days. If an employee has two shifts on the same day, that day is counted only once because we group by `toEpochDay()`. If your employee might have multiple shifts per day and you want to count days (not shifts), first `groupBy(employee, date)` to deduplicate, then apply `toConsecutiveSequences`.

---

## 8. Preferred and Unpreferred Availability

**Level:** SOFT — reward shifts on preferred days; penalize shifts on unpreferred days.

```java
Constraint preferredAvailability(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .join(Availability.class,
                equal(Shift::getEmployee, Availability::getEmployee),
                equal(Shift::getDate, Availability::getDate))
            .filter((shift, avail) ->
                    avail.getType() == AvailabilityType.PREFERRED)
            .reward(HardSoftScore.ONE_SOFT)
            .asConstraint("Preferred availability");
}

Constraint unpreferredAvailability(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .join(Availability.class,
                equal(Shift::getEmployee, Availability::getEmployee),
                equal(Shift::getDate, Availability::getDate))
            .filter((shift, avail) ->
                    avail.getType() == AvailabilityType.UNPREFERRED)
            .penalize(HardSoftScore.ONE_SOFT)
            .asConstraint("Unpreferred availability");
}
```

---

## 9. Fair Shift Distribution

**Level:** SOFT — distribute shifts evenly across all employees.

```java
Constraint fairShiftDistribution(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee, count())
            .complement(Employee.class, employee -> 0)
            .groupBy(loadBalance(
                    (employee, count) -> employee,
                    (employee, count) -> count))
            .penalizeBigDecimal(HardSoftBigDecimalScore.ONE_SOFT,
                    LoadBalance::unfairness)
            .asConstraint("Fair shift distribution");
}
```

**Requires:**
```java
import ai.timefold.solver.core.api.score.buildin.hardsoftbigdecimal.HardSoftBigDecimalScore;
import ai.timefold.solver.core.api.score.stream.common.LoadBalance;
```

**Why BigDecimal?** The unfairness value is a rational number. Using `HardSoftScore` (int) causes score traps due to rounding. If you must use `HardSoftScore`, multiply unfairness by 1000 and convert:

```java
// Approximate version with HardSoftScore (not recommended)
.penalize(HardSoftScore.ONE_SOFT,
        loadBalance -> (int) (loadBalance.unfairness().doubleValue() * 1000))
```

**Why complement?** Without `complement()`, employees with zero shifts are invisible to the stream. `complement(Employee.class, e -> 0)` adds them with count = 0, so the solver knows about them and can assign shifts fairly.

---

## 10. Fair Hours Distribution

**Level:** SOFT — balance total working hours (not just shift count) across employees.

```java
Constraint fairHoursDistribution(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getEmployee, sum(Shift::getDurationInHours))
            .complement(Employee.class, employee -> 0)
            .groupBy(loadBalance(
                    (employee, hours) -> employee,
                    (employee, hours) -> hours))
            .penalizeBigDecimal(HardSoftBigDecimalScore.ONE_SOFT,
                    LoadBalance::unfairness)
            .asConstraint("Fair hours distribution");
}
```

**Note:** `getDurationInHours()` should return `int`. If your shifts have different durations, this is fairer than counting shifts.

---

## 11. Weekend Balance

**Level:** SOFT — each employee should work a similar number of weekend shifts.

```java
Constraint weekendBalance(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .filter(shift -> isWeekend(shift.getDate()))
            .groupBy(Shift::getEmployee, count())
            .complement(Employee.class, employee -> 0)
            .groupBy(loadBalance(
                    (employee, weekendCount) -> employee,
                    (employee, weekendCount) -> weekendCount))
            .penalizeBigDecimal(HardSoftBigDecimalScore.ONE_SOFT,
                    LoadBalance::unfairness)
            .asConstraint("Weekend balance");
}

private static boolean isWeekend(LocalDate date) {
    return date.getDayOfWeek() == java.time.DayOfWeek.SATURDAY
        || date.getDayOfWeek() == java.time.DayOfWeek.SUNDAY;
}
```

---

## 12. Minimum Staffing Per Shift Type

**Level:** HARD — each shift "slot" (defined by time + required skill) must have enough employees assigned.

This constraint depends on your model. If you have a separate `ShiftType` or `ShiftSlot` problem fact that defines required headcount:

```java
// Assumes: ShiftSlot { LocalDateTime start, end; String skill; int requiredCount; }
// Shifts reference their ShiftSlot.

Constraint minimumStaffing(ConstraintFactory factory) {
    return factory.forEach(ShiftSlot.class)
            .join(Shift.class,
                equal(ShiftSlot::getStart, Shift::getStart),
                equal(ShiftSlot::getEnd, Shift::getEnd),
                equal(ShiftSlot::getSkill, Shift::getRequiredSkill))
            .groupBy((slot, shift) -> slot, count())
            .filter((slot, assignedCount) ->
                    assignedCount < slot.getRequiredCount())
            .penalize(HardSoftScore.ONE_HARD,
                    (slot, assignedCount) ->
                            slot.getRequiredCount() - assignedCount)
            .asConstraint("Minimum staffing");
}
```

**Simpler alternative** if you just want "at least one employee per shift" (always satisfied when `@PlanningVariable` doesn't use `allowsUnassigned`): no constraint needed — the solver always assigns a value. If you use `allowsUnassigned = true`, use `forEachIncludingUnassigned` and penalize nulls:

```java
Constraint assignEveryShift(ConstraintFactory factory) {
    return factory.forEachIncludingUnassigned(Shift.class)
            .filter(shift -> shift.getEmployee() == null)
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("Assign every shift");
}
```

---

## 13. No Shift on Requested Day Off

**Level:** HARD — if an employee has requested a specific day off, don't schedule them.

If you model day-off requests as a separate problem fact (not using Availability):

```java
// Assumes: DayOffRequest { Employee employee; LocalDate date; }

Constraint noShiftOnDayOff(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .ifExists(DayOffRequest.class,
                equal(Shift::getEmployee, DayOffRequest::getEmployee),
                equal(Shift::getDate, DayOffRequest::getDate))
            .penalize(HardSoftScore.ONE_HARD)
            .asConstraint("No shift on requested day off");
}
```

---

## 14. No Back-to-Back Close-Open

**Level:** SOFT — avoid scheduling an employee for a late shift followed by an early shift the next day.

```java
private static final int CLOSE_HOUR = 20;  // "close" shift starts at 8 PM or later
private static final int OPEN_HOUR = 8;    // "open" shift starts at 8 AM or earlier

Constraint noCloseOpen(ConstraintFactory factory) {
    return factory.forEachUniquePair(Shift.class,
                equal(Shift::getEmployee))
            .filter((s1, s2) -> isCloseOpen(s1, s2) || isCloseOpen(s2, s1))
            .penalize(HardSoftScore.ONE_SOFT)
            .asConstraint("No close-open pattern");
}

private static boolean isCloseOpen(Shift earlier, Shift later) {
    return earlier.getStart().getHour() >= CLOSE_HOUR
        && later.getStart().getHour() <= OPEN_HOUR
        && earlier.getDate().plusDays(1).equals(later.getDate());
}
```

---

## Assembling the ConstraintProvider

Pick the constraints you need and assemble them:

```java
public class EmployeeSchedulingConstraintProvider implements ConstraintProvider {

    @Override
    public Constraint[] defineConstraints(ConstraintFactory factory) {
        return new Constraint[] {
            // Hard
            requiredSkill(factory),
            noOverlappingShifts(factory),
            unavailableEmployee(factory),
            minimumRestBetweenShifts(factory),
            maxShiftsPerDay(factory),
            maxConsecutiveWorkingDays(factory),
            // Soft
            maxShiftsPerWeek(factory),
            preferredAvailability(factory),
            unpreferredAvailability(factory),
            fairShiftDistribution(factory),
            weekendBalance(factory),
            noCloseOpen(factory)
        };
    }

    // ... paste constraint methods here
}
```

**Ordering convention:** Hard constraints first, then soft. Within each level, order doesn't matter for correctness but helps readability.

---

## Tuning Constraint Weights

If you need different weights per constraint (not just ONE_HARD / ONE_SOFT), use multiplied scores:

```java
// Higher penalty for missing required skill vs other hard constraints
.penalize(HardSoftScore.ofHard(10))

// Weekend work is twice as undesirable as weekday work
.penalize(HardSoftScore.ofSoft(2))
```

For runtime-adjustable weights, see `@ConstraintConfiguration` and `penalizeConfigurable()` in the Timefold docs.
