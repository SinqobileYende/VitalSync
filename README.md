# VitalSync System Overview
a console based C# system that simulates live campus operations. Requests flow in while background threads keep the system functional and alerts the moment a rule is broken. 
It behaves like a working clinic front desk. Appointment requests arrive, patients are assigned to consultation rooms and practitioners, and a background process keeps watch over the clinic while the operator works raising alerts the moment something needs attention.

The clinic is all-gender inclusive. It handles general consultations and also runs a dedicated women's-health service line, represented through a service type on each appointment (General, Women's Health, Urgent) rather than as a separate system.

### Core Entities
| Entity | Represents | Key attributes |
|---|---|---|
| **Patient** | Someone needing care | Priority (Normal / Urgent), Status (Waiting / InConsult / Seen) |
| **Practitioner** | The nurse, doctor, or counsellor who sees patients | Role, Availability (Free / Busy) |
| **ConsultRoom** | A consultation room | Capacity, Occupancy |
| **Appointment** | A booking linking a patient, practitioner, and room | Service type, Time, Status (Booked / Completed / Cancelled) |

All four entities inherit from a common abstract base class: ClinicEntity which gives every entity a unique name and ID. 

### Domain rule
A room cannot be booked beyond its capacity
An appointment cannot be assigned without a practitioner status bein free/available.
Urgent and Womens health cases are prioritized in the waiting queue.

## 2. How to Run the Application
 
**Requirements:**  Visual Studio or the `dotnet` command-line tools.
 
**Using Visual Studio**
1. Open `VitalSync.slnx` in Visual Studio.
2. Set `VitalSync` as the startup project (right-click the project → Set as Startup Project).
3. Press **F5** to build and run, or **Ctrl+F5** to run without debugging.
**Using the command line**
1. Open a terminal in the folder containing `VitalSync.csproj`.
2. Run `dotnet run`.
On start-up the application seeds randomised data, launches the background monitor, and displays the main menu. The operator interacts by typing the number next to a menu option and pressing Enter.


## 3. Menu Structure
 
VitalSync - Campus Wellness Clinic Operations
Background monitor: RUNNING
 
PATIENTS
1. Register a patient
2. View all patients
3. Modify a patient
4. Remove a patient
 
PRACTITIONERS
5. Add a practitioner
6. View all practitioners
 
ROOMS
7. Add a consult room
8. View all rooms
 
APPOINTMENTS
9. Book an appointment
10. View all appointments
11. Cancel an appointment
 
REPORTS
12. View waiting queue (by priority)
13. View appointments by service type
14. Save clinic state to file
15. Load clinic state from file
 
0. Exit

## 4. Key Design Decisions

**IDs are read-only from outside.** An entity's `ID` has a private setter. it is set once when the entity is created and can never be changed externally.
**Interfaces are applied only where they are meaningful.** Two behaviour contracts are defined:
- `ISchedulable`: for anything that can be booked into a slot and checked for availability.
- `IMonitorable`: for anything the background monitor inspects, via a `GetReport()` status string.

`ConsultRoom` implements both (it can be booked and it is monitored); `Practitioner` implements `IMonitorable` (its Free/Busy state is watched).

**Descriptions are returned, not printed.** Each entity's `Describe()` method returns a string rather than writing to the console directly. This lets the caller decide what to do with it, show it on screen.

**Starting state is set by the constructor** A new patient always starts as *Waiting*, a new room starts empty, and a new practitioner starts *Free*. These starting values are set inside each constructor rather than being passed in, because they are not the caller's decision.

## 5. Object-Oriented Principles
 
- **Encapsulation** : fields are exposed through properties with controlled access (for example, the read-only `ID`), so an entity cannot be put into an invalid state from outside.
- **Inheritance** : `Patient`, `Practitioner`, `ConsultRoom`, and `Appointment` all derive from `ClinicEntity`, inheriting its `ID` and `Name`.
- **Abstraction** :`ClinicEntity` is an abstract class that cannot be instantiated on its own; it declares an abstract `Describe()` method with no body, plus the two interfaces define behaviour without dictating implementation.
- **Polymorphism** — every entity overrides `Describe()` with its own version, so a single loop over a mixed collection of entities produces the correct description for each type at runtime.

- ## 6. Multithreading
 
VitalSync runs a **background monitor** on its own thread, independently of user input, so the system exhibits genuine concurrent behaviour rather than only appearing to.
 
**What the monitor does.** On a periodic loop, the monitor thread wakes, inspects the shared clinic state through the `IMonitorable` contract (rooms and practitioners), and checks for conditions that warrant attention — for example a room pushed to capacity or a patient who has waited beyond a threshold. When it finds one, it raises the relevant event (see below). All of this happens while the main thread continues to serve the operator's menu, so the interface never blocks.
 
**Why threading is necessary here.** A clinic must be watched continuously; the watching cannot pause every time the operator is reading the menu or typing a command. The monitor therefore has to run on its own thread. This is why the "Background monitor: RUNNING" line appears at the top of the menu — it is a visible signal that the second thread is alive.
 
**Safe execution.** The background thread and the main thread both read and modify shared collections (patients, rooms, appointments). Access to that shared state is synchronised so the two threads never corrupt it and the program never freezes or crashes. _(Document the exact synchronisation mechanism used here once implemented — e.g. `lock` around shared-collection access.)_

### Events and Delegates
 
VitalSync uses custom **events**, delivered through delegates, to represent meaningful changes in the system's state. Events establish a clear publisher–subscriber relationship: the code that detects a change raises an event without needing to know who is listening, and subscribers react automatically.
 
Planned events:
- **AppointmentBooked** — raised when a booking succeeds, so the outcome can be logged and reported.
- **PatientOverdue** — raised by the monitor when a patient has waited beyond the safe threshold.
- **RoomAtCapacity** — raised when a room reaches its capacity limit, so the operator is warned immediately.
Because these events are decoupled from the code that triggers them, the part of the system that detects a problem does not call the subscribers directly — it simply raises the event, and any interested part of the system responds. _(Record the exact delegate/event signatures here once implemented.
 
## 8. Exception Handling
 
The system handles foreseeable failures in a controlled way, using `try`/`catch`/`finally` and its own domain-specific exception types rather than letting the program crash. Planned custom exceptions:
- **RoomAtCapacityException** :thrown when a booking would exceed a room's capacity. The seam for this is already marked in `ConsultRoom.Reserve()`.
- **NoPractitionerAvailableException** : thrown when an appointment is requested but no practitioner is free.
When one of these is thrown, it is caught where it can be reported sensibly, a clear message is shown to the operator, and the system continues running in a consistent state.
 
---
 
## 9. Bonus Feature

- **File I/O** — saving and loading the clinic state (menu options 14 and 15), so a session can be preserved and restored.
- **LINQ** — used to sort the waiting queue by priority and to filter appointments by service type (menu options 12 and 13).

- ---

## Acknowledgements

Portions of this documentation were drafted with AI assistance (Claude) and
reviewed, edited, and verified by the project team.


## 10. Team and Responsibilities
 
| Member | Responsibility |
|---|---|
| Sinqobile Yende | Entities and OOP core: `ClinicEntity`, the four entity classes, and the two interfaces |
| Reitumetse Mdlapo | Menu, and exception handling | LINQ
| Masana Sakala| Events and Delegates
