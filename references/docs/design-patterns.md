# Design Patterns for Timefold Solver

> Architectural patterns and modeling decisions for employee scheduling and related problems.
> Read this when making structural decisions about your domain model.

## Table of Contents

1. [Domain Model Design Checklist](#1-domain-model-design-checklist)
2. [Choosing the Planning Entity](#2-choosing-the-planning-entity)
3. [Overconstrained Planning](#3-overconstrained-planning)
4. [Pinning (Immovable Entities)](#4-pinning-immovable-entities)
5. [Many-to-Many Relationships](#5-many-to-many-relationships)
6. [Multiple Planning Variables](#6-multiple-planning-variables)
7. [Score Level Strategy](#7-score-level-strategy)
8. [Backup Planning](#8-backup-planning)
9. [Constraint Weight Strategy](#9-constraint-weight-strategy)
10. [Time Grain Pattern vs Direct Assignment](#10-time-grain-pattern-vs-direct-assignment)

---

## 1. Domain Model Design Checklist

Follow this process when designing your domain model:

1. **Draw a class diagram.** Identify all classes and their relationships.
2. **Create sample instances.** E.g., Employee: Ann, Beth, Carl. Shift: Monday-Morning, Monday-Evening.
3. **Color the relationships that change during planning orange.** These become planning variables.
4. **Color relationships derived from planning variables purple.** These become shadow variables.
5. **Ensure every planning entity has at least one problem property** (a field that is NOT a planning variable). A planning entity cannot consist solely of planning variables + ID.
6. **Minimize planning variables.** Every additional planning variable exponentially increases the search space. If one variable can be derived from another, make it a shadow variable.
7. **Verify:** when all planning variables are null, is the entity still describable? E.g., "the Monday 6AM-2PM Cook shift" makes sense even before an employee is assigned.

---

## 2. Choosing the Planning Entity

**The class that changes during planning is the planning entity.** The field that the solver fills in is the planning variable.

For employee scheduling:

| Pattern | Planning Entity | Planning Variable | When to use |
|---|---|---|---|
| **Shift → Employee** | Shift | `employee` | Most common. "Assign an employee to each shift." |
| **ShiftAssignment** | ShiftAssignment | `employee` (and/or `shift`) | When many-to-many (see §5) |
| **Employee → List<Shift>** | Employee | `@PlanningListVariable shifts` | When ordering matters (routing-style) |

**Recommended for employee scheduling:** Shift as planning entity, Employee as planning variable value. This is the pattern used in the official quickstarts.

```java
@PlanningEntity
public class Shift {
    @PlanningVariable   // Solver assigns this
    private Employee employee;

    // Problem properties (fixed during solving)
    private LocalDateTime start;
    private LocalDateTime end;
    private String requiredSkill;
}
```

---

## 3. Overconstrained Planning

When there aren't enough employees to fill all shifts (more shifts than available employee-hours), the solver will break hard constraints trying to assign everyone. Instead, allow some shifts to be unassigned.

### Step 1: Allow unassigned

```java
@PlanningVariable(allowsUnassigned = true)
private Employee employee;
```

### Step 2: Switch to HardMediumSoftScore

```java
@PlanningScore
private HardMediumSoftScore score;
```

### Step 3: Penalize unassigned shifts at the medium level

```java
Constraint assignEveryShift(ConstraintFactory factory) {
    return factory.forEachIncludingUnassigned(Shift.class)
            .filter(shift -> shift.getEmployee() == null)
            .penalize(HardMediumSoftScore.ONE_MEDIUM)
            .asConstraint("Assign every shift");
}
```

### Step 4: Guard existing constraints against null

When `allowsUnassigned = true`, `forEach()` still excludes null-variable entities by default. But if you use `forEachIncludingUnassigned()` anywhere, add null checks:

```java
// This is safe — forEach already excludes null employees
factory.forEach(Shift.class)
    .filter(shift -> !shift.getEmployee().getSkills()...)  // OK

// This needs a null check
factory.forEachIncludingUnassigned(Shift.class)
    .filter(shift -> shift.getEmployee() != null && ...)  // Needed
```

**Score level guidance:**
- **Hard:** Must not be violated (physical impossibilities, legal requirements)
- **Medium:** "Assign as much as possible" — penalize unassigned entities here
- **Soft:** Optimization preferences (fairness, employee preferences)

---

## 4. Pinning (Immovable Entities)

Pinned entities are locked to their current values. The solver cannot change them.

### Use cases
- Past shifts that are already worked
- Manually assigned shifts that a manager locked in
- Shifts in a continuous planning window that have passed the "freeze" point

### Implementation

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

If `pinned == true`, the solver will not change the employee assignment for this shift, even if the current value is suboptimal. The solver plans around pinned entities.

**Important:** A pinned entity with a null planning variable and `allowsUnassigned = false` will throw an exception in Timefold 1.10+. If a shift is pinned, it must already have a valid assignment (or use `allowsUnassigned = true`).

---

## 5. Many-to-Many Relationships

If your problem has a many-to-many relationship (e.g., multiple employees can be assigned to the same shift, and each employee works multiple shifts), introduce an intermediate assignment class.

### Before (broken)
```
Shift ←→ Employee  (many-to-many)
```

### After (correct)
```
Shift ←one-to-many→ ShiftAssignment ←many-to-one→ Employee
              (problem fact)     (planning entity)    (planning variable)
```

```java
@PlanningEntity
public class ShiftAssignment {
    @PlanningId
    private String id;

    // Problem property (fixed)
    private Shift shift;

    // Planning variable (solver assigns this)
    @PlanningVariable
    private Employee employee;
}
```

The `@PlanningSolution` then has:
```java
@PlanningEntityCollectionProperty
private List<ShiftAssignment> assignments;

@ProblemFactCollectionProperty
@ValueRangeProvider
private List<Employee> employees;

@ProblemFactCollectionProperty
private List<Shift> shifts;
```

**When to use:** Only when you need multiple employees per shift slot. If each shift needs exactly one employee, use the simpler Shift-as-entity pattern from §2.

---

## 6. Multiple Planning Variables

A planning entity can have multiple planning variables. For school timetabling, a Lesson has both `timeslot` and `room`:

```java
@PlanningEntity
public class Lesson {
    @PlanningVariable
    private Timeslot timeslot;

    @PlanningVariable
    private Room room;
}
```

**Caution:** Each additional planning variable multiplies the search space. With N entities, V1 values for variable 1, and V2 values for variable 2, the search space is `(V1 × V2)^N`. Only add a second planning variable if the solver genuinely needs to choose both independently.

For employee scheduling, typically only `employee` is a planning variable. The shift's time and role are fixed inputs.

---

## 7. Score Level Strategy

| Levels | Score type | When to use |
|---|---|---|
| 2 | `HardSoftScore` | Default. Hard = physical/legal. Soft = preferences. |
| 3 | `HardMediumSoftScore` | Overconstrained (§3). Medium = "assign as many as possible." |
| 2 (decimal) | `HardSoftBigDecimalScore` | Fairness/load balancing with rational weights. |
| N | `BendableScore` | Many priority levels (rare). |

**Rule of thumb:** Start with `HardSoftScore`. Switch to `HardMediumSoftScore` only if you need overconstrained planning. Use `HardSoftBigDecimalScore` only if you have `loadBalance` constraints.

**Never use float/double in scores.** They cause score corruption due to floating-point arithmetic. Use `BigDecimal` or scaled `long` instead.

---

## 8. Backup Planning

For resilience against unexpected changes (employee calls in sick), build buffer into the plan:

```java
// Soft constraint: Keep one "spare" employee per time period
Constraint spareEmployee(ConstraintFactory factory) {
    return factory.forEach(Shift.class)
            .groupBy(Shift::getTimePeriod, countDistinct(Shift::getEmployee))
            .filter((period, employeeCount) ->
                    employeeCount >= period.getRequiredEmployees())
            // No spare if all slots are filled by unique employees
            // Reward having MORE employees than required
            .reward(HardSoftScore.ONE_SOFT)
            .asConstraint("Spare employee capacity");
}
```

This creates slack in the schedule so that when an employee calls in sick, another can cover without replanning.

---

## 9. Constraint Weight Strategy

**Don't overthink weights at the start.** Begin with `ONE_HARD` / `ONE_SOFT` for everything. Iterate with stakeholders later.

Guidance:

| Approach | Code | When |
|---|---|---|
| **Fixed weights** | `penalize(HardSoftScore.ONE_HARD)` | Prototype, simple problems |
| **Scaled fixed weights** | `penalize(HardSoftScore.ofSoft(10))` | When some soft constraints are more important |
| **Configurable weights** | `penalizeConfigurable()` + `@ConstraintConfiguration` | When business users need to tune weights via UI |

For configurable weights, see `@ConstraintConfiguration` and `@ConstraintWeight` in the Timefold docs. This enables runtime tuning without code changes.

---

## 10. Time Grain Pattern vs Direct Assignment

Two common ways to model time in scheduling:

### Direct Assignment (recommended for employee scheduling)
Shifts have fixed start/end times. The solver assigns employees to shifts.

```java
class Shift {  // fixed time, variable employee
    LocalDateTime start;
    LocalDateTime end;
    @PlanningVariable Employee employee;
}
```

### Time Grain Pattern (for timetabling/meeting scheduling)
Time is divided into discrete grains. The solver assigns entities to grains.

```java
class Lesson {  // variable time slot and room
    @PlanningVariable Timeslot timeslot;
    @PlanningVariable Room room;
}
```

**For employee scheduling, use Direct Assignment.** The shifts are pre-defined with fixed times; only the employee assignment changes. Time Grain is for problems where the solver also chooses *when* things happen.
