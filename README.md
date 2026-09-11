# Campus Equipment Borrowing System

Laboratory Activity 1 — From Requirements to Application Structure
(ITSD 81 – Desktop Application Development)

No database and no graphical user interface are implemented in this
activity. The goal is the architectural foundation those pieces will
plug into later.

## 1. Solution Structure

| Project | Purpose |
|---|---|
| **EquipmentBorrowing.Domain** | The concepts and rules that belong to the problem itself, independent of any application or technology: `Student`, `Equipment`, `Borrowing`, `BorrowingStatus`. These classes protect their own invariants (e.g. `Equipment` refuses to be borrowed twice in a row) but know nothing about databases, UIs, or use cases. |
| **EquipmentBorrowing.Application** | The use cases the system performs: `BorrowEquipmentService`, `ReturnEquipmentService`, `FindAvailableEquipmentService`. This layer coordinates Domain objects and repository interfaces to satisfy a request. It also defines the repository interfaces (`IStudentRepository`, `IEquipmentRepository`, `IBorrowingRepository`) — it depends only on Domain. |
| **EquipmentBorrowing.Infrastructure** | Concrete, technology-specific implementations of the Application layer's interfaces. For this activity: `InMemoryStudentRepository`, `InMemoryEquipmentRepository`, `InMemoryBorrowingRepository`. A future SQLite implementation would live here too, alongside the in-memory one. |
| **EquipmentBorrowing.Console** | A throwaway demonstration program. Wires the layers together with manual constructor injection and prints a successful and a failing borrow attempt. This is the project a future Avalonia UI would effectively replace. |
| **EquipmentBorrowing.Tests** | Automated tests for `BorrowEquipmentService`, covering the success case and each business-rule failure. |

## 2. Dependency Direction

```
   Console (today) / Avalonia UI (later)
                │
                ▼
           Application  ◄────────┐
                │                │
                ▼                │
             Domain              │
                                 │
           Infrastructure ───────┘
      (implements Application's interfaces)
```

- **Domain** depends on nothing else in the solution.
- **Application** depends only on **Domain**. It defines repository
  *interfaces* but never implements them.
- **Infrastructure** depends on **Application** (to implement its
  interfaces) and, transitively, on **Domain**.
- **Console** (and later, the Avalonia UI project) depends on
  **Application** to invoke use cases, and on **Infrastructure** only
  at startup, to construct the concrete repository objects that get
  injected into the application services.

The dependency arrow between Application and Infrastructure points
*inward*: Infrastructure depends on Application's interfaces, not the
other way around. This is what allows the storage technology to change
without the business logic changing.

## 3. Use Case Mapping

**Actor:** Student

**Use Case:** Borrow Equipment

**Application Service:** `BorrowEquipmentService`

**Domain Objects Used:** `Student`, `Equipment`, `Borrowing`, `BorrowingStatus`

**Repository Interfaces Used:** `IStudentRepository`, `IEquipmentRepository`, `IBorrowingRepository`

**Infrastructure Implementations Used:** `InMemoryStudentRepository`, `InMemoryEquipmentRepository`, `InMemoryBorrowingRepository`

(Two additional use cases — Return Equipment and Find Available
Equipment — are implemented the same way, as `ReturnEquipmentService`
and `FindAvailableEquipmentService`, to demonstrate the pattern holds
beyond a single service.)

## 4. Reflection

**Why should the application service depend on a repository interface
instead of directly depending on a database implementation?**
Because the business rule being expressed — "a student may borrow
equipment only if eligible, available, and under their limit" — has
nothing to do with *how* that data is stored. Coding directly against
SQLite (or any concrete technology) would mean every future change to
storage, or every unit test, would need a real database. Depending on
an interface lets the same service run against an in-memory
implementation in tests and a real database in production, unchanged.

**Which parts of your current solution could remain unchanged if
SQLite were added later?**
All of Domain and Application. Only Infrastructure would grow a new
`SqliteEquipmentRepository`, `SqliteStudentRepository`, and
`SqliteBorrowingRepository` implementing the exact same interfaces the
in-memory versions implement today. `Program.cs` (or the composition
root of the future UI) would change one line — which concrete
repository gets constructed — and nothing else.

