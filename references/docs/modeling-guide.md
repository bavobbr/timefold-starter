# Timefold Modeling Guide — Java/Quarkus

Reference for all Timefold domain model annotations and patterns.
For constraint definitions, see `constraint-streams-guide.md`.
For Quarkus config and REST patterns, see `quarkus-integration.md`.

## Table of Contents
- [Decision: Problem Fact vs Planning Entity](#decision-problem-fact-vs-planning-entity)
- [@PlanningId](#planningid)
- [@PlanningEntity](#planningentity)
- [@PlanningVariable](#planningvariable)
- [@PlanningListVariable](#planninglistvariable)
- [@PlanningSolution](#planningsolution)
- [@ValueRangeProvider](#valuerangeprovider)
- [@ProblemFactCollectionProperty](#problemfactcollectionproperty)
- [@PlanningEntityCollectionProperty](#planningentitycollectionproperty)
- [@PlanningScore](#planningscore)
- [@PlanningPin](#planningpin)
- [Shadow Variables](#shadow-variables)
- [No-Arg Constructor Requirement](#no-arg-constructor-requirement)
- [Complete Example: Employee Scheduling Domain](#complete-example-employee-scheduling-domain)

---

## Decision: Problem Fact vs Planning Entity

Every domain class falls into one of three categories:

1. **Unrelated** — not referenced by any constraint. Ignore it.
2. **Problem fact** — used by constraints but does NOT change during solving. No Timefold annotations needed. Examples: `Employee`, `Availability`, `Timeslot`, `Room`.
3. **Planning entity** — used by constraints AND changes during solving. Annotated with `@PlanningEntity`. The fields that the solver changes are **planning variables**.

**The key question: "What class has a field I want the solver to fill in for me?"** That class is the planning entity. In employee scheduling, a `Shift` has an `employee` field that starts as `null` and gets assigned by the solver — so `Shift` is the planning entity.

> **Important:** Choose the model where the number of planning entities is fixed. In employee scheduling, the number of shifts is known in advance (fixed), but the number of shifts per employee is not. Therefore `Shift` is the entity and `Employee` is the value — not the other way around.

---

## @PlanningId

```java
import ai.timefold.solver.core.api.domain.lookup.PlanningId;
```

Marks the unique identifier field on both problem facts and planning entities. Required for multi-threaded solving, real-time planning, and `SolutionManager.analyze()`.

```java
@PlanningId
private String id;
```

Requirements:
- Must be unique within that class (different classes can share the same ID value)
- Must implement `hashCode()` and `equals()` (String, Long, Integer, UUID all work)
- Must never be `null` when `Solver.solve()` is called

---

## @PlanningEntity

```java
import ai.timefold.solver.core.api.domain.entity.PlanningEntity;
```

Marks a class whose instances change during solving. Must have at least one `@PlanningVariable` field and a **public no-arg constructor**.

```java
@PlanningEntity
public class Shift {

    @PlanningId
    private String id;

    // Problem properties — do NOT change during solving
    private LocalDateTime start;
    private LocalDateTime end;
    private String location;
    private String requiredSkill;

    // Planning variable — the solver assigns this
    @PlanningVariable
    private Employee employee;

    // REQUIRED: no-arg constructor
    public Shift() {
    }

    public Shift(String id, LocalDateTime start, LocalDateTime end,
                 String location, String requiredSkill) {
        this.id = id;
        this.start = start;
        this.end = end;
        this.location = location;
        this.requiredSkill = requiredSkill;
    }

    // Getters and setters for ALL fields (required by Timefold)
    // ...
}
```

### Optional: Pinning filter

To prevent the solver from changing certain entities (e.g., already-published shifts), use a pinning filter:

```java
@PlanningEntity(pinningFilter = ShiftPinningFilter.class)
public class Shift { ... }
```

See [@PlanningPin](#planningpin) for the modern boolean-field alternative.

### Optional: Difficulty comparator

Helps Construction Heuristics assign harder entities first:

```java
@PlanningEntity(difficultyComparatorClass = ShiftDifficultyComparator.class)
public class Shift { ... }
```

The comparator should order entities ascending (easy → hard). Do not reference planning variable values in the comparator.

---

## @PlanningVariable

```java
import ai.timefold.solver.core.api.domain.variable.PlanningVariable;
```

Marks a field on a `@PlanningEntity` whose value the solver assigns. The solver picks values from a `@ValueRangeProvider` that returns the same type.

```java
@PlanningVariable
private Employee employee;
```

The field must have both a getter and setter. It starts as `null` (unassigned) and the solver fills it in.

### Type matching

Timefold automatically connects `@PlanningVariable` to `@ValueRangeProvider` by matching the Java type. If `employee` is of type `Employee`, Timefold looks for a `@ValueRangeProvider` returning `List<Employee>`.

### Allowing unassigned values (overconstrained planning)

By default, every planning variable must be assigned after solving. For overconstrained problems (more shifts than employees can fill), allow `null`:

```java
@PlanningVariable(allowsUnassigned = true)
private Employee employee;
```

> **Important:** When using `allowsUnassigned = true`, you MUST add a soft constraint that penalizes unassigned shifts, or the solver may leave everything unassigned (since that trivially satisfies all hard constraints).

Example constraint:
```java
Constraint unassignedShift(ConstraintFactory constraintFactory) {
    return constraintFactory.forEach(Shift.class)
            .filter(shift -> shift.getEmployee() == null)
            .penalize(HardSoftScore.ONE_SOFT, shift -> 2) // Higher weight
            .asConstraint("Unassigned shift");
}
```

---

## @PlanningListVariable

```java
import ai.timefold.solver.core.api.domain.variable.PlanningListVariable;
```

For problems where an entity contains an **ordered list** of values (e.g., vehicle routing: a Vehicle has an ordered list of Visits). The solver decides both the assignment and the ordering.

```java
@PlanningEntity
public class Vehicle {
    @PlanningListVariable
    private List<Visit> visits = new ArrayList<>();
}
```

> **For employee scheduling, use `@PlanningVariable`, NOT `@PlanningListVariable`.** List variables are for routing-style problems where a single entity "owns" an ordered sequence. In employee scheduling, you assign one employee per shift — that's a basic `@PlanningVariable`.

When to use which:

| Pattern | Annotation | Example |
|---------|-----------|---------|
| Assign one value to each entity | `@PlanningVariable` | Shift → Employee |
| Assign an ordered list of values to each entity | `@PlanningListVariable` | Vehicle → List\<Visit\> |

---

## @PlanningSolution

```java
import ai.timefold.solver.core.api.domain.solution.PlanningSolution;
```

Marks the container class that holds the entire problem and its solution. It has four required component types:

1. `@PlanningEntityCollectionProperty` — all entity instances
2. `@ProblemFactCollectionProperty` — all problem fact instances
3. `@ValueRangeProvider` — possible values for planning variables
4. `@PlanningScore` — the score field

```java
@PlanningSolution
public class EmployeeSchedule {

    @ValueRangeProvider
    @ProblemFactCollectionProperty
    private List<Employee> employees;

    @ProblemFactCollectionProperty
    private List<Availability> availabilities;

    @PlanningEntityCollectionProperty
    private List<Shift> shifts;

    @PlanningScore
    private HardSoftScore score;

    // REQUIRED: no-arg constructor
    public EmployeeSchedule() {
    }

    public EmployeeSchedule(List<Employee> employees,
                            List<Availability> availabilities,
                            List<Shift> shifts) {
        this.employees = employees;
        this.availabilities = availabilities;
        this.shifts = shifts;
    }

    // Getters and setters for ALL fields
    // ...
}
```

> **Note:** `@ValueRangeProvider` and `@ProblemFactCollectionProperty` can be on the same field. The `employees` field above serves double duty: it provides the value range for `Shift.employee` (via type matching) AND registers employees as problem facts accessible in constraints.

---

## @ValueRangeProvider

```java
import ai.timefold.solver.core.api.domain.valuerange.ValueRangeProvider;
```

Declares a collection of possible values for a `@PlanningVariable`. Placed on a field or getter in the `@PlanningSolution` class.

```java
@ValueRangeProvider
private List<Employee> employees;
```

Timefold matches the value range to the planning variable by type: `List<Employee>` matches `@PlanningVariable private Employee employee`.

If you have multiple value range providers of the same type (rare), use the `id` attribute to disambiguate.

---

## @ProblemFactCollectionProperty

```java
import ai.timefold.solver.core.api.domain.solution.ProblemFactCollectionProperty;
```

Registers a collection of problem facts so they can be queried in constraint streams via `forEach()` and `join()`.

```java
@ProblemFactCollectionProperty
private List<Availability> availabilities;
```

Every class you reference in a constraint (`forEach`, `join`, `ifExists`) must be registered either as a `@ProblemFactCollectionProperty` or as a `@PlanningEntityCollectionProperty`.

For a singleton problem fact (not a collection), use `@ProblemFactProperty` instead.

---

## @PlanningEntityCollectionProperty

```java
import ai.timefold.solver.core.api.domain.solution.PlanningEntityCollectionProperty;
```

Registers the collection of planning entities. The solver iterates over these to assign planning variables.

```java
@PlanningEntityCollectionProperty
private List<Shift> shifts;
```

This list must contain ALL entity instances. Do not filter it.

---

## @PlanningScore

```java
import ai.timefold.solver.core.api.domain.solution.PlanningScore;
import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;
```

The score field where Timefold stores the quality of the current solution. Starts as `null`; Timefold fills it in.

```java
@PlanningScore
private HardSoftScore score;
```

Score types (most common first):

| Type | Levels | Use when |
|------|--------|----------|
| `HardSoftScore` | 2 (hard, soft) | Most scheduling problems. Hard = must not break. Soft = prefer not to break. |
| `HardMediumSoftScore` | 3 | Need a middle tier (e.g., medium = staffing coverage). |
| `SimpleScore` | 1 | Only hard constraints, no preferences. |
| `BendableScore` | N | Custom number of hard/soft levels. |

**Default choice: `HardSoftScore`.** Use this unless you have a specific reason for more levels.

---

## @PlanningPin

```java
import ai.timefold.solver.core.api.domain.entity.PlanningPin;
```

A boolean field on a `@PlanningEntity`. When `true`, the solver will not change this entity's planning variables. Used for continuous planning (locking already-published assignments).

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

When `pinned == true`, the solver treats this Shift as immutable. The employee assignment stays as-is.

> **Note:** `@PlanningPin` is the modern approach. The older `pinningFilter` on `@PlanningEntity` still works but is more verbose. Prefer `@PlanningPin`.

For the continuous planning pattern (published vs draft shifts), see `continuous-planning.md`.

---

## Shadow Variables

A shadow variable is derived from genuine planning variables. It's automatically updated when genuine variables change. Employee scheduling typically does NOT need shadow variables — they're mainly used in routing problems.

### @InverseRelationShadowVariable

Creates a bi-directional relationship. If `Shift.employee` is a `@PlanningVariable`, you can add an inverse collection on `Employee`:

```java
import ai.timefold.solver.core.api.domain.variable.InverseRelationShadowVariable;

@PlanningEntity  // Must be annotated even though it's a "shadow" entity
public class Employee {

    @InverseRelationShadowVariable(sourceVariableName = "employee")
    private List<Shift> shifts = new ArrayList<>();
}
```

The `shifts` list is auto-maintained by Timefold. The `sourceVariableName` must match the field name on the entity that references this class.

> **Important:** The shadow collection must be initialized (not null) and mutable.

### Other shadow variable types (routing-focused)

These are mainly used with `@PlanningListVariable` for routing problems:

- `@PreviousElementShadowVariable` — the previous element in a list variable
- `@NextElementShadowVariable` — the next element in a list variable
- `@AnchorShadowVariable` — the anchor (e.g., vehicle/depot) of a chain
- `@ShadowVariable(supplierClass = ...)` — custom shadow via a `VariableListener`

See the vehicle-routing quickstart (`examples/vehicle-routing/`) for working examples.

---

## No-Arg Constructor Requirement

Every class annotated with `@PlanningEntity` or `@PlanningSolution` **MUST** have a public no-arg constructor. Timefold uses it internally for cloning solutions during solving.

```java
// ✅ Correct
@PlanningEntity
public class Shift {
    public Shift() {}                      // no-arg constructor
    public Shift(String id, ...) { ... }   // convenience constructor
}

// ❌ WRONG — will throw exception at startup
@PlanningEntity
public class Shift {
    public Shift(String id, ...) { ... }   // no no-arg constructor!
}
```

Problem fact classes (without `@PlanningEntity`) do NOT need a no-arg constructor.

---

## Complete Example: Employee Scheduling Domain

### Employee.java — Problem Fact

```java
package org.acme.employeescheduling.domain;

import java.util.Set;
import ai.timefold.solver.core.api.domain.lookup.PlanningId;

// No @PlanningEntity — Employee is a problem fact (does not change)
public class Employee {

    @PlanningId
    private String name;

    private Set<String> skills;

    public Employee() {
    }

    public Employee(String name, Set<String> skills) {
        this.name = name;
        this.skills = skills;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Set<String> getSkills() {
        return skills;
    }

    public void setSkills(Set<String> skills) {
        this.skills = skills;
    }

    @Override
    public String toString() {
        return name;
    }
}
```

### Availability.java — Problem Fact

```java
package org.acme.employeescheduling.domain;

import java.time.LocalDate;

public class Availability {

    private Employee employee;
    private LocalDate date;
    private AvailabilityType availabilityType;

    public Availability() {
    }

    public Availability(Employee employee, LocalDate date,
                        AvailabilityType availabilityType) {
        this.employee = employee;
        this.date = date;
        this.availabilityType = availabilityType;
    }

    // Getters and setters ...

    public Employee getEmployee() { return employee; }
    public void setEmployee(Employee employee) { this.employee = employee; }
    public LocalDate getDate() { return date; }
    public void setDate(LocalDate date) { this.date = date; }
    public AvailabilityType getAvailabilityType() { return availabilityType; }
    public void setAvailabilityType(AvailabilityType availabilityType) {
        this.availabilityType = availabilityType;
    }
}
```

### AvailabilityType.java — Enum

```java
package org.acme.employeescheduling.domain;

public enum AvailabilityType {
    DESIRED,
    UNDESIRED,
    UNAVAILABLE
}
```

### Shift.java — Planning Entity

```java
package org.acme.employeescheduling.domain;

import java.time.LocalDateTime;

import ai.timefold.solver.core.api.domain.entity.PlanningEntity;
import ai.timefold.solver.core.api.domain.lookup.PlanningId;
import ai.timefold.solver.core.api.domain.variable.PlanningVariable;

@PlanningEntity
public class Shift {

    @PlanningId
    private String id;

    private LocalDateTime start;
    private LocalDateTime end;
    private String location;
    private String requiredSkill;

    // The solver assigns this field — it's the planning variable
    @PlanningVariable
    private Employee employee;

    // REQUIRED: no-arg constructor
    public Shift() {
    }

    public Shift(String id, LocalDateTime start, LocalDateTime end,
                 String location, String requiredSkill) {
        this.id = id;
        this.start = start;
        this.end = end;
        this.location = location;
        this.requiredSkill = requiredSkill;
    }

    // Constructor for tests (with pre-assigned employee)
    public Shift(String id, LocalDateTime start, LocalDateTime end,
                 String location, String requiredSkill, Employee employee) {
        this(id, start, end, location, requiredSkill);
        this.employee = employee;
    }

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public LocalDateTime getStart() { return start; }
    public void setStart(LocalDateTime start) { this.start = start; }
    public LocalDateTime getEnd() { return end; }
    public void setEnd(LocalDateTime end) { this.end = end; }
    public String getLocation() { return location; }
    public void setLocation(String location) { this.location = location; }
    public String getRequiredSkill() { return requiredSkill; }
    public void setRequiredSkill(String requiredSkill) { this.requiredSkill = requiredSkill; }
    public Employee getEmployee() { return employee; }
    public void setEmployee(Employee employee) { this.employee = employee; }

    @Override
    public String toString() {
        return location + " " + start + "-" + end;
    }
}
```

### EmployeeSchedule.java — Planning Solution

```java
package org.acme.employeescheduling.domain;

import java.util.List;

import ai.timefold.solver.core.api.domain.solution.PlanningEntityCollectionProperty;
import ai.timefold.solver.core.api.domain.solution.PlanningScore;
import ai.timefold.solver.core.api.domain.solution.PlanningSolution;
import ai.timefold.solver.core.api.domain.solution.ProblemFactCollectionProperty;
import ai.timefold.solver.core.api.domain.valuerange.ValueRangeProvider;
import ai.timefold.solver.core.api.score.buildin.hardsoft.HardSoftScore;

@PlanningSolution
public class EmployeeSchedule {

    // Provides the value range for Shift.employee (type = Employee)
    // AND registers employees as queryable problem facts
    @ValueRangeProvider
    @ProblemFactCollectionProperty
    private List<Employee> employees;

    // Registers availabilities as queryable problem facts
    @ProblemFactCollectionProperty
    private List<Availability> availabilities;

    // Registers all shifts as planning entities for the solver
    @PlanningEntityCollectionProperty
    private List<Shift> shifts;

    // Timefold fills this in — the quality of the current solution
    @PlanningScore
    private HardSoftScore score;

    // REQUIRED: no-arg constructor
    public EmployeeSchedule() {
    }

    public EmployeeSchedule(List<Employee> employees,
                            List<Availability> availabilities,
                            List<Shift> shifts) {
        this.employees = employees;
        this.availabilities = availabilities;
        this.shifts = shifts;
    }

    public List<Employee> getEmployees() { return employees; }
    public void setEmployees(List<Employee> employees) { this.employees = employees; }
    public List<Availability> getAvailabilities() { return availabilities; }
    public void setAvailabilities(List<Availability> availabilities) {
        this.availabilities = availabilities;
    }
    public List<Shift> getShifts() { return shifts; }
    public void setShifts(List<Shift> shifts) { this.shifts = shifts; }
    public HardSoftScore getScore() { return score; }
    public void setScore(HardSoftScore score) { this.score = score; }
}
```

### How they connect

```
EmployeeSchedule (@PlanningSolution)
├── employees: List<Employee>         ← @ValueRangeProvider + @ProblemFactCollectionProperty
├── availabilities: List<Availability> ← @ProblemFactCollectionProperty
├── shifts: List<Shift>               ← @PlanningEntityCollectionProperty
│   └── each Shift has:
│       ├── start, end, location, requiredSkill  (problem properties — fixed)
│       └── employee: Employee                    (@PlanningVariable — solver assigns)
└── score: HardSoftScore              ← @PlanningScore (solver fills in)
```

The solver picks `Employee` instances from the `employees` list and assigns them to `Shift.employee` fields, evaluating the constraints after each change to find the best score.