**Which project would eventually contain Avalonia Views?**
A new UI project (e.g. `EquipmentBorrowing.Desktop`), sitting at the
same layer as `EquipmentBorrowing.Console` does now — depending on
Application to call use cases, and on Infrastructure only to wire up
concrete repositories at startup. It would not belong inside Domain,
Application, or Infrastructure.

**Should an Avalonia button directly execute database queries? Why or
why not?**
No. A button's click handler should call an application service (e.g.
`BorrowEquipmentService.ExecuteAsync`), the same way `Program.cs` does
here. If the UI executed SQL directly, the borrowing rules (eligibility,
availability, borrowing limit) would either be duplicated in the UI
layer or bypassed entirely, and the same logic could no longer be
reused by, say, a future mobile app or an automated test.

**What part of your implementation represents the actual business
operation requested by the actor?**
`BorrowEquipmentService.ExecuteAsync` — it is the single place where
"a student wants to borrow a piece of equipment" is actually decided
and carried out. Everything else in the solution exists to support
that method: Domain provides the vocabulary and invariants it works
with, and Infrastructure provides the data it needs.

---

# Laboratory Activity 2 — Extending the Application with Avalonia UI and MVVM

This section extends, rather than replaces, the explanation above. Every
project and class described in Laboratory Activity 1 is unchanged in
behavior; this activity only adds a presentation layer on top.

## 5. Desktop Project

`EquipmentBorrowing.Desktop` is an Avalonia application that gives the
existing architecture a graphical interface. Its responsibilities:

- displaying equipment and active borrowings (Views + ViewModels);
- collecting user input (student, equipment, expected return date);
- invoking the existing `BorrowEquipmentService` and
  `ReturnEquipmentService` through ViewModels;
- acting as the application's **composition root** (`App.axaml.cs`),
  the single place that constructs the concrete `InMemory*Repository`
  instances and registers everything with a
  `Microsoft.Extensions.DependencyInjection` container.

It references `EquipmentBorrowing.Application` (to call use cases) and
`EquipmentBorrowing.Infrastructure` (to construct repositories at
startup). It does **not** reference `EquipmentBorrowing.Domain`
directly — Domain types flow into the Desktop project only through
Application's public surface. Just as importantly, `Domain` and
`Application` do not reference Avalonia at all; nothing about them
changed to accommodate the UI.

Two small, documented additions were made to the existing repository
interfaces so the UI could list data it previously had no reason to
list: `IStudentRepository.GetAllAsync()` and
`IBorrowingRepository.GetAllActiveAsync()`. Both are plain queries —
they add no borrowing rules and change no existing method.

## 6. Updated Architecture

```
                 Avalonia View
                       │
                       │ Binding / Command
                       ▼
                  ViewModel
                       │
                       │ Application Operation
                       ▼
               Application Service
                       │
                       ├──────────► Domain
                       │
                       ▼
              Repository Interface
                       ▲
                       │
           Infrastructure Implementation
```

- **View** (`EquipmentView.axaml`, `BorrowingsView.axaml`,
  `MainWindow.axaml`) — XAML layout, controls, bindings, and styles
  only. No business logic, no direct repository access.
- **ViewModel** (`EquipmentViewModel`, `BorrowingsViewModel`,
  `MainWindowViewModel`) — presentation state, commands, selected
  values, observable collections, and calls into Application services.
  Implemented with `CommunityToolkit.Mvvm`'s `ObservableObject`,
  `[ObservableProperty]`, and `[RelayCommand]`.
- **Application Service** (`BorrowEquipmentService`,
  `ReturnEquipmentService`, `FindAvailableEquipmentService`) —
  unchanged from Laboratory Activity 1. Owns the business workflow and
  coordination of repositories.
- **Domain** (`Student`, `Equipment`, `Borrowing`,
  `BorrowingStatus`) — unchanged. Owns its own invariants.
- **Repository Interface / Infrastructure Implementation** —
  unchanged, aside from the two additive query methods noted above.

## 7. Borrow Equipment Flow

1. The user selects a student and a piece of equipment in
   `EquipmentView`, and picks an expected return date, then presses
   **Borrow Equipment**. The `Button.Command` binding invokes
   `EquipmentViewModel.BorrowCommand` (an `AsyncRelayCommand`
   generated from the `[RelayCommand]`-decorated `BorrowAsync` method).
2. `BorrowAsync` first checks only that a student, equipment, and a
   non-past date were actually selected — presentation validation. It
   does **not** check eligibility, availability, or borrowing limits.
3. It calls `await _borrowEquipmentService.ExecuteAsync(studentId,
   equipmentId, expectedReturnDate)` — the same, unmodified Application
   service from Laboratory Activity 1.
4. `BorrowEquipmentService` performs the real business validation
   (student exists and is eligible, equipment exists and is available,
   borrowing limit not exceeded), and on success mutates the `Equipment`
   and creates a `Borrowing` through the repositories.
5. The `Result<Borrowing>` returned flows back to the ViewModel, which
   sets `StatusMessage` for success or failure, reloads its own
   equipment/student lists, and broadcasts a
   `BorrowingStateChangedMessage` via `WeakReferenceMessenger` so
   `BorrowingsViewModel` refreshes its list too, even though the two
   ViewModels hold no direct reference to each other.
6. The bound `ListBox`, `TextBlock`, and status message in the View
   update automatically because `ObservableCollection` and
   `[ObservableProperty]` raise the necessary change notifications —
   no manual UI-refresh code is written anywhere.

## 8. Return Equipment Flow

1. The user selects an active borrowing in `BorrowingsView` and
   presses **Return Equipment**, invoking
   `BorrowingsViewModel.ReturnCommand`.
2. `ReturnAsync` checks only that a borrowing was actually selected,
   then calls `await _returnEquipmentService.ExecuteAsync(equipmentId)`
   — again, the same unmodified Application service.
3. `ReturnEquipmentService` locates the active borrowing, marks it
   `Returned`, and marks the equipment available again through the
   repositories — all business logic, none of it in the ViewModel.
4. On success, `BorrowingsViewModel` reloads its list and broadcasts
   `BorrowingStateChangedMessage`, which `EquipmentViewModel` receives
   and uses to reload — so the equipment the student just returned
   shows as "Available" again immediately, without navigating away and
   back.

## 9. Architectural Reflection

**Why should the View not call a repository directly?**
The View's only job is to display state and forward user input. If it
called a repository directly, it would need to know about
`IEquipmentRepository`, `Result<T>`, and the borrowing rules well
enough to interpret failures — logic that belongs in the Application
layer and would then be duplicated (or contradicted) if a second UI,
or a test, needed the same operation.

**Why should business rules not be implemented in the ViewModel?**
The ViewModel exists to adapt Application services to a specific UI
technology (Avalonia's binding and command model). Putting rules like
"equipment must be available" there would tie business logic to
Avalonia, make it untestable without spinning up UI infrastructure,
and duplicate logic `BorrowEquipmentService` already owns and already
has tests for.

**What is the responsibility of the ViewModel?**
To hold presentation state (selected items, form values, status
messages), expose commands the View can bind to, perform presentation
validation (was something selected at all?), and translate a user
action into a single call to the appropriate Application service —
then translate that service's `Result<T>` back into something the View
can display.

**Why can the existing Application layer work without knowing that
Avalonia is being used?**
Because `BorrowEquipmentService` and `ReturnEquipmentService` depend
only on repository interfaces and Domain types, both defined inside
the solution with no reference to any UI framework. Avalonia sits
entirely on the other side of that boundary, in the Desktop project,
calling into Application the same way `EquipmentBorrowing.Console` did
in Laboratory Activity 1.

**What advantage is gained from registering dependencies in one
composition point?**
`App.axaml.cs` is the only place that knows which concrete repository
and service implementations exist. Every other class asks only for an
interface or a concrete Application-layer type through its
constructor. That means the wiring can change — for example, swapping
which repository implementation is registered — by editing one method,
`ConfigureServices`, without touching any ViewModel, View, or
Application service.

**If the in-memory repository were replaced by SQLite later, which
parts of the current interface should remain largely unchanged?**
All of it. The Views, ViewModels, and Application services depend only
on `IStudentRepository`, `IEquipmentRepository`, and
`IBorrowingRepository` — never on `InMemory*Repository` directly.
Replacing the in-memory classes with `Sqlite*Repository`
implementations of the same interfaces, and changing the two lines in
`App.axaml.cs`'s `ConfigureServices` that construct them, is the only
change required.
