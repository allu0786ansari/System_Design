# Hevo Data SDE I — Complete Study Notes (Python)

**Contents**
1. Object-Oriented Programming (OOP)
2. Low-Level Design (LLD) & Machine Coding
3. System Design Basics
4. SQL & DBMS
5. Operating Systems & Concurrency Basics
6. Behavioral / HR Round

---

# Object-Oriented Programming (OOP) — Complete Interview Notes (Python)

> Goal: understand each concept well enough to (a) explain it in an interview,
> (b) recognize it in a design problem, and (c) implement it correctly under
> time pressure in a machine-coding round.

---

## 1. What is OOP and why does it exist?

**Definition:** Object-Oriented Programming is a way of structuring code around
**objects** — bundles of *data* (attributes) and *behavior* (methods) that
model real-world or logical entities — instead of structuring code purely
around functions acting on raw data.

**Why it exists (the problem it solves):**
Before OOP became common, large programs were written as long lists of
functions operating on loosely related data (procedural style). As programs
grew, this became hard to maintain:
- Data could be changed from anywhere, with no clear owner → bugs.
- Related data and logic were scattered → hard to find, hard to reuse.
- No natural way to model "a Car has an Engine" or "a Dog is an Animal".

OOP fixes this by giving code a shape that matches how we think about the
real world: things (objects) that have properties and can do things.

**Effect on real projects:**
- Easier to reason about: "a `Connector` object knows how to `extract()` and
  `load()` data" is easier to hold in your head than a pile of functions.
- Easier to extend: add a new type of object without rewriting old code.
- Easier to test: objects can be tested in isolation.
- Easier to collaborate: teams can own different classes/objects.

**A class vs an object:**
- A **class** is a blueprint/template (e.g., `Car`).
- An **object** (or **instance**) is a concrete thing built from that
  blueprint (e.g., `my_red_honda_civic`).

```python
class Car:
    def __init__(self, brand: str, color: str):
        self.brand = brand
        self.color = color

    def honk(self) -> str:
        return f"{self.brand} says beep!"

my_car = Car("Honda", "Red")   # my_car is an *object* of class Car
print(my_car.honk())           # "Honda says beep!"
```

---

## 2. The Four Pillars of OOP

Every OOP interview question eventually touches these four ideas.

### 2.1 Encapsulation

**What it is:** Bundling data (attributes) and the methods that operate on
that data into a single unit (a class), and **restricting direct access** to
some of that data from outside the class. The internal state is only changed
through a controlled interface (methods/properties).

**Why it is used:**
- Prevents external code from putting an object into an invalid state.
- Hides internal implementation details so they can change later without
  breaking code that uses the class (this is the real payoff).
- Creates a clear "contract" — other developers only need to know the public
  methods, not the internal mess.

**Effect if you don't use it:** Any part of a large codebase could directly
mutate an object's internals, causing bugs that are very hard to trace
because "the bug could be anywhere that touches this field."

**Implementation details in Python:**
Python doesn't have true `private` like Java/C++. It uses **naming
conventions**:
- `self.name` → public, anyone can access.
- `self._name` → "protected" by convention — internal use, but still
  accessible (a signal, not a lock).
- `self.__name` → "private" — Python performs **name mangling**
  (`_ClassName__name`) to make accidental access harder, not impossible.
- The idiomatic Python way to control access is the **`@property`** decorator,
  which lets you expose a controlled getter/setter while the calling code
  still uses simple attribute syntax.

```python
class BankAccount:
    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner
        self.__balance = balance          # private-by-convention

    @property
    def balance(self) -> float:
        return self.__balance

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self.__balance += amount

    def withdraw(self, amount: float) -> None:
        if amount > self.__balance:
            raise ValueError("Insufficient funds")
        self.__balance -= amount

acc = BankAccount("Allaudin", 1000)
acc.deposit(500)
print(acc.balance)      # 1500  (read-only access via property)
# acc.balance = 99999   # AttributeError: no setter defined -> protects invariants
```

**Real-world example:** A `BankAccount` class should never let outside code do
`account.balance = -500`. All changes go through `deposit()`/`withdraw()`,
which enforce rules (no negative balances, no overdrafts). This is exactly
why encapsulation matters — it protects business rules (invariants).

---

### 2.2 Abstraction

**What it is:** Hiding *complex implementation details* and exposing only the
*essential features/behavior* through a simple interface. Where encapsulation
is about **restricting access**, abstraction is about **reducing complexity**
by hiding "how" and showing only "what."

**Why it is used:**
- Lets the user of a class focus on *what* it does, not *how* it does it.
- Allows the internal implementation to be swapped out completely as long as
  the interface (the "what") stays the same.

**Effect:** Reduces cognitive load. A driver uses a steering wheel and pedals
(the abstraction) without knowing how the engine's combustion cycle works
(the implementation).

**Implementation details in Python:**
Abstraction is usually implemented using **abstract base classes (ABC)** from
the `abc` module. An abstract class defines *what* methods a subclass must
implement, without saying *how*.

```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> None:
        """Every payment method must implement this."""
        raise NotImplementedError

class CreditCardPayment(PaymentMethod):
    def pay(self, amount: float) -> None:
        print(f"Charged ${amount} to credit card")

class UpiPayment(PaymentMethod):
    def pay(self, amount: float) -> None:
        print(f"Paid ${amount} via UPI")

def checkout(method: PaymentMethod, amount: float) -> None:
    # checkout() doesn't care HOW payment happens, only THAT it can pay()
    method.pay(amount)

checkout(CreditCardPayment(), 250)
checkout(UpiPayment(), 100)
```

You cannot do `PaymentMethod()` directly — Python raises
`TypeError: Can't instantiate abstract class`. This forces every subclass to
implement `pay()`, guaranteeing the contract.

**Real-world example:** A `DataConnector` interface in a data-pipeline tool
(relevant to Hevo!) might define `extract()` and `load()` as abstract
methods. `SalesforceConnector`, `MySQLConnector`, `S3Connector` all implement
these differently, but the pipeline engine that calls them only needs to know
the abstraction: "every connector can extract and load."

---

### 2.3 Inheritance

**What it is:** A mechanism where a new class (**child/subclass**) derives
attributes and methods from an existing class (**parent/superclass**),
allowing code reuse and the modeling of "is-a" relationships (a `Dog` **is an**
`Animal`).

**Why it is used:**
- Avoids duplicating code across similar classes.
- Models natural hierarchies.
- Enables **polymorphism** (see below) — code can work with the parent type
  and automatically work correctly with any child type.

**Effect:** Reduces duplication, but **overuse causes tight coupling** — a
change in the parent class can unexpectedly break every child class. This is
why "favor composition over inheritance" is a well-known design principle
(see Section 4).

**Implementation details in Python:**
```python
class Animal:
    def __init__(self, name: str):
        self.name = name

    def make_sound(self) -> str:
        return "..."

    def describe(self) -> str:
        return f"{self.name} says {self.make_sound()}"

class Dog(Animal):
    def make_sound(self) -> str:      # method overriding
        return "Woof!"

class Cat(Animal):
    def make_sound(self) -> str:
        return "Meow!"

d = Dog("Rex")
print(d.describe())    # "Rex says Woof!"
```

Key mechanics to know for interviews:
- `super().__init__(...)` calls the parent's constructor — very common in
  machine coding rounds to avoid duplicating setup logic.
- Python supports **multiple inheritance** (`class C(A, B):`), resolved via
  **MRO (Method Resolution Order)**, computed with the C3 linearization
  algorithm. You can check it with `ClassName.__mro__`.
- **"is-a" test:** Only use inheritance when the relationship is genuinely
  "is-a." A `Square` is **not** safely a subclass of `Rectangle` in terms of
  behavior (this violates the Liskov Substitution Principle — see Section 4).

**Real-world example:** In a data-pipeline system, a base class `Source`
might have common logic (retry handling, logging), and `MySQLSource`,
`PostgresSource`, `KafkaSource` inherit from it, overriding only the
connection-specific parts.

---

### 2.4 Polymorphism

**What it is:** "Poly" (many) + "morph" (forms). The ability for different
classes to be treated through a **common interface**, where each class
provides its **own specific implementation** of the same method call.

**Why it is used:**
- Lets you write code that works with a general type (e.g., `Animal`) without
  knowing the exact subtype (`Dog` or `Cat`) at compile/write time.
- Makes systems extensible: you can add a new subclass, and existing code
  that calls the common method automatically works with it — **no changes
  needed to the calling code**.

**Effect:** This is the concept most responsible for extensible, maintainable
systems. It directly enables the **Open/Closed Principle** (Section 4):
"open for extension, closed for modification."

**Implementation details in Python:**
Python supports polymorphism naturally via **duck typing**: "if it walks
like a duck and quacks like a duck, treat it like a duck." You don't need a
shared parent class for polymorphism to work — you just need the method to
exist.

```python
class Dog:
    def make_sound(self):
        return "Woof!"

class Cat:
    def make_sound(self):
        return "Meow!"

class Duck:
    def make_sound(self):
        return "Quack!"

animals = [Dog(), Cat(), Duck()]
for a in animals:
    print(a.make_sound())     # works for all, no shared base class required
```

There is also **operator overloading**, a form of polymorphism where you
redefine what an operator does for your class, using **dunder (double
underscore) methods**:

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):        # overloads the '+' operator
        return Vector(self.x + other.x, self.y + other.y)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

print(Vector(1, 2) + Vector(3, 4))    # Vector(4, 6)
```

**Real-world example:** A `NotificationService.send(notifier)` function that
accepts any object with a `.send_message()` method — `EmailNotifier`,
`SMSNotifier`, `SlackNotifier` — and calls it the same way regardless of
type. Adding a new `PushNotifier` requires zero changes to
`NotificationService`.

---

## 3. Supporting Concepts You Must Know

### 3.1 Class variables vs Instance variables

- **Instance variable:** belongs to a specific object (`self.x`). Each object
  has its own copy.
- **Class variable:** shared across *all* instances of a class. Defined
  directly in the class body.

```python
class Employee:
    company = "Hevo"          # class variable — shared by all employees

    def __init__(self, name):
        self.name = name       # instance variable — unique per employee

e1 = Employee("A")
e2 = Employee("B")
print(e1.company, e2.company)   # Hevo Hevo
Employee.company = "Hevo Data"  # changes for ALL instances
print(e1.company, e2.company)   # Hevo Data Hevo Data
```

**Why it matters:** A common interview bug is mutating a **mutable class
variable** (like a list) thinking it's per-instance — this causes shared
state bugs across objects.

### 3.2 Instance methods, Class methods, Static methods

```python
class Pipeline:
    total_pipelines = 0

    def __init__(self, name):
        self.name = name
        Pipeline.total_pipelines += 1

    def run(self):                       # instance method: needs an object
        return f"Running {self.name}"

    @classmethod
    def pipeline_count(cls):             # operates on the class, not object
        return cls.total_pipelines

    @staticmethod
    def is_valid_name(name: str) -> bool:  # utility, doesn't need self/cls
        return len(name) > 0
```

- **Instance method** — first arg is `self`; needs an object to call.
- **Class method** (`@classmethod`) — first arg is `cls`; operates on class
  state; often used for alternate constructors
  (e.g., `Pipeline.from_config(config)`).
- **Static method** (`@staticmethod`) — no `self` or `cls`; it's just a
  utility function placed inside the class for organizational purposes.

### 3.3 Constructors and `__init__` vs `__new__`

`__init__` initializes an already-created object. `__new__` actually creates
the object (rarely overridden except for singleton patterns or immutable
types). In 95% of interviews, you only need `__init__`.

### 3.4 Dunder (magic) methods

These let your objects integrate naturally with Python's built-in behavior.
Common ones asked about in interviews:

| Method | Purpose |
|---|---|
| `__init__` | constructor |
| `__str__` | human-readable string (`print(obj)`) |
| `__repr__` | developer-facing string (debugging) |
| `__eq__` | defines `==` behavior |
| `__lt__` | defines `<` behavior (enables sorting) |
| `__len__` | defines `len(obj)` |
| `__iter__` / `__next__` | makes the object iterable (`for x in obj`) |
| `__call__` | makes an object callable like a function |

```python
class Task:
    def __init__(self, name, priority):
        self.name, self.priority = name, priority

    def __lt__(self, other):
        return self.priority < other.priority

    def __repr__(self):
        return f"Task({self.name}, p={self.priority})"

tasks = [Task("B", 2), Task("A", 1)]
print(sorted(tasks))     # [Task(A, p=1), Task(B, p=2)] -- uses __lt__
```

### 3.5 Composition ("has-a") vs Inheritance ("is-a")

**What it is:** Composition means building a class out of *references to
other objects*, rather than inheriting from them.

**Why it matters (very common interview/design discussion):**
Inheritance creates **tight coupling** — a child is locked into the parent's
structure. Composition is more **flexible**: you can swap components at
runtime, and you avoid fragile hierarchies.

```python
# Inheritance approach (rigid)
class Engine:
    def start(self):
        return "Engine starting"

class Car(Engine):          # Car "is-a" Engine?? Doesn't make sense!
    pass

# Composition approach (correct)
class Engine:
    def start(self):
        return "Engine starting"

class Car:
    def __init__(self, engine: Engine):
        self.engine = engine     # Car "has-a" Engine

    def start(self):
        return self.engine.start()

car = Car(Engine())
print(car.start())
```

**Rule of thumb:** *"Favor composition over inheritance."* Use inheritance
only for genuine is-a relationships with shared behavior; use composition
when you're combining independent capabilities.

---

## 4. SOLID Principles (Advanced — very commonly asked in LLD rounds)

SOLID is a set of five design guidelines that make OOP systems easier to
maintain and extend.

### S — Single Responsibility Principle (SRP)
**Definition:** A class should have only **one reason to change** — i.e., it
should do exactly one job.

**Why:** If a class does multiple unrelated things, a change to one
responsibility risks breaking the others, and the class becomes hard to
understand/test.

```python
# BAD: one class doing two jobs (data storage AND persistence)
class Report:
    def __init__(self, data):
        self.data = data

    def generate_summary(self):
        ...

    def save_to_database(self):     # unrelated responsibility!
        ...

# GOOD: split responsibilities
class Report:
    def __init__(self, data):
        self.data = data

    def generate_summary(self):
        ...

class ReportRepository:
    def save(self, report: Report):
        ...
```

### O — Open/Closed Principle (OCP)
**Definition:** Classes should be **open for extension, but closed for
modification** — you should be able to add new behavior without editing
existing, tested code.

**Why:** Editing existing code risks breaking things that already work.
Adding new code (a new subclass) is safer.

```python
# BAD: must edit this function every time a new discount type is added
def calculate_discount(customer_type, amount):
    if customer_type == "regular":
        return amount * 0.9
    elif customer_type == "premium":
        return amount * 0.8
    # every new type = another elif here (violates OCP)

# GOOD: extend via polymorphism, don't modify existing classes
class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, amount): ...

class RegularDiscount(DiscountStrategy):
    def apply(self, amount): return amount * 0.9

class PremiumDiscount(DiscountStrategy):
    def apply(self, amount): return amount * 0.8

# Adding VIPDiscount later requires ZERO changes to existing classes
class VIPDiscount(DiscountStrategy):
    def apply(self, amount): return amount * 0.7
```

### L — Liskov Substitution Principle (LSP)
**Definition:** Objects of a subclass should be usable **wherever the parent
class is expected**, without breaking correctness.

**Why:** If a subclass changes expected behavior, code relying on the parent
type can break in surprising ways.

**Classic counter-example:** `Square extends Rectangle`. If `Rectangle` has
`set_width()` and `set_height()` independently, but `Square` forces
width == height, then code that assumes "changing width doesn't affect
height" breaks when given a `Square`. This is an LSP violation.

### I — Interface Segregation Principle (ISP)
**Definition:** Clients should not be forced to depend on methods they don't
use. Prefer several small, specific interfaces over one large, general one.

```python
# BAD: one fat interface forces irrelevant methods on implementers
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def eat(self): ...     # a RobotWorker doesn't eat!

# GOOD: split into focused interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): ...

class Eatable(ABC):
    @abstractmethod
    def eat(self): ...

class HumanWorker(Workable, Eatable):
    def work(self): ...
    def eat(self): ...

class RobotWorker(Workable):     # only implements what it needs
    def work(self): ...
```

### D — Dependency Inversion Principle (DIP)
**Definition:** High-level modules should not depend on low-level modules
directly — both should depend on **abstractions** (interfaces).

**Why:** If a high-level class (business logic) directly creates/depends on a
specific low-level class (e.g., a specific database), swapping that
implementation later means editing the high-level class too. Depend on an
abstraction instead, and inject the concrete implementation from outside.

```python
# BAD: NotificationService is tightly coupled to EmailSender
class EmailSender:
    def send(self, msg): print(f"Email: {msg}")

class NotificationService:
    def __init__(self):
        self.sender = EmailSender()     # hardcoded dependency

    def notify(self, msg):
        self.sender.send(msg)

# GOOD: depend on an abstraction, inject the implementation ("dependency injection")
class MessageSender(ABC):
    @abstractmethod
    def send(self, msg): ...

class EmailSender(MessageSender):
    def send(self, msg): print(f"Email: {msg}")

class SMSSender(MessageSender):
    def send(self, msg): print(f"SMS: {msg}")

class NotificationService:
    def __init__(self, sender: MessageSender):   # injected, not hardcoded
        self.sender = sender

    def notify(self, msg):
        self.sender.send(msg)

# Swapping channels requires zero change to NotificationService
NotificationService(EmailSender()).notify("Hello")
NotificationService(SMSSender()).notify("Hello")
```

**Real-world tie-in:** This exact pattern (`NotificationService` +
injectable `sender`) is a favorite in machine-coding interviews because it
tests SRP, OCP, and DIP all at once.

---

## 5. Common Interview Questions to Be Ready For

- Explain the 4 pillars with real examples (not textbook animals — use a
  system like an e-commerce app or a payment gateway).
- Difference between abstraction and encapsulation (abstraction hides
  complexity/"what vs how"; encapsulation restricts access/protects state).
- Why does Python not have true private variables? (name mangling, "we're
  all consenting adults" philosophy).
- Composition vs inheritance — when would you choose one over the other?
- What breaks if a subclass violates LSP? Give an example.
- What is duck typing, and how is it different from interfaces in Java?
- Explain SOLID with one violation + fix example for each principle.
- What's the difference between `@staticmethod` and `@classmethod`?
- How does Python resolve method calls in multiple inheritance (MRO)?

---

## 6. Quick Revision Table

| Concept | One-line definition | Key Python tool |
|---|---|---|
| Encapsulation | Restrict access to internal state | `_var`, `__var`, `@property` |
| Abstraction | Hide "how", expose "what" | `abc.ABC`, `@abstractmethod` |
| Inheritance | Reuse/extend behavior via "is-a" | `class Child(Parent):`, `super()` |
| Polymorphism | Same interface, many implementations | duck typing, dunder methods |
| Composition | Build behavior via "has-a" | store objects as attributes |
| SRP | One class, one reason to change | — |
| OCP | Extend without modifying | inheritance/polymorphism |
| LSP | Subtypes must be substitutable | — |
| ISP | Small, focused interfaces | multiple small ABCs |
| DIP | Depend on abstractions, inject concretes | constructor injection |
-e 

---


# Low-Level Design (LLD) & Machine Coding — Complete Interview Notes (Python)

> Prerequisite: read `01_OOP.md` first — LLD is OOP applied to real,
> multi-class systems. This file assumes you already know the 4 pillars and
> SOLID.

---

## 1. What is LLD / Machine Coding, and why is it a separate interview round?

**Definition:** Low-Level Design is the process of designing the **classes,
their attributes, their methods, and the relationships between them** for a
specific software system — as opposed to High-Level Design (HLD), which
deals with servers, databases, load balancers, and how big pieces of a
distributed system talk to each other.

"Machine coding round" = you're given an LLD problem (e.g., "design a parking
lot") and asked to **actually write working, compilable/runnable code** in
45–90 minutes — not just draw boxes on a whiteboard.

**Why companies test this:**
- DSA rounds test raw problem-solving on small, isolated problems.
- LLD rounds test whether you can design something **larger** that's clean,
  extensible, and doesn't collapse the moment requirements change — which is
  what real day-to-day engineering work actually looks like.
- It reveals whether you *actually* understand OOP/SOLID or just memorized
  the definitions.

**Effect of doing it well vs poorly:**
- Good LLD: adding a new feature = adding a new class, minimal changes
  elsewhere.
- Poor LLD: adding a new feature = editing 6 different `if/elif` chains
  scattered across the codebase, high risk of breaking existing behavior.

---

## 2. The General Framework for Any Machine Coding Round

Interviewers are evaluating your **process**, not just your final code.
Follow this sequence every time:

### Step 1 — Clarify Requirements (5–10 min)
Do NOT start coding immediately. Ask questions to pin down scope:
- What are the core use cases? (e.g., for a Parking Lot: park a vehicle,
  unpark, find nearest spot, calculate fee)
- What are the entities involved? (Vehicle, ParkingSpot, Ticket, Gate...)
- What's explicitly *out of scope*? (e.g., "no need to handle payments yet")
- Any constraints? (e.g., multiple vehicle types, multiple floors)

This step alone often separates strong candidates from weak ones — jumping
straight to code with wrong assumptions wastes the whole round.

### Step 2 — Identify Core Entities/Classes (Nouns → Classes)
Read the problem statement and underline the **nouns** — these usually become
classes. E.g., in "Design a parking lot with multiple floors, spots of
different sizes, and vehicles of different types":
- Nouns → `ParkingLot`, `Floor`, `ParkingSpot`, `Vehicle`, `Ticket`

### Step 3 — Identify Relationships
- **"is-a"** → inheritance (`Car is-a Vehicle`)
- **"has-a"** → composition/aggregation (`ParkingLot has-a list of Floors`)
- Identify which entities are **abstract** (need multiple implementations,
  e.g. different vehicle types) vs **concrete**.

### Step 4 — Identify Behaviors (Verbs → Methods)
Underline the **verbs**: park, unpark, find spot, calculate fee → these
become methods on the relevant classes.

### Step 5 — Apply Design Patterns Where They Fit Naturally
Don't force patterns. Recognize when a problem naturally needs one (see
Section 4). Common fits:
- Multiple algorithms/strategies for the same action → **Strategy**
- Need exactly one instance of something → **Singleton**
- Complex object creation logic → **Factory** or **Builder**
- Need to notify multiple parts of the system on an event → **Observer**

### Step 6 — Code It
- Start with the abstract/interface classes.
- Build concrete classes.
- Wire them together in a "manager"/"service" class
  (e.g., `ParkingLot` orchestrates `Floor`s and `ParkingSpot`s).
- Keep methods short; keep responsibilities single (SRP).

### Step 7 — Test With a Driver / main()
Write a small `if __name__ == "__main__":` block that exercises the main use
cases. This shows the interviewer your code actually runs and proves your
design works end-to-end.

### Step 8 — Discuss Extensibility
Be ready to answer: "How would you add X?" (e.g., "How would you add
electric vehicle charging spots?"). A good design answers this with "I'd add
one new class/enum value, nothing else changes."

---

## 3. UML Basics You Should Recognize (even if not drawing them)

You don't need to draw formal UML in most Python machine-coding rounds, but
you must understand the **relationship types**, since interviewers ask about
them verbally:

| Relationship | Meaning | "has-a" strength | Example |
|---|---|---|---|
| **Association** | Two classes are related, no ownership | weakest | `Teacher` teaches `Student` |
| **Aggregation** | "has-a" but child can exist independently | medium | `Department` has `Professor`s (professor exists even if department is deleted) |
| **Composition** | "has-a" but child's lifecycle depends on the parent | strongest | `House` has `Room`s (rooms don't exist without the house) |
| **Inheritance** | "is-a" | — | `Car` is a `Vehicle` |
| **Realization/Implementation** | class implements an interface/abstract class | — | `CreditCardPayment` implements `PaymentMethod` |

In Python, aggregation and composition both look like "store a reference as
an attribute" — the difference is conceptual (who controls the object's
lifecycle), and it's worth stating out loud in the interview to show you
understand the distinction.

---

## 4. Design Patterns — Definition, Why, Effect, Implementation, Real Example

Design patterns are **proven, reusable solutions to common design problems**.
They are not code you copy-paste — they're a *shape* of solution. Interviewers
care that you know **when** to use each one, not just the syntax.

### 4.1 Singleton Pattern

**What it is:** Ensures a class has **exactly one instance** across the
entire application, and provides a single global point of access to it.

**Why used:** Some things genuinely should exist only once — a
configuration manager, a logging service, a connection pool, a
`ParkingLot` (there's only one physical lot).

**Effect:** Saves resources (don't create the expensive object twice),
guarantees consistency (everyone reads/writes the same state). **Downside:**
overuse makes testing harder (global state) and can hide dependencies.

**Implementation in Python:**
```python
class ConfigManager:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._settings = {}
        return cls._instance

    def set(self, key, value):
        self._settings[key] = value

    def get(self, key):
        return self._settings.get(key)

a = ConfigManager()
b = ConfigManager()
a.set("env", "production")
print(b.get("env"))     # "production" -- a and b are the SAME object
print(a is b)            # True
```
(In real Python projects, a simpler and very common approach is just a
module-level variable, since Python modules are already singletons — but
interviewers usually want to see the `__new__` pattern to test your OOP
understanding.)

**Real-world example:** A `Logger` class in a data pipeline system — you
want every part of the app writing to the *same* log instance/file handle,
not creating a new file handle per class.

---

### 4.2 Factory Pattern

**What it is:** A pattern where object **creation logic is centralized** in
a separate method/class, instead of calling `SomeClass()` directly all over
your code. The client asks the factory for an object by "type," and doesn't
need to know which concrete class gets instantiated.

**Why used:** When object creation is complex, or when you want the calling
code to depend only on an abstraction — not on concrete classes directly
(this is Dependency Inversion in action).

**Effect:** Adding a new type = adding a new branch in the factory (or a
new registered class), without touching client code that *uses* the objects.

**Implementation in Python:**
```python
class Notification(ABC):
    @abstractmethod
    def notify(self, msg): ...

class EmailNotification(Notification):
    def notify(self, msg): print(f"Email: {msg}")

class SMSNotification(Notification):
    def notify(self, msg): print(f"SMS: {msg}")

class NotificationFactory:
    @staticmethod
    def create(channel: str) -> Notification:
        if channel == "email":
            return EmailNotification()
        elif channel == "sms":
            return SMSNotification()
        raise ValueError(f"Unknown channel: {channel}")

notifier = NotificationFactory.create("email")
notifier.notify("Your order shipped!")
```

**Real-world example:** A data pipeline tool creating the right `Connector`
object based on a source name string coming from a config file
(`"mysql"` → `MySQLConnector()`, `"salesforce"` → `SalesforceConnector()`).
This is directly relevant to how Hevo's actual product likely works
internally.

---

### 4.3 Builder Pattern

**What it is:** Separates the **construction of a complex object** from its
representation, letting you build the object step-by-step, especially useful
when an object has many optional parameters.

**Why used:** Avoids constructors with 8+ parameters (a "telescoping
constructor"), where it's easy to mix up argument order or forget which are
optional.

**Effect:** Readable, flexible object construction; you can validate before
final `build()`.

**Implementation in Python:**
```python
class Pizza:
    def __init__(self):
        self.size = None
        self.cheese = False
        self.toppings = []

    def __repr__(self):
        return f"Pizza(size={self.size}, cheese={self.cheese}, toppings={self.toppings})"

class PizzaBuilder:
    def __init__(self):
        self.pizza = Pizza()

    def set_size(self, size):
        self.pizza.size = size
        return self          # returning self enables method chaining

    def add_cheese(self):
        self.pizza.cheese = True
        return self

    def add_topping(self, topping):
        self.pizza.toppings.append(topping)
        return self

    def build(self):
        return self.pizza

pizza = (PizzaBuilder()
         .set_size("Large")
         .add_cheese()
         .add_topping("Mushroom")
         .add_topping("Olives")
         .build())
print(pizza)
```

**Real-world example:** Building a complex `HttpRequest` object (method,
headers, body, query params, timeout) step by step, or building a `Pipeline`
configuration object with many optional settings in a data tool.

---

### 4.4 Observer Pattern

**What it is:** Defines a **one-to-many dependency** between objects: when
one object (the "subject") changes state, all its dependents ("observers")
are notified and updated automatically.

**Why used:** Decouples the object that changes from the objects that need
to react — the subject doesn't need to know *what* the observers do, only
*that* they should be told something happened.

**Effect:** Enables event-driven designs. Adding a new observer = registering
it, zero changes to the subject's logic (Open/Closed Principle in action).

**Implementation in Python:**
```python
class Observer(ABC):
    @abstractmethod
    def update(self, event: str): ...

class Subject:
    def __init__(self):
        self._observers = []

    def subscribe(self, observer: Observer):
        self._observers.append(observer)

    def unsubscribe(self, observer: Observer):
        self._observers.remove(observer)

    def notify_all(self, event: str):
        for obs in self._observers:
            obs.update(event)

class EmailAlert(Observer):
    def update(self, event):
        print(f"[Email] Alert: {event}")

class SlackAlert(Observer):
    def update(self, event):
        print(f"[Slack] Alert: {event}")

pipeline_monitor = Subject()
pipeline_monitor.subscribe(EmailAlert())
pipeline_monitor.subscribe(SlackAlert())
pipeline_monitor.notify_all("Pipeline sync failed!")
# Both EmailAlert and SlackAlert react automatically
```

**Real-world example:** A `Pipeline` object that notifies subscribed
`AlertChannel`s (email, Slack, PagerDuty) whenever a sync fails — exactly the
kind of feature a data-pipeline product like Hevo would have.

---

### 4.5 Strategy Pattern

**What it is:** Defines a **family of interchangeable algorithms**, puts
each one behind a common interface, and lets the algorithm be selected/
swapped at runtime.

**Why used:** Avoids giant `if/elif` chains for choosing behavior; makes it
trivial to add new algorithms without touching existing code (again, OCP).

**Effect:** The "context" class becomes agnostic to *which* algorithm it's
using.

**Implementation in Python:**
```python
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: list) -> list: ...

class AscendingSort(SortStrategy):
    def sort(self, data): return sorted(data)

class DescendingSort(SortStrategy):
    def sort(self, data): return sorted(data, reverse=True)

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self.strategy = strategy

    def set_strategy(self, strategy: SortStrategy):    # swap at runtime
        self.strategy = strategy

    def sort(self, data):
        return self.strategy.sort(data)

sorter = Sorter(AscendingSort())
print(sorter.sort([5, 1, 3]))     # [1, 3, 5]
sorter.set_strategy(DescendingSort())
print(sorter.sort([5, 1, 3]))     # [5, 3, 1]
```

**Real-world example:** A fee-calculation module in a parking lot system,
where the fee strategy differs by vehicle type (flat rate for bikes,
hourly-tiered for cars) — swap the `FeeStrategy` per vehicle type without
touching the `ParkingLot` class.

---

### 4.6 Decorator Pattern

**What it is:** Lets you **add new behavior to an object dynamically**,
without modifying its class or affecting other objects of the same class —
by "wrapping" the object.

**Why used:** Avoids subclass explosion when you need many combinations of
optional features (e.g., a coffee with milk + sugar + whipped cream — you
don't want a subclass for every combination).

**Implementation in Python** (Python's `@decorator` syntax on functions is a
special case of this pattern; here's the OOP class-based version, which is
what's usually asked in interviews):
```python
class Coffee(ABC):
    @abstractmethod
    def cost(self): ...
    @abstractmethod
    def description(self): ...

class SimpleCoffee(Coffee):
    def cost(self): return 50
    def description(self): return "Coffee"

class MilkDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee
    def cost(self): return self._coffee.cost() + 10
    def description(self): return self._coffee.description() + " + Milk"

class SugarDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee
    def cost(self): return self._coffee.cost() + 5
    def description(self): return self._coffee.description() + " + Sugar"

order = SugarDecorator(MilkDecorator(SimpleCoffee()))
print(order.description(), "=", order.cost())    # Coffee + Milk + Sugar = 65
```

**Real-world example:** Wrapping a base `DataStream` with `CompressedStream`
and `EncryptedStream` decorators, combining features without a combinatorial
explosion of subclasses.

---

### 4.7 Adapter Pattern

**What it is:** Converts the interface of one class into another interface
that the client expects, letting incompatible interfaces work together.

**Why used:** You have an existing class (often third-party/legacy) whose
interface doesn't match what your system expects, and you can't/shouldn't
modify it directly.

**Implementation in Python:**
```python
class OldPrinter:
    def print_old(self, text):
        print(f"[OLD PRINTER]: {text}")

class NewPrinterInterface(ABC):
    @abstractmethod
    def print(self, text): ...

class PrinterAdapter(NewPrinterInterface):
    def __init__(self, old_printer: OldPrinter):
        self._old_printer = old_printer

    def print(self, text):
        self._old_printer.print_old(text)      # adapts the call

printer: NewPrinterInterface = PrinterAdapter(OldPrinter())
printer.print("Hello")     # works with the new interface, uses old code internally
```

**Real-world example:** Adapting a legacy third-party API's response format
into your internal `Connector` interface so the rest of your pipeline code
doesn't need to know the third party's quirks.

---

## 5. Classic Machine Coding Problems — Worked Approach

For each, practice going through Steps 1–8 from Section 2. Below are the
**entity breakdowns** to get you started — implement the full code yourself
as practice.

### 5.1 Design a Parking Lot
- **Entities:** `ParkingLot`, `ParkingFloor`, `ParkingSpot` (abstract, sized:
  Small/Medium/Large), `Vehicle` (abstract: Bike/Car/Truck), `Ticket`,
  `Gate`.
- **Patterns:** Strategy (fee calculation), Singleton (`ParkingLot` itself),
  Factory (creating the right spot/vehicle type).
- **Core methods:** `park(vehicle) -> Ticket`, `unpark(ticket) -> fee`,
  `find_available_spot(vehicle_type)`.
- **Tricky bits to discuss:** what happens when the lot is full; how spots
  are matched to vehicle size; thread-safety if two vehicles arrive at once
  (mention locks conceptually even if you don't implement them).

### 5.2 Design an Elevator System
- **Entities:** `Elevator`, `ElevatorController` (manages multiple
  elevators), `Request` (internal floor request vs external hall call with
  direction), `Floor`.
- **Patterns:** Strategy (elevator-selection algorithm — nearest elevator,
  least busy), Observer (elevator notifies controller of state changes).
- **Core methods:** `request_elevator(floor, direction)`,
  `select_elevator(request)`, `move()`, `open_doors()`, `close_doors()`.
- **Tricky bits:** direction handling (an elevator going up shouldn't stop
  for a down request below it — this is the classic "SCAN algorithm"
  discussion), multiple elevators and how the controller picks the best one.

### 5.3 Design a Rate Limiter
- **Entities:** `RateLimiter` (interface), concrete strategies: `FixedWindow`,
  `SlidingWindow`, `TokenBucket`.
- **Patterns:** Strategy (different algorithms), Factory (choosing algorithm).
- **Core methods:** `allow_request(client_id) -> bool`.
- **Tricky bits:** discuss trade-offs between algorithms (Fixed Window is
  simple but has burst issues at window boundaries; Token Bucket smooths
  traffic).

### 5.4 Design a Ride-Sharing App (simplified, e.g. Uber-lite)
- **Entities:** `Rider`, `Driver`, `Ride`, `Vehicle`, `Location`,
  `MatchingService`, `PricingService`.
- **Patterns:** Strategy (pricing/matching algorithm), Observer (notify
  rider when driver is matched/arriving), State (ride status: Requested →
  Matched → InProgress → Completed).
- **Core methods:** `request_ride()`, `match_driver()`, `start_ride()`,
  `end_ride()`, `calculate_fare()`.

### 5.5 Design a Job/Task Processing System (relevant — reported in a real
Hevo interview as "Design Job Processing")
- **Entities:** `Job` (abstract, has status: Pending/Running/Completed/
  Failed), `JobQueue`, `JobProcessor`/`Worker`, `JobScheduler`.
- **Patterns:** Strategy (different job execution logic per job type),
  Observer (notify on job completion/failure), Factory (creating job objects
  from input — note the real interview clarified input could be "a File...
  but later could be anything," which is exactly testing whether you design
  around an **abstraction** like `JobInput` rather than hardcoding `File`).
- **Core methods:** `submit(job)`, `process()`, `retry(job)`,
  `get_status(job_id)`.
- **Key lesson from the real report:** when the interviewer changes a
  requirement mid-discussion ("input can be anything, not just a File"),
  that's testing whether your design already anticipated abstraction
  (program to an interface, not a concrete type) — a strong sign to always
  ask *"should I design this to support other input types later?"* during
  Step 1 (clarify requirements) even if not explicitly asked.

```python
# Minimal skeleton showing the abstraction-first approach for Job Processing
from enum import Enum

class JobStatus(Enum):
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"

class JobInput(ABC):          # abstraction instead of hardcoding "File"
    @abstractmethod
    def read(self): ...

class FileInput(JobInput):
    def __init__(self, path): self.path = path
    def read(self): return f"reading file {self.path}"

class APIInput(JobInput):     # added later with ZERO change to Job/JobProcessor
    def __init__(self, url): self.url = url
    def read(self): return f"calling API {self.url}"

class Job:
    def __init__(self, job_id: str, input_source: JobInput):
        self.job_id = job_id
        self.input_source = input_source
        self.status = JobStatus.PENDING

class JobProcessor:
    def process(self, job: Job):
        job.status = JobStatus.RUNNING
        try:
            data = job.input_source.read()      # doesn't care what kind of input
            print(f"Processing job {job.job_id}: {data}")
            job.status = JobStatus.COMPLETED
        except Exception:
            job.status = JobStatus.FAILED
```

---

## 6. Common Mistakes in Machine Coding Rounds (and how to avoid them)

1. **Jumping to code before clarifying requirements** → always spend the
   first 5–10 minutes on Step 1.
2. **Over-engineering** — applying 4 design patterns to a simple problem.
   Only use a pattern if it solves a real problem in *this* design.
3. **God classes** — one class doing everything (violates SRP). If a class
   has more than ~5-6 methods doing unrelated things, split it.
4. **Hardcoding types with `if/elif` chains** instead of using polymorphism
   — this is the #1 thing interviewers push back on (as seen in the real
   "Job Processing" report where input type flexibility was tested).
5. **Not handling edge cases out loud** — even if you don't code every edge
   case, say them: "if the lot is full, `park()` should raise/return None
   — I'll handle that with a custom exception."
6. **No `main`/driver code** — always demo that it runs.
7. **Ignoring the interviewer's follow-up direction** — if they say "now
   input can be anything," don't argue for your original design; adapt
   quickly and explain how your abstraction absorbs the change (or refactor
   live if needed).

---

## 7. Interview Question Bank (LLD)

- Design a Parking Lot / Elevator System / Vending Machine
- Design a Rate Limiter
- Design a Job/Task Scheduler or Processing System
- Design a Ride-Sharing App
- Design a Library Management System
- Design a Notification System (multi-channel: email/SMS/push)
- Design a Tic-Tac-Toe or Chess board (tests state modeling)
- Design a Logging Framework
- Design a URL Shortener (LLD-level: focus on class design, not distributed
  system aspects)
- "Which design pattern would you use here and why?" — always justify with
  the *problem* it solves, not just the name.

---

## 8. Full Worked Answers — Mock Interview Style

> How to use this section: each problem is answered the way you'd actually
> talk through it out loud in a 45–60 minute machine coding round — starting
> with clarifying questions, then walking the interviewer through entities,
> relationships, patterns, code, and extensibility. Read each once, then
> **re-derive it yourself without looking**, out loud, against a timer. That's
> the actual skill being tested.

---

### 8.1 Design a Parking Lot

**How I'd open with the interviewer:**
"Before I start coding, let me confirm scope. Are we handling multiple
floors? Multiple vehicle types with different spot sizes? Do we need to
calculate a parking fee, or just track occupancy? Should I assume single
entry/exit gate or multiple? I'll assume: multiple floors, 3 vehicle types
(Bike/Car/Truck), 3 spot sizes (Small/Medium/Large), and a simple
hourly fee — let me know if that's off."

**Entities (nouns):**
- `Vehicle` (abstract) → `Bike`, `Car`, `Truck`
- `ParkingSpot` (abstract, has a `SpotSize` and occupancy state) →
  `SmallSpot`, `MediumSpot`, `LargeSpot`
- `ParkingFloor` — holds a list of spots
- `ParkingLot` — holds a list of floors; the orchestrator; **Singleton**
  (there's only one physical lot)
- `Ticket` — issued on entry, holds vehicle, spot, entry time
- `FeeStrategy` (abstract) — **Strategy pattern**, since fee rules differ per
  vehicle type

**Relationships:**
- `ParkingLot` **has-a** list of `ParkingFloor` (composition — floors don't
  exist without the lot)
- `ParkingFloor` **has-a** list of `ParkingSpot` (composition)
- `Bike`/`Car`/`Truck` **is-a** `Vehicle` (inheritance)
- `ParkingLot` **has-a** `FeeStrategy` (composition, injected — DIP)

**Why Singleton here:** I'd say out loud: "There's physically one parking
lot, so I'll make `ParkingLot` a Singleton to guarantee the whole app shares
one source of truth for spot availability — important so two entry points
never double-book the same spot."

**Core code:**
```python
from abc import ABC, abstractmethod
from enum import Enum
from datetime import datetime
import uuid

class SpotSize(Enum):
    SMALL, MEDIUM, LARGE = "SMALL", "MEDIUM", "LARGE"

class VehicleType(Enum):
    BIKE, CAR, TRUCK = "BIKE", "CAR", "TRUCK"

class Vehicle:
    def __init__(self, plate: str, vehicle_type: VehicleType):
        self.plate = plate
        self.vehicle_type = vehicle_type

VEHICLE_TO_SPOT = {                      # matching rule, easy to extend
    VehicleType.BIKE: SpotSize.SMALL,
    VehicleType.CAR: SpotSize.MEDIUM,
    VehicleType.TRUCK: SpotSize.LARGE,
}

class ParkingSpot:
    def __init__(self, spot_id: str, size: SpotSize):
        self.spot_id = spot_id
        self.size = size
        self.vehicle: Vehicle | None = None

    def is_free(self) -> bool:
        return self.vehicle is None

    def assign(self, vehicle: Vehicle):
        self.vehicle = vehicle

    def vacate(self):
        self.vehicle = None

class ParkingFloor:
    def __init__(self, floor_id: str, spots: list[ParkingSpot]):
        self.floor_id = floor_id
        self.spots = spots

    def find_free_spot(self, size: SpotSize) -> ParkingSpot | None:
        for spot in self.spots:
            if spot.size == size and spot.is_free():
                return spot
        return None

class FeeStrategy(ABC):
    @abstractmethod
    def calculate(self, entry_time: datetime, exit_time: datetime) -> float: ...

class HourlyFeeStrategy(FeeStrategy):
    RATE_PER_HOUR = {SpotSize.SMALL: 10, SpotSize.MEDIUM: 20, SpotSize.LARGE: 30}

    def calculate(self, entry_time, exit_time, size: SpotSize = SpotSize.MEDIUM):
        hours = max(1, int((exit_time - entry_time).total_seconds() // 3600) + 1)
        return hours * self.RATE_PER_HOUR[size]

class Ticket:
    def __init__(self, vehicle: Vehicle, spot: ParkingSpot):
        self.ticket_id = str(uuid.uuid4())
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()

class ParkingLot:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, floors: list[ParkingFloor] = None, fee_strategy: FeeStrategy = None):
        if not hasattr(self, "initialized"):
            self.floors = floors or []
            self.fee_strategy = fee_strategy or HourlyFeeStrategy()
            self.active_tickets: dict[str, Ticket] = {}
            self.initialized = True

    def park(self, vehicle: Vehicle) -> Ticket | None:
        required_size = VEHICLE_TO_SPOT[vehicle.vehicle_type]
        for floor in self.floors:
            spot = floor.find_free_spot(required_size)
            if spot:
                spot.assign(vehicle)
                ticket = Ticket(vehicle, spot)
                self.active_tickets[ticket.ticket_id] = ticket
                return ticket
        return None                       # lot full for this vehicle type

    def unpark(self, ticket_id: str) -> float:
        ticket = self.active_tickets.pop(ticket_id)
        fee = self.fee_strategy.calculate(ticket.entry_time, datetime.now())
        ticket.spot.vacate()
        return fee
```

**How I'd close (extensibility discussion):**
"If we wanted to add electric vehicle charging spots, I'd add
`VehicleType.EV` and `SpotSize.EV_CHARGING` — zero changes to `ParkingLot`
or `ParkingFloor` logic, since matching is driven by the `VEHICLE_TO_SPOT`
map and spot-search is generic. If fee rules become time-of-day dependent,
I'd swap in a new `FeeStrategy` implementation — again, zero changes to
`ParkingLot`."

---

### 8.2 Design an Elevator System

**How I'd open:**
"Clarifying: how many elevators? Do we need to optimize which elevator
answers a hall call, or is a single elevator fine for scope? Should I
implement the SCAN-style direction logic (an elevator going up shouldn't
reverse for a lower down-request) or keep it simple with a FIFO queue? I'll
assume multiple elevators with a basic 'nearest idle elevator' selection
strategy, and I can extend to SCAN if time allows."

**Entities:**
- `Elevator` — has current floor, direction, state (IDLE/MOVING/
  DOOR_OPEN), and a request queue
- `ElevatorController` — owns all elevators, decides which elevator serves a
  hall call (**Strategy pattern** for the selection algorithm)
- `Request` — internal (button pressed inside cabin) or external (hall call
  with direction)
- `Direction` — enum: UP, DOWN, IDLE

**Relationships:**
- `ElevatorController` **has-a** list of `Elevator` (composition)
- `ElevatorController` **has-a** `ElevatorSelectionStrategy` (injected —
  DIP/Strategy)

**Core code:**
```python
from enum import Enum
from abc import ABC, abstractmethod

class Direction(Enum):
    UP, DOWN, IDLE = "UP", "DOWN", "IDLE"

class ElevatorState(Enum):
    IDLE, MOVING, DOOR_OPEN = "IDLE", "MOVING", "DOOR_OPEN"

class Elevator:
    def __init__(self, elevator_id: str):
        self.elevator_id = elevator_id
        self.current_floor = 0
        self.direction = Direction.IDLE
        self.state = ElevatorState.IDLE
        self.requests: set[int] = set()

    def add_request(self, floor: int):
        self.requests.add(floor)
        if self.state == ElevatorState.IDLE:
            self.direction = Direction.UP if floor > self.current_floor else Direction.DOWN

    def step(self):
        """Move one floor toward the nearest pending request (simplified SCAN)."""
        if not self.requests:
            self.direction = Direction.IDLE
            self.state = ElevatorState.IDLE
            return
        target = min(self.requests, key=lambda f: abs(f - self.current_floor))
        if target > self.current_floor:
            self.current_floor += 1
        elif target < self.current_floor:
            self.current_floor -= 1
        if self.current_floor == target:
            self.requests.discard(self.current_floor)
            self.state = ElevatorState.DOOR_OPEN
        else:
            self.state = ElevatorState.MOVING

class SelectionStrategy(ABC):
    @abstractmethod
    def select(self, elevators: list[Elevator], floor: int) -> Elevator: ...

class NearestIdleStrategy(SelectionStrategy):
    def select(self, elevators, floor):
        idle = [e for e in elevators if e.state == ElevatorState.IDLE] or elevators
        return min(idle, key=lambda e: abs(e.current_floor - floor))

class ElevatorController:
    def __init__(self, elevators: list[Elevator], strategy: SelectionStrategy = None):
        self.elevators = elevators
        self.strategy = strategy or NearestIdleStrategy()

    def request_elevator(self, floor: int, direction: Direction):
        chosen = self.strategy.select(self.elevators, floor)
        chosen.add_request(floor)
        return chosen.elevator_id
```

**Extensibility discussion (say this out loud):**
"Swapping `NearestIdleStrategy` for a `LeastBusyStrategy` or a true SCAN
implementation requires zero changes to `ElevatorController` — that's the
Strategy pattern paying off. If we needed to persist state or notify a
building-management dashboard on floor arrival, I'd add an Observer
so `Elevator` notifies subscribers without knowing who's listening."

---

### 8.3 Design a Rate Limiter

**How I'd open:**
"Clarifying: is this per-user/per-API-key, or global? Do we need multiple
algorithms, or just one? I'll implement it so the algorithm is swappable via
Strategy, and default to Token Bucket since it handles bursts well, but I
can show Fixed Window and Sliding Window too if useful."

**Entities:**
- `RateLimiter` (abstract interface: `allow_request(client_id) -> bool`)
- `TokenBucketLimiter`, `FixedWindowLimiter`, `SlidingWindowLimiter`
  (concrete strategies)
- `RateLimiterFactory` — creates the configured limiter (**Factory**)

**Core code (Token Bucket — the one most likely to impress):**
```python
import time
from abc import ABC, abstractmethod

class RateLimiter(ABC):
    @abstractmethod
    def allow_request(self, client_id: str) -> bool: ...

class TokenBucketLimiter(RateLimiter):
    def __init__(self, capacity: int, refill_rate_per_sec: float):
        self.capacity = capacity
        self.refill_rate = refill_rate_per_sec
        self.buckets: dict[str, tuple[float, float]] = {}   # client -> (tokens, last_refill_ts)

    def allow_request(self, client_id: str) -> bool:
        now = time.time()
        tokens, last_ts = self.buckets.get(client_id, (self.capacity, now))
        tokens = min(self.capacity, tokens + (now - last_ts) * self.refill_rate)
        if tokens >= 1:
            self.buckets[client_id] = (tokens - 1, now)
            return True
        self.buckets[client_id] = (tokens, now)
        return False

class FixedWindowLimiter(RateLimiter):
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.counters: dict[str, tuple[int, int]] = {}      # client -> (window_start, count)

    def allow_request(self, client_id: str) -> bool:
        now = int(time.time())
        window_start = now - (now % self.window_seconds)
        stored_start, count = self.counters.get(client_id, (window_start, 0))
        if stored_start != window_start:
            count = 0
            stored_start = window_start
        if count < self.max_requests:
            self.counters[client_id] = (stored_start, count + 1)
            return True
        self.counters[client_id] = (stored_start, count)
        return False
```

**What I'd say about trade-offs (this is what interviewers are really
listening for):**
"Fixed Window is simplest but allows up to 2x the limit right at window
boundaries — a client could send `max` requests at 11:59:59 and `max` again
at 12:00:00. Sliding Window fixes that by weighting the previous window, at
the cost of more bookkeeping. Token Bucket is the best general-purpose
choice because it naturally allows short bursts up to `capacity` while
enforcing a steady average rate via `refill_rate` — that matches how most
production APIs (e.g., AWS, Stripe) actually rate-limit."

---

### 8.4 Design a Ride-Sharing App (Uber-lite)

**How I'd open:**
"I'll scope this down to: rider requests a ride, we match a nearby
available driver, ride goes through a state machine (Requested → Matched →
InProgress → Completed), and we compute a fare at the end. I'll skip
payments/routing/maps and focus on the class design and state transitions —
let me know if you want me to go deeper anywhere."

**Entities:**
- `Rider`, `Driver` — both could share a common `User` base for name/id
- `Location` — simple lat/lng value object
- `Ride` — has a `RideStatus` (**State pattern** — status determines what
  actions are valid)
- `MatchingStrategy` (abstract) — nearest-driver logic (**Strategy**)
- `PricingStrategy` (abstract) — fare calculation (**Strategy**)
- `RideService` — orchestrator

**Core code:**
```python
from enum import Enum
from abc import ABC, abstractmethod
import math

class RideStatus(Enum):
    REQUESTED, MATCHED, IN_PROGRESS, COMPLETED, CANCELLED = range(5)

class Location:
    def __init__(self, lat: float, lng: float):
        self.lat, self.lng = lat, lng

    def distance_to(self, other: "Location") -> float:
        return math.hypot(self.lat - other.lat, self.lng - other.lng)

class Driver:
    def __init__(self, driver_id: str, location: Location):
        self.driver_id = driver_id
        self.location = location
        self.available = True

class Rider:
    def __init__(self, rider_id: str, location: Location):
        self.rider_id = rider_id
        self.location = location

class Ride:
    def __init__(self, rider: Rider, driver: Driver):
        self.rider = rider
        self.driver = driver
        self.status = RideStatus.MATCHED
        self.fare = None

    def start(self):
        if self.status != RideStatus.MATCHED:
            raise ValueError(f"Cannot start ride from state {self.status}")
        self.status = RideStatus.IN_PROGRESS

    def complete(self, fare: float):
        if self.status != RideStatus.IN_PROGRESS:
            raise ValueError(f"Cannot complete ride from state {self.status}")
        self.status = RideStatus.COMPLETED
        self.fare = fare
        self.driver.available = True

class MatchingStrategy(ABC):
    @abstractmethod
    def match(self, rider: Rider, drivers: list[Driver]) -> Driver | None: ...

class NearestDriverMatching(MatchingStrategy):
    def match(self, rider, drivers):
        available = [d for d in drivers if d.available]
        if not available:
            return None
        return min(available, key=lambda d: d.location.distance_to(rider.location))

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fare(self, distance_km: float) -> float: ...

class BasicPricing(PricingStrategy):
    BASE_FARE, PER_KM = 30, 12
    def calculate_fare(self, distance_km):
        return self.BASE_FARE + distance_km * self.PER_KM

class RideService:
    def __init__(self, drivers: list[Driver], matching: MatchingStrategy = None,
                 pricing: PricingStrategy = None):
        self.drivers = drivers
        self.matching = matching or NearestDriverMatching()
        self.pricing = pricing or BasicPricing()
        self.rides: dict[str, Ride] = {}

    def request_ride(self, rider: Rider) -> Ride | None:
        driver = self.matching.match(rider, self.drivers)
        if not driver:
            return None
        driver.available = False
        ride = Ride(rider, driver)
        self.rides[rider.rider_id] = ride
        return ride

    def end_ride(self, ride: Ride, distance_km: float) -> float:
        fare = self.pricing.calculate_fare(distance_km)
        ride.complete(fare)
        return fare
```

**How I'd defend the state checks:** "I guard every transition
(`start()`, `complete()`) with a status check that raises if called out of
order — this is exactly the kind of invariant-protection encapsulation
gives us, and it prevents bugs like completing a ride that was never
started."

---

### 8.5 Design a Job/Task Processing System

*(Already covered in depth in Section 5.5 — the abstraction-first `JobInput`
design. In an interview, walk through that exact reasoning: define `Job`,
`JobInput` as an abstraction from the start — even if only asked for "File"
input — since interviewers commonly widen scope mid-round to test this.)*

**Additional pieces worth adding live if time allows:**
- A `JobQueue` (simple `collections.deque` wrapped in a class) feeding a
  `JobProcessor`.
- A `RetryPolicy` (Strategy) for failed jobs — e.g., exponential backoff.
- An `Observer` so a `JobScheduler` notifies subscribers (logging, alerting)
  on status changes, exactly like Section 4.4's Observer example.

**What to say when the interviewer changes requirements mid-round:**
"Since I modeled input behind a `JobInput` abstraction and the processor
only calls `.read()`, adding an `APIInput` or `QueueInput` class is a pure
addition — I don't need to touch `Job` or `JobProcessor` at all. That's the
Open/Closed Principle in action."

---

### 8.6 Design a Library Management System

**How I'd open:**
"Scoping: members can borrow/return books, books have multiple copies, and
there's a due-date/fine system. Should I handle reservations/holds for
already-borrowed books? I'll include a simple hold queue since it's a
common follow-up ask."

**Entities:**
- `Book` — metadata (title, author, ISBN)
- `BookCopy` — a physical copy of a `Book`, has status (AVAILABLE/BORROWED/
  RESERVED)
- `Member` — has borrowed copies, holds
- `Library` — orchestrator, catalog of books/copies
- `FineStrategy` (Strategy) — overdue fine calculation
- `Loan` — record of a borrow (member, copy, due date)

**Core code:**
```python
from enum import Enum
from datetime import datetime, timedelta

class CopyStatus(Enum):
    AVAILABLE, BORROWED, RESERVED = "AVAILABLE", "BORROWED", "RESERVED"

class Book:
    def __init__(self, isbn: str, title: str, author: str):
        self.isbn, self.title, self.author = isbn, title, author

class BookCopy:
    def __init__(self, copy_id: str, book: Book):
        self.copy_id = copy_id
        self.book = book
        self.status = CopyStatus.AVAILABLE

class Loan:
    def __init__(self, member_id: str, copy: BookCopy, days_allowed: int = 14):
        self.member_id = member_id
        self.copy = copy
        self.borrow_date = datetime.now()
        self.due_date = self.borrow_date + timedelta(days=days_allowed)
        self.returned = False

class FineStrategy:
    RATE_PER_DAY = 5
    def calculate(self, loan: Loan) -> float:
        overdue_days = max(0, (datetime.now() - loan.due_date).days)
        return overdue_days * self.RATE_PER_DAY

class Library:
    def __init__(self):
        self.copies: dict[str, BookCopy] = {}
        self.active_loans: dict[str, Loan] = {}       # copy_id -> Loan
        self.hold_queue: dict[str, list[str]] = {}     # isbn -> [member_ids]
        self.fine_strategy = FineStrategy()

    def borrow(self, member_id: str, isbn: str) -> Loan | None:
        copy = next((c for c in self.copies.values()
                     if c.book.isbn == isbn and c.status == CopyStatus.AVAILABLE), None)
        if not copy:
            self.hold_queue.setdefault(isbn, []).append(member_id)   # queue a hold
            return None
        copy.status = CopyStatus.BORROWED
        loan = Loan(member_id, copy)
        self.active_loans[copy.copy_id] = loan
        return loan

    def return_book(self, copy_id: str) -> float:
        loan = self.active_loans.pop(copy_id)
        loan.returned = True
        fine = self.fine_strategy.calculate(loan)
        copy = self.copies[copy_id]
        isbn = copy.book.isbn
        if self.hold_queue.get(isbn):
            copy.status = CopyStatus.RESERVED       # next person in line gets it
        else:
            copy.status = CopyStatus.AVAILABLE
        return fine
```

**Discussion point I'd raise myself:** "Note `Book` (the metadata) and
`BookCopy` (a physical item) are separate classes — a common LLD mistake is
merging these, but a library can have 5 physical copies of the same book,
each with independent availability. Splitting them is basically applying
SRP at the data-modeling level."

---

### 8.7 Design a Notification System (multi-channel: email/SMS/push)

*(Core structure already shown in Section 4.4's Observer example and
Section 4.2's Factory example — here's how I'd combine them live.)*

**How I'd open:**
"I'll design this so adding a new channel — say WhatsApp — never touches
existing code. That means: a common `Notifier` interface (Strategy/
Factory), and a `NotificationService` that fans out to multiple channels per
user preference (Observer-ish fan-out)."

**Core code:**
```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, recipient, message):
        print(f"[Email to {recipient}]: {message}")

class SMSNotifier(Notifier):
    def send(self, recipient, message):
        print(f"[SMS to {recipient}]: {message}")

class PushNotifier(Notifier):
    def send(self, recipient, message):
        print(f"[Push to {recipient}]: {message}")

class NotifierFactory:
    _registry = {"email": EmailNotifier, "sms": SMSNotifier, "push": PushNotifier}

    @classmethod
    def create(cls, channel: str) -> Notifier:
        if channel not in cls._registry:
            raise ValueError(f"Unsupported channel: {channel}")
        return cls._registry[channel]()

    @classmethod
    def register(cls, channel: str, notifier_cls):
        """Lets new channels be added WITHOUT modifying this class body — OCP."""
        cls._registry[channel] = notifier_cls

class NotificationService:
    def __init__(self, user_preferences: dict[str, list[str]]):
        self.user_preferences = user_preferences   # user_id -> list of channels

    def notify(self, user_id: str, recipient: str, message: str):
        for channel in self.user_preferences.get(user_id, []):
            notifier = NotifierFactory.create(channel)
            notifier.send(recipient, message)

service = NotificationService({"u1": ["email", "sms"]})
service.notify("u1", "user@example.com", "Your pipeline sync completed.")
```

**Extensibility answer I'd give:** "To add WhatsApp, I register a new
`WhatsAppNotifier` via `NotifierFactory.register()` — I never touch
`NotificationService` or the factory's internals. If we later need retry-
on-failure per channel, I'd wrap each `Notifier` in a Decorator that retries
before giving up, without changing the notifiers themselves."

---

### 8.8 Design a Tic-Tac-Toe or Chess Board (state modeling)

**How I'd open (Tic-Tac-Toe, the more common ask):**
"I'll model the board, players, and a simple win-checker. For extensibility
I'll make the board size configurable (not hardcoded 3x3) since that's a
common follow-up."

**Entities:**
- `Board` — grid of cells, knows how to check for a winner
- `Player` — symbol (X/O)
- `Game` — orchestrator, turn management, win/draw detection

**Core code:**
```python
class Board:
    def __init__(self, size: int = 3):
        self.size = size
        self.grid = [[None] * size for _ in range(size)]

    def place(self, row: int, col: int, symbol: str) -> bool:
        if self.grid[row][col] is not None:
            return False
        self.grid[row][col] = symbol
        return True

    def check_winner(self) -> str | None:
        lines = []
        lines += self.grid                                        # rows
        lines += [list(col) for col in zip(*self.grid)]            # columns
        lines.append([self.grid[i][i] for i in range(self.size)])  # main diagonal
        lines.append([self.grid[i][self.size - 1 - i] for i in range(self.size)])  # anti-diagonal
        for line in lines:
            if line[0] is not None and all(cell == line[0] for cell in line):
                return line[0]
        return None

    def is_full(self) -> bool:
        return all(cell is not None for row in self.grid for cell in row)

class Player:
    def __init__(self, name: str, symbol: str):
        self.name, self.symbol = name, symbol

class Game:
    def __init__(self, players: list[Player], size: int = 3):
        self.board = Board(size)
        self.players = players
        self.turn = 0

    def play_move(self, row: int, col: int) -> str:
        current = self.players[self.turn % len(self.players)]
        if not self.board.place(row, col, current.symbol):
            return "Invalid move"
        winner = self.board.check_winner()
        if winner:
            return f"{current.name} wins!"
        if self.board.is_full():
            return "Draw"
        self.turn += 1
        return "Continue"
```

**If asked to extend toward Chess (say this, don't necessarily code it
all):** "Chess needs a `Piece` hierarchy (`King`, `Queen`, `Rook`... each
overriding a `get_valid_moves(board, position)` method — that's
polymorphism doing the heavy lifting), a `Board` that tracks piece
positions, and a `Game` that validates moves, detects check/checkmate, and
manages turns. The key design decision is putting movement rules **inside
each piece subclass** rather than a giant `if piece_type == 'King': ...` in
the board — textbook Open/Closed Principle."

---

### 8.9 Design a Logging Framework

**How I'd open:**
"I'll design for: multiple log levels, multiple output destinations
(console, file — extensible to more), and a single shared logger instance
per application. I'll use Singleton for the logger and Strategy for output
destinations (called 'appenders')."

**Core code:**
```python
from enum import IntEnum
from abc import ABC, abstractmethod
from datetime import datetime

class LogLevel(IntEnum):
    DEBUG, INFO, WARNING, ERROR = 0, 1, 2, 3

class LogAppender(ABC):
    @abstractmethod
    def append(self, formatted_message: str) -> None: ...

class ConsoleAppender(LogAppender):
    def append(self, formatted_message):
        print(formatted_message)

class FileAppender(LogAppender):
    def __init__(self, filepath: str):
        self.filepath = filepath
    def append(self, formatted_message):
        with open(self.filepath, "a") as f:
            f.write(formatted_message + "\n")

class Logger:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.appenders = []
            cls._instance.min_level = LogLevel.DEBUG
        return cls._instance

    def add_appender(self, appender: LogAppender):
        self.appenders.append(appender)

    def set_level(self, level: LogLevel):
        self.min_level = level

    def log(self, level: LogLevel, message: str):
        if level < self.min_level:
            return
        formatted = f"[{datetime.now().isoformat()}] [{level.name}] {message}"
        for appender in self.appenders:
            appender.append(formatted)

    def info(self, message):    self.log(LogLevel.INFO, message)
    def error(self, message):   self.log(LogLevel.ERROR, message)
    def debug(self, message):   self.log(LogLevel.DEBUG, message)
    def warning(self, message): self.log(LogLevel.WARNING, message)

logger = Logger()
logger.add_appender(ConsoleAppender())
logger.info("Pipeline started")
```

**Why Singleton is defensible here (say this if pushed back on):** "Unlike
some overused Singletons, a logger genuinely benefits from being global —
every part of the app should write to the same configured appenders and
respect the same minimum level, and creating multiple loggers with
different configs would cause inconsistent/missing logs."

---

### 8.10 Design a URL Shortener (LLD-level — class design, not distributed systems)

**How I'd open:**
"At the LLD level (not the HLD/distributed-systems level), I'll focus on:
the encoding strategy for generating short codes, the mapping storage
abstraction, and expiry handling — not on sharding databases or load
balancers, since that's an HLD concern."

**Entities:**
- `ShortCodeGenerator` (abstract, Strategy) — e.g., base62 counter-based vs
  random-hash-based
- `UrlRepository` (abstract) — storage abstraction (in-memory now, DB later
  — DIP)
- `UrlShortenerService` — orchestrator: create short URL, resolve short URL,
  handle expiry

**Core code:**
```python
from abc import ABC, abstractmethod
from datetime import datetime, timedelta
import string, random

class ShortCodeGenerator(ABC):
    @abstractmethod
    def generate(self, long_url: str) -> str: ...

class Base62CounterGenerator(ShortCodeGenerator):
    ALPHABET = string.digits + string.ascii_letters
    def __init__(self):
        self.counter = 0
    def generate(self, long_url: str) -> str:
        self.counter += 1
        n, code = self.counter, ""
        while n:
            n, rem = divmod(n, 62)
            code = self.ALPHABET[rem] + code
        return code.zfill(6)

class UrlRepository(ABC):
    @abstractmethod
    def save(self, code: str, long_url: str, expiry: datetime): ...
    @abstractmethod
    def get(self, code: str) -> tuple[str, datetime] | None: ...

class InMemoryUrlRepository(UrlRepository):
    def __init__(self):
        self.store: dict[str, tuple[str, datetime]] = {}
    def save(self, code, long_url, expiry):
        self.store[code] = (long_url, expiry)
    def get(self, code):
        return self.store.get(code)

class UrlShortenerService:
    def __init__(self, generator: ShortCodeGenerator = None,
                 repository: UrlRepository = None, default_ttl_days: int = 30):
        self.generator = generator or Base62CounterGenerator()
        self.repository = repository or InMemoryUrlRepository()
        self.default_ttl_days = default_ttl_days

    def shorten(self, long_url: str) -> str:
        code = self.generator.generate(long_url)
        expiry = datetime.now() + timedelta(days=self.default_ttl_days)
        self.repository.save(code, long_url, expiry)
        return code

    def resolve(self, code: str) -> str | None:
        record = self.repository.get(code)
        if not record:
            return None
        long_url, expiry = record
        if datetime.now() > expiry:
            return None                    # expired
        return long_url
```

**The trade-off discussion interviewers actually want here:** "Base62
counter-based generation guarantees no collisions and short codes, but
requires a centralized counter (a scaling concern at the HLD level, out of
scope here). A random-hash approach avoids the centralized counter but
needs a collision check against the repository before saving. I made both
swappable via the `ShortCodeGenerator` abstraction so this decision doesn't
lock in the rest of the design — and `UrlRepository` being abstract means
swapping in-memory storage for a real database later touches nothing else."

---

### 8.11 "Which design pattern would you use here and why?" — How to Answer On the Spot

This is a meta-question interviewers drop into *any* of the problems above.
Answer with this 3-part structure every time, out loud:

1. **Name the problem, not the pattern first:** "We have N interchangeable
   ways to do X, and I don't want an `if/elif` chain that grows every time
   we add a new way" → *then* say "...so I'd use Strategy."
2. **State what breaks without it:** "Without it, every new fee type/vehicle
   type/notification channel means editing this existing, tested method —
   violates Open/Closed and risks regressions."
3. **Name the trade-off:** "The cost is one extra class per variant and a
   bit more indirection — worth it once you expect more than ~2-3 variants
   or expect them to grow over time. For something that will only ever have
   one implementation, I wouldn't add the abstraction."

Interviewers consistently rate candidates higher when they justify a pattern
by the **problem it solves**, not by naming it first and reverse-engineering
a justification.


## 9. Quick Revision Table

| Pattern | One-line definition | Use when |
|---|---|---|
| Singleton | Exactly one instance, global access | Config, Logger, shared resource |
| Factory | Centralized object creation | Creating objects by "type" |
| Builder | Step-by-step construction of complex objects | Many optional params |
| Observer | Notify many dependents on state change | Event-driven, pub/sub |
| Strategy | Swappable algorithms behind one interface | Multiple ways to do one action |
| Decorator | Add behavior dynamically via wrapping | Avoid subclass explosion |
| Adapter | Make incompatible interfaces work together | Wrapping legacy/third-party code |

**The 8-step machine coding framework, memorized:**
1. Clarify requirements
2. Identify entities (nouns → classes)
3. Identify relationships (is-a / has-a)
4. Identify behaviors (verbs → methods)
5. Apply patterns where they naturally fit
6. Code (abstractions first, then concrete classes, then orchestrator)
7. Write a driver/`main()` to prove it runs
8. Discuss extensibility ("how would you add X?")

---

# System Design Basics — Complete Interview Notes

> Scope note: for an SDE I role, interviewers rarely expect deep HLD
> (distributed systems at scale). What they *do* expect is that you
> understand the vocabulary and reasoning behind common building blocks well
> enough to (a) design a REST API cleanly, (b) reason about basic
> scalability/caching trade-offs, and (c) speak intelligently about
> data-pipeline concepts, since that's Hevo's core product domain. This
> section is written at that level — solid fundamentals, not distributed
> systems mastery.

---

## 1. What is System Design, and how is it different from LLD?

**Definition:** System Design is the process of defining the **architecture,
components, and how they communicate** to satisfy functional and
non-functional requirements (scale, reliability, latency) of a software
system.

**LLD vs System Design (HLD), in one line each:**
- **LLD** = "What classes do I need, and how do they relate?" (inside one
  service/application)
- **System Design (HLD)** = "What *services/servers/databases* do I need,
  and how do they talk to each other?" (across the whole system)

**Why it's tested:** Even at SDE I level, you'll be working inside a larger
system (e.g., Hevo's data pipeline platform has ingestion services, a
scheduler, connectors, a control plane, storage). Understanding how pieces
fit together — even if you're not designing the whole thing — makes you a
far more effective contributor and lets you reason about where your code
runs and what it depends on.

---

## 2. Client-Server Architecture — the starting point

**What it is:** The fundamental model where a **client** (browser, mobile
app, another service) sends a **request** to a **server**, which processes
it and sends back a **response**.

**Why it matters as a baseline:** Almost every system design question builds
on this. Before adding load balancers, caches, and queues, always be able to
draw the simplest version: `Client → Server → Database`. Interviewers want
to see you start simple and add complexity only to solve a stated problem —
not throw every buzzword at a small problem.

**Effect of skipping this step:** Candidates who jump straight to
"we'll use Kafka and shard the database" without justifying *why* come
across as pattern-matching buzzwords rather than reasoning from
requirements. Always earn each component.

---

## 3. REST API Design Principles

**What REST is:** REST (**RE**presentational **S**tate **T**ransfer) is an
architectural style for designing networked APIs around **resources**
(nouns, like `/users`, `/pipelines`) manipulated via standard HTTP methods
(verbs).

**Why it's used:** Before REST conventions became standard, APIs were
inconsistent — every team invented its own verbs and structure
(`/getUser?id=5`, `/deleteUserById`, ...). REST gives a predictable,
uniform way to design APIs so any developer can guess the shape of an
endpoint they've never seen before.

### 3.1 Core principles

| Principle | What it means |
|---|---|
| **Resources are nouns** | `/users`, `/pipelines`, `/pipelines/42/runs` — not `/getUser` |
| **HTTP methods are verbs** | `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove) |
| **Statelessness** | Each request contains everything needed to process it; the server stores no client session state between requests |
| **Uniform interface** | Same conventions apply everywhere — predictability over cleverness |
| **Resource representations** | Usually JSON; the same resource can be represented differently (JSON, XML) without changing the underlying concept |

### 3.2 HTTP Methods — effect and correct usage

```
GET    /pipelines           -> list all pipelines
GET    /pipelines/42        -> get pipeline with id 42
POST   /pipelines           -> create a new pipeline (body = pipeline data)
PUT    /pipelines/42        -> replace pipeline 42 entirely
PATCH  /pipelines/42        -> partially update pipeline 42
DELETE /pipelines/42        -> delete pipeline 42
```

**Idempotency (a favorite interview follow-up):**
An operation is **idempotent** if calling it multiple times has the same
effect as calling it once.
- `GET`, `PUT`, `DELETE` → idempotent (calling `DELETE /pipelines/42` five
  times still ends with pipeline 42 gone — no different from calling it
  once).
- `POST` → **not** idempotent by default (calling `POST /pipelines` five
  times creates five pipelines).

**Why this matters in practice:** If a client's request times out, it's
generally *safe* to retry an idempotent request automatically, but *not*
safe to blindly retry a `POST` (you'd risk duplicate resource creation) —
this is exactly why many real APIs (Stripe, Hevo-like ingestion APIs)
support an **idempotency key** header on `POST` requests, so retries with
the same key don't create duplicates.

### 3.3 Status Codes — know these cold

| Code | Meaning | When to use |
|---|---|---|
| `200 OK` | Success | Successful GET/PUT/PATCH |
| `201 Created` | Resource created | Successful POST |
| `204 No Content` | Success, no body | Successful DELETE |
| `400 Bad Request` | Client sent invalid data | Validation failure |
| `401 Unauthorized` | Missing/invalid auth | Not logged in / bad token |
| `403 Forbidden` | Authenticated but not allowed | Permission denied |
| `404 Not Found` | Resource doesn't exist | Bad ID |
| `409 Conflict` | Request conflicts with current state | Duplicate resource, version conflict |
| `429 Too Many Requests` | Rate limited | Client exceeded rate limit |
| `500 Internal Server Error` | Server-side bug | Unhandled exception |
| `503 Service Unavailable` | Server temporarily can't handle requests | Overload, maintenance |

### 3.4 Designing a good REST API — a worked example

**Problem:** design endpoints for a data-pipeline platform managing
pipelines and their connectors (directly relevant to Hevo's domain).

```
POST   /pipelines                        -> create a pipeline
GET    /pipelines                        -> list pipelines (supports ?status=active)
GET    /pipelines/{id}                   -> get one pipeline
PATCH  /pipelines/{id}                   -> update pipeline config
DELETE /pipelines/{id}                   -> delete a pipeline

GET    /pipelines/{id}/runs              -> list execution history (nested resource)
POST   /pipelines/{id}/runs              -> trigger a manual run
GET    /pipelines/{id}/runs/{run_id}     -> get status of a specific run

GET    /connectors                       -> list available connector types (MySQL, Salesforce...)
```

**Design choices worth explaining out loud in an interview:**
- **Nesting** (`/pipelines/{id}/runs`) models the "a run belongs to a
  pipeline" relationship directly in the URL — this is a very common LLD-
  adjacent design decision interviewers probe.
- **Filtering/pagination via query params** (`?status=active&page=2&limit=50`)
  keeps the resource path clean while supporting real-world list sizes —
  always mention pagination when discussing any `list` endpoint; returning
  unbounded lists is a common junior-candidate mistake.
- **Versioning:** mention that APIs are usually versioned
  (`/v1/pipelines`) so breaking changes don't affect existing clients.

---

## 4. Scalability Basics

**What it is:** The ability of a system to handle **increased load**
(more users, more data, more requests) without degrading performance.

### 4.1 Vertical vs Horizontal Scaling

| | Vertical Scaling ("scale up") | Horizontal Scaling ("scale out") |
|---|---|---|
| **What it is** | Add more power (CPU/RAM) to one machine | Add more machines |
| **Effect** | Simple, no architecture change needed | Requires load balancing, often needs stateless services |
| **Limit** | Hits a hardware ceiling eventually | Scales much further, but adds complexity |
| **Example** | Upgrading a database server's RAM | Adding more web servers behind a load balancer |

**Why this matters:** A very common interview question is "how would you
scale this?" — always mention **both** options and explain the trade-off:
vertical scaling is the simplest first step (and often good enough), but
horizontal scaling is what you reach for once a single machine can't handle
the load, at the cost of added complexity (needing to coordinate multiple
machines, handle partial failures, etc.).

### 4.2 Statelessness — why it enables horizontal scaling

**What it is:** A **stateless service** doesn't store client-specific data
between requests — every request is self-contained (e.g., carries its own
auth token).

**Why it's used:** If a service is stateless, **any** server instance can
handle **any** request — this is what makes horizontal scaling behind a
load balancer straightforward. If a service is **stateful** (e.g., stores
session data in server memory), you need "sticky sessions" (routing a
client to the same server every time) or externalize that state (e.g., to
Redis) — both add complexity.

---

## 5. Load Balancing

**What it is:** A component that sits in front of multiple server instances
and **distributes incoming requests** across them.

**Why it's used:**
- Prevents any single server from being overwhelmed.
- Enables horizontal scaling (see above).
- Provides fault tolerance: if one server goes down, the load balancer
  stops routing traffic to it (via health checks) and the system keeps
  running.

**Common algorithms (know these at a conceptual level):**
- **Round Robin** — requests go to servers in rotating order.
- **Least Connections** — send the next request to whichever server
  currently has the fewest active connections.
- **IP Hash** — route based on a hash of the client's IP, useful for
  sticky sessions.

**Effect:** Without a load balancer, scaling horizontally is pointless —
you'd have multiple servers but no way to distribute traffic across them,
so one server would still take all the load.

---

## 6. Caching

**What it is:** Storing a copy of frequently-accessed or expensive-to-
compute data in a **fast-access layer** (often in-memory) so future
requests for the same data don't have to redo the expensive work (e.g., a
slow database query or a heavy computation).

**Why it's used:** Reduces latency (in-memory reads are orders of magnitude
faster than disk/database reads) and reduces load on the underlying system
(database, external API).

### 6.1 Where caching happens (know the layers)
- **Client-side cache** (browser cache)
- **CDN** (caches static assets close to users geographically)
- **Application-level cache** (e.g., Redis/Memcached, storing computed
  results or DB query results)
- **Database query cache**

### 6.2 Cache Invalidation — "the hard part"
There's a well-known saying: *"There are only two hard things in Computer
Science: cache invalidation and naming things."* The hard part isn't
storing the cache — it's knowing **when to update/remove** a cached value
once the underlying data changes.

**Common strategies:**
| Strategy | How it works | Trade-off |
|---|---|---|
| **TTL (Time-To-Live)** | Cached value expires automatically after N seconds | Simple, but data can be stale for up to N seconds |
| **Write-through** | Update cache and database together, on every write | Cache always fresh, but writes are slower |
| **Write-behind (write-back)** | Write to cache immediately, persist to DB asynchronously | Fast writes, but risk of data loss if cache fails before DB sync |
| **Cache-aside (lazy loading)** | App checks cache first; on a miss, reads DB and populates cache | Most common pattern; simple, but first request after expiry is always slow ("cache miss penalty") |

**Real-world example relevant to Hevo's domain:** Caching a connector's
schema metadata (e.g., a Salesforce object's field list) so every pipeline
run doesn't have to re-fetch it from the source API — with a TTL so schema
changes eventually get picked up.

```python
# Simple cache-aside pattern illustration
import time

class SchemaCache:
    def __init__(self, ttl_seconds=300):
        self.ttl = ttl_seconds
        self.store: dict[str, tuple[dict, float]] = {}   # key -> (value, timestamp)

    def get_schema(self, connector_id: str, fetch_fn):
        cached = self.store.get(connector_id)
        if cached and (time.time() - cached[1]) < self.ttl:
            return cached[0]                     # cache hit
        schema = fetch_fn(connector_id)           # cache miss -> fetch from source
        self.store[connector_id] = (schema, time.time())
        return schema
```

---

## 7. Databases — SQL vs NoSQL, and key concepts

### 7.1 SQL (Relational) vs NoSQL

| | SQL (Relational) | NoSQL |
|---|---|---|
| **Structure** | Fixed schema, tables with rows/columns | Flexible schema — documents, key-value, wide-column, graph |
| **Relationships** | Strong support via foreign keys, joins | Usually denormalized; relationships handled in application logic |
| **Consistency** | Strong consistency (ACID) by default | Often eventual consistency (varies by DB) |
| **Scaling** | Traditionally vertical (harder to shard) | Designed for horizontal scaling |
| **Examples** | PostgreSQL, MySQL | MongoDB (document), Redis (key-value), Cassandra (wide-column) |
| **When to use** | Structured data, complex queries/joins, strong consistency needs | High write throughput, flexible/evolving schema, massive horizontal scale |

**Why this matters for Hevo specifically:** a data-pipeline platform moves
data *between* both worlds constantly — extracting from relational sources
(MySQL, Postgres) and NoSQL sources (MongoDB) and loading into data
warehouses (Redshift, Snowflake, BigQuery, which are columnar/OLAP
databases, not general-purpose OLTP databases). Knowing this vocabulary
signals real domain awareness in the interview.

### 7.2 ACID (know the acronym and meaning)
- **Atomicity** — a transaction either fully completes or fully fails (no
  partial updates).
- **Consistency** — a transaction moves the database from one valid state
  to another (constraints/rules are never violated).
- **Isolation** — concurrent transactions don't interfere with each other's
  intermediate states.
- **Durability** — once committed, data survives crashes/power loss.

### 7.3 Indexing (brief but important)
**What it is:** A data structure (usually a B-tree) that lets the database
find rows matching a condition **without scanning every row**.

**Why it's used:** Turns an O(n) table scan into roughly O(log n) lookup for
queries filtering/sorting on the indexed column.

**Effect/trade-off:** Indexes speed up reads but slow down writes (every
insert/update must also update the index) and use extra storage — so you
index columns that are frequently queried/filtered, not every column.

### 7.4 Replication and Sharding (conceptual level is enough for SDE I)
- **Replication** — copying data across multiple database servers (a
  primary + replicas) for **fault tolerance** and to **spread read load**
  (reads can go to replicas, writes go to the primary).
- **Sharding** — splitting a single logical database into multiple smaller
  pieces ("shards") across different servers, usually by some key (e.g.,
  `user_id % N`), so each server holds only part of the data — this is how
  systems scale writes horizontally, since a single machine's write
  throughput has a ceiling.

---

## 8. Message Queues & Asynchronous Processing

**What it is:** A **message queue** (e.g., Kafka, RabbitMQ, SQS) is a
component that lets one part of a system (**producer**) send a message/task
without waiting for another part (**consumer**) to process it immediately —
the message sits in the queue until a consumer is ready.

**Why it's used:**
- **Decoupling:** the producer doesn't need to know anything about how/when
  the consumer processes the message — you can change or scale either side
  independently.
- **Handling load spikes:** if requests come in faster than they can be
  processed, the queue absorbs the burst instead of the system falling over.
- **Reliability:** if a consumer crashes mid-processing, the message can be
  redelivered instead of being lost.

**Effect if you *don't* use a queue for a slow operation:** the caller
(e.g., an HTTP request handler) is blocked waiting for a slow task to
finish (e.g., "sync 2 million rows from Salesforce"), leading to timeouts
and poor user experience. Moving that work behind a queue lets you respond
immediately ("sync started") and process it in the background.

**Real-world example, directly relevant to Hevo's domain:** When a user
triggers a pipeline run, the API shouldn't perform the entire data sync
synchronously inside the HTTP request — it should **enqueue a job**
("sync pipeline 42") and return immediately. A pool of background workers
then pulls jobs off the queue and executes the actual extract/load work.
This is precisely the kind of "Job Processing" system covered in the LLD
section — the queue is the piece that sits between "job submitted" and
"job processed."

```python
# Conceptual illustration (not a real queue implementation)
from collections import deque
import threading

class JobQueue:
    def __init__(self):
        self._queue = deque()
        self._lock = threading.Lock()

    def enqueue(self, job):
        with self._lock:
            self._queue.append(job)

    def dequeue(self):
        with self._lock:
            return self._queue.popleft() if self._queue else None

# Producer: API handler
def trigger_pipeline_run(pipeline_id, queue: JobQueue):
    queue.enqueue({"pipeline_id": pipeline_id, "type": "SYNC"})
    return {"status": "queued"}          # returns immediately, doesn't block

# Consumer: background worker (would run in a separate process/thread in reality)
def worker_loop(queue: JobQueue):
    while True:
        job = queue.dequeue()
        if job:
            print(f"Processing {job}")   # actual extract/load logic goes here
```

---

## 9. CAP Theorem (conceptual — know the definition and trade-off)

**What it is:** In a **distributed** system, when a network partition
happens (some servers can't communicate with others), you can only
guarantee **two** of the following three at the same time:
- **C — Consistency:** every read receives the most recent write (or an
  error).
- **A — Availability:** every request receives a response (not
  necessarily the most recent data).
- **P — Partition Tolerance:** the system keeps working despite network
  partitions.

**Why it's framed this way:** In real distributed systems, network
partitions **will** happen eventually — so partition tolerance isn't really
optional. The real, practical trade-off system designers make is between
**Consistency and Availability** during a partition: do you refuse to
answer requests until you're sure the data is fully consistent (CP,
e.g., traditional relational databases with strict replication), or do you
keep answering requests with possibly-stale data (AP, e.g., DNS, many
NoSQL databases)?

**Effect:** This is why you'll hear "eventually consistent" as a term —
AP-leaning systems accept temporary inconsistency in exchange for staying
available, and the data "catches up" (becomes consistent) shortly after.

*(For SDE I, being able to state this trade-off clearly is usually enough —
you're unlikely to be asked to design a CP vs AP system from scratch.)*

---

## 10. Data Pipeline Architecture (Hevo-domain specific — worth knowing well)

Since Hevo's product **is** a data pipeline platform, understanding this
domain vocabulary is a strong signal in interviews — even for an SDE I role
focused on connectors.

### 10.1 ETL vs ELT

| | ETL (Extract, Transform, Load) | ELT (Extract, Load, Transform) |
|---|---|---|
| **Order** | Transform happens *before* loading into the destination | Transform happens *after* loading, inside the destination warehouse |
| **Why** | Historically necessary when destination systems were expensive/limited | Modern cloud warehouses (Snowflake, BigQuery, Redshift) are cheap and powerful enough to do transforms themselves |
| **Trade-off** | Transformation logic lives in a separate processing layer — more control, but another system to maintain | Raw data lands in the warehouse first — faster to get data flowing, transformation can be iterated on later using SQL |

**Why this matters:** Hevo (and most modern data pipeline tools) is
primarily an **ELT** tool — it focuses on reliably extracting from sources
and loading into a warehouse, letting the transformation happen downstream
(sometimes via a separate tool like dbt). Knowing this distinction and being
able to say "Hevo is ELT-oriented" shows you've actually looked at the
product, not just prepped generic system design.

### 10.2 Batch vs Streaming (Real-time) Processing

| | Batch | Streaming |
|---|---|---|
| **What it is** | Data is collected and processed in chunks at intervals (e.g., every hour) | Data is processed continuously, as it arrives, often within seconds |
| **Why** | Simpler, more resource-efficient when near-real-time isn't required | Needed when freshness matters (e.g., fraud detection, live dashboards) |
| **Trade-off** | Higher latency (data can be up to one batch-interval stale) | More complex infrastructure (needs to handle continuous flow, backpressure, out-of-order events) |
| **Example tools** | Scheduled jobs (cron), Airflow | Kafka Streams, Flink, Spark Streaming |

**Why this matters for connector design:** A source connector might support
either mode — e.g., **polling** a REST API every 15 minutes (batch) vs
subscribing to a **webhook/CDC (Change Data Capture)** stream for near-
real-time updates. Being able to discuss "would this connector be
batch-polled or event-driven?" is a very realistic SDE I-level design
question at a company like Hevo.

### 10.3 Change Data Capture (CDC) — worth knowing the term

**What it is:** A technique for capturing **only the rows that changed**
in a source database (inserts/updates/deletes) since the last sync, instead
of re-reading the entire table every time.

**Why it's used:** Re-extracting an entire multi-million-row table on every
sync is slow and wasteful. CDC (often implemented by reading a database's
replication/transaction log) lets a pipeline sync incrementally — much
faster and cheaper.

**Effect:** This is the difference between a **full sync** (re-extract
everything, safe but slow/expensive) and an **incremental sync** (extract
only what's new/changed, fast but needs a reliable way to detect "what
changed" — e.g., an `updated_at` column, or true CDC via the DB's log).

### 10.4 Idempotency and Retries in Pipelines (ties back to REST section)

**Why it matters here specifically:** Data pipelines fail and retry
constantly (network blips, source API rate limits, etc.). If a load
operation isn't idempotent, retrying a failed load could **duplicate rows**
in the destination. This is why real pipeline tools use techniques like
**upserts** (insert-or-update based on a primary key) instead of plain
inserts when loading data — a retry of an upsert has the same end state as
running it once.

---

## 11. Putting It Together — A Worked Mini System Design Answer

**Sample prompt an SDE I candidate might realistically get:** *"Design a
system that lets a user connect a data source (e.g., MySQL) and see it
synced into a destination warehouse on a schedule."*

**How I'd talk through it out loud:**

1. **Clarify scope:** "I'll assume: one source type initially, scheduled
   (batch) syncs, and a simple full-refresh strategy to start — I can
   discuss incremental/CDC as an extension."
2. **Start simple:** `Client (UI) → API Server → Database (pipeline
   configs)`. The API server exposes REST endpoints
   (`POST /pipelines`, `POST /pipelines/{id}/runs`) as covered in Section 3.
3. **Add async processing:** triggering a run shouldn't block the API
   request — enqueue a job onto a **message queue** (Section 8); a pool of
   **worker services** consumes jobs and performs the actual
   extract-and-load.
4. **Add a scheduler:** a separate component (e.g., a cron-like scheduler)
   enqueues "run" jobs automatically based on each pipeline's configured
   frequency.
5. **Add caching where relevant:** cache source schema metadata
   (Section 6) so it's not re-fetched on every run.
6. **Add reliability:** workers should retry failed jobs
   (idempotent loads, Section 10.4), and a **dead-letter queue** for jobs
   that fail repeatedly, so they don't get silently dropped or retried
   forever.
7. **Discuss scaling:** if sync volume grows, scale worker instances
   horizontally (Section 4) behind the queue — since workers are stateless
   consumers, this is straightforward.
8. **Discuss data storage:** pipeline **metadata** (configs, run history)
   lives in a relational DB (strong consistency, structured); actual synced
   **data** lands in the destination warehouse (Section 10.1's ELT point).

This structure — start simple, add one component per stated requirement,
narrate the trade-off of each addition — is exactly what interviewers want
to see, even at a "basics" level.

---

## 12. Common Interview Questions to Be Ready For

- Walk me through what happens when you type a URL into a browser and hit
  enter. (Classic warm-up question — client-server, DNS, HTTP request/
  response at a high level.)
- What's the difference between `PUT` and `PATCH`? Between `PUT` and `POST`?
- What does it mean for an API to be idempotent, and why does it matter for
  retries?
- How would you design pagination for a `list pipelines` endpoint?
- Vertical vs horizontal scaling — trade-offs?
- What's cache invalidation, and what strategies exist to handle it?
- SQL vs NoSQL — when would you pick one over the other?
- What's the difference between batch and streaming data processing, and
  when would you choose each?
- What is ETL vs ELT, and which does a tool like Hevo lean toward?
- Why use a message queue instead of processing everything synchronously?
- What is CDC, and why is it faster than a full table sync?
- Explain the CAP theorem in your own words.

---

## 13. Quick Revision Table

| Concept | One-line definition |
|---|---|
| REST | Architectural style: resources as nouns, HTTP methods as verbs |
| Idempotency | Repeating an operation has the same effect as doing it once |
| Vertical scaling | Add power to one machine |
| Horizontal scaling | Add more machines |
| Load balancer | Distributes traffic across multiple servers |
| Cache | Fast-access layer storing expensive-to-recompute/fetch data |
| Cache-aside | Check cache first; on miss, fetch + populate cache |
| ACID | Atomicity, Consistency, Isolation, Durability — DB transaction guarantees |
| Indexing | Speeds up reads at the cost of slower writes/extra storage |
| Replication | Copies of data across servers for fault tolerance + read scaling |
| Sharding | Splitting data across servers to scale writes |
| Message queue | Decouples producer/consumer, absorbs load spikes, enables async work |
| CAP theorem | Distributed systems trade off Consistency vs Availability during a network partition |
| ETL vs ELT | Transform-before-load vs transform-after-load (modern warehouses favor ELT) |
| Batch vs Streaming | Process in scheduled chunks vs continuously in near real-time |
| CDC | Capture only changed rows instead of re-reading a full table |

---

# SQL & DBMS — Complete Interview Notes

> Why this section matters for Hevo specifically: the product moves data
> between databases and warehouses all day. Even connector-focused SDE I
> work touches schemas, queries, joins, and transactional behavior
> constantly — so this is one of the highest-leverage topics to know well,
> not just "another CS fundamentals checkbox."

---

## 1. What is a DBMS, and why do we use one instead of files?

**Definition:** A Database Management System (DBMS) is software that lets
you **store, retrieve, update, and manage** structured data reliably and
efficiently, instead of managing raw files yourself.

**Why it's used (the problem it solves):** If you tried to store
application data in plain files:
- Concurrent writes from multiple users could corrupt data (no locking).
- No way to enforce rules ("email must be unique") without writing that
  logic yourself, everywhere.
- No efficient way to search/filter without scanning the entire file.
- A crash mid-write could leave data half-written, with no recovery.

A DBMS solves all of this: it enforces structure (schema), provides fast
lookups (indexes), guarantees safety during crashes/concurrent access
(transactions — see ACID below), and gives a powerful query language (SQL)
for retrieving exactly the data you need.

**Effect:** Correctness and reliability become the database's job, not
something every application developer re-implements badly.

---

## 2. Relational Model Basics

**What it is:** Data is organized into **tables** (also called
"relations"), where each table has **columns** (attributes, with a fixed
type) and **rows** (individual records).

- **Primary Key (PK):** a column (or combination) that **uniquely
  identifies** each row in a table. No two rows can share a primary key;
  it cannot be NULL.
- **Foreign Key (FK):** a column in one table that references the primary
  key of another table, **modeling a relationship** between the two.

**Why this matters:** Foreign keys are how relational databases model
real-world connections (a `Ride` belongs to a `Rider`) while keeping data in
one place — this is the foundation of **normalization** (Section 5).

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,
    amount DECIMAL(10, 2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

Here, `orders.user_id` is a foreign key pointing to `users.id` — this
relationship is what a `JOIN` (Section 4) exploits to combine data from both
tables in one query.

---

## 3. Core SQL — Basic to Intermediate

### 3.1 CRUD Operations

```sql
-- CREATE (insert)
INSERT INTO users (id, name, email) VALUES (1, 'Allaudin', 'a@example.com');

-- READ (select)
SELECT name, email FROM users WHERE id = 1;

-- UPDATE
UPDATE users SET email = 'new@example.com' WHERE id = 1;

-- DELETE
DELETE FROM users WHERE id = 1;
```

### 3.2 Filtering, Sorting, Limiting

```sql
SELECT * FROM orders
WHERE amount > 500 AND user_id = 3
ORDER BY amount DESC
LIMIT 10;
```

**Why `LIMIT`/pagination matters (ties to the System Design section):**
Never return unbounded result sets in a real API — the same reasoning from
REST pagination applies directly at the query level.

### 3.3 Aggregate Functions and GROUP BY

**What it is:** Aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`)
compute a single value across multiple rows. `GROUP BY` buckets rows before
aggregating, so you get one aggregate value **per group**.

```sql
-- total order amount per user
SELECT user_id, SUM(amount) AS total_spent, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 1000        -- filters on the aggregated result
ORDER BY total_spent DESC;
```

**Key distinction interviewers check:** `WHERE` filters rows **before**
grouping; `HAVING` filters groups **after** aggregation — you cannot use an
aggregate function inside `WHERE`.

---

## 4. JOINs — Explained Simply, With Effect

**What a JOIN is:** A way to **combine rows from two or more tables** based
on a related column (usually a foreign key → primary key relationship).

**Why it's used:** Normalized databases (Section 5) intentionally split
data across multiple tables to avoid duplication. JOINs are how you
reconstruct a combined view of that data when you need it — e.g., "give me
each order along with the user's name," which requires combining `orders`
and `users`.

### 4.1 Types of JOINs

```sql
-- INNER JOIN: only rows that match in BOTH tables
SELECT o.id, u.name, o.amount
FROM orders o
INNER JOIN users u ON o.user_id = u.id;

-- LEFT JOIN: all rows from the left table, matched rows from the right
-- (unmatched right-side columns come back as NULL)
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- includes users with ZERO orders (o.amount will be NULL for them)

-- RIGHT JOIN: mirror of LEFT JOIN (all rows from the right table)

-- FULL OUTER JOIN: all rows from both sides, matched where possible
```

**Effect / when to use which (a very common interview question):**
- Use **INNER JOIN** when you only care about rows that have a match on
  both sides (e.g., "orders with a valid user").
- Use **LEFT JOIN** when you need every row from the primary table
  regardless of whether a match exists (e.g., "every user, and their
  orders if any — including users who never ordered").

### 4.2 A visual way to remember it
```
INNER JOIN:  A ∩ B              (intersection only)
LEFT JOIN:   all of A + matches from B
RIGHT JOIN:  all of B + matches from A
FULL JOIN:   all of A + all of B, matched where possible
```

---

## 5. Normalization (and When to Denormalize)

**What it is:** The process of organizing tables to **minimize data
duplication** and avoid update anomalies, by splitting data into multiple
related tables.

**Why it's used:** Without normalization, if the same piece of information
(e.g., a customer's address) is duplicated across many rows, updating it
means updating every duplicate — miss one, and your data becomes
inconsistent ("update anomaly").

### 5.1 Normal Forms (know these at a conceptual level)

- **1NF (First Normal Form):** every column holds a single, atomic value —
  no lists/arrays crammed into one column (e.g., don't store
  `"pizza,burger,fries"` in one `orders.items` column).
- **2NF:** 1NF + every non-key column depends on the **whole** primary key
  (relevant mainly for composite keys).
- **3NF:** 2NF + no non-key column depends on **another non-key column**
  (i.e., every column depends only on the primary key, "the whole key and
  nothing but the key").

**Practical example of a 3NF violation and fix:**
```sql
-- BAD (violates 3NF): city_population depends on 'city', not on the order's PK
CREATE TABLE orders_bad (
    id INT PRIMARY KEY,
    user_id INT,
    city VARCHAR(50),
    city_population INT      -- depends on `city`, not on `id` -- duplicated per order!
);

-- GOOD: split city info into its own table
CREATE TABLE cities (
    city VARCHAR(50) PRIMARY KEY,
    population INT
);
CREATE TABLE orders_good (
    id INT PRIMARY KEY,
    user_id INT,
    city VARCHAR(50),
    FOREIGN KEY (city) REFERENCES cities(city)
);
```

### 5.2 When to Denormalize (yes, on purpose)

**What it is:** Deliberately duplicating some data to **avoid expensive
joins**, trading storage/consistency risk for read speed.

**Why/when:** Read-heavy systems (e.g., analytics dashboards, data
warehouses) often denormalize intentionally — this is actually the default
shape of data in OLAP/warehouse systems (Section 7), because join-heavy
queries across normalized tables get slow at large scale, and warehouses
are optimized for scanning wide, flat tables instead.

**This is a great thing to mention for a Hevo interview specifically:**
"A pipeline tool often extracts normalized OLTP data from a source, but the
destination warehouse table may be intentionally denormalized/flattened for
analytics performance — that's a deliberate trade-off, not a mistake."

---

## 6. Transactions and ACID (deeper than the System Design mention)

**What a transaction is:** A group of one or more SQL operations executed
as a **single unit** — either all of them succeed, or none of them take
effect.

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- credit

COMMIT;    -- both succeed together, or...
-- ROLLBACK;   -- ...neither takes effect if something fails
```

**Why it's used:** Without transactions, if the server crashes between the
debit and the credit above, money would simply vanish from the system — a
classic "half-done operation" bug. Transactions guarantee this can't happen.

### 6.1 ACID, explained with this example
- **Atomicity:** both updates happen, or neither does — no "only the debit
  happened" state.
- **Consistency:** the total balance across both accounts stays the same
  before and after (business rule preserved).
- **Isolation:** if another transaction reads the balances mid-transfer, it
  sees either the state *before* or *after* the transfer — never a
  half-updated state.
- **Durability:** once `COMMIT` succeeds, the change survives even if the
  server crashes immediately after.

### 6.2 Isolation Levels (know these exist, at a conceptual level)
Databases let you tune how strictly "Isolation" is enforced, trading
correctness guarantees for performance:
- **Read Uncommitted** — can see other transactions' uncommitted changes
  ("dirty reads") — rarely used.
- **Read Committed** — only sees committed data (common default, e.g.
  PostgreSQL).
- **Repeatable Read** — same query run twice in one transaction returns the
  same rows.
- **Serializable** — strictest; transactions behave as if run one at a
  time, sequentially.

**Effect:** Stricter isolation = safer but slower (more locking/blocking);
looser isolation = faster but more prone to subtle concurrency bugs. This
mirrors the CAP theorem's consistency-vs-performance trade-off discussed in
the System Design section.

---

## 7. OLTP vs OLAP (directly relevant to Hevo's product)

| | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| **Purpose** | Day-to-day application operations (create an order, update a user) | Analytics, reporting, business intelligence |
| **Query pattern** | Many small, fast reads/writes affecting few rows | Fewer, complex queries scanning/aggregating many rows |
| **Schema shape** | Normalized (Section 5) | Often denormalized/star-schema, columnar storage |
| **Examples** | MySQL, PostgreSQL powering an app's backend | Snowflake, BigQuery, Redshift |

**Why this matters for Hevo specifically:** this is *exactly* the boundary
a data pipeline tool sits on — **extracting** from OLTP sources (an app's
production MySQL database) and **loading** into an OLAP destination
(Snowflake/BigQuery/Redshift) for analytics. Being able to say this clearly
in an interview shows real understanding of the product's purpose, not just
generic DBMS trivia.

**Columnar vs row storage (a nice specific detail to mention):**
OLTP databases typically store data **row-by-row** on disk (fast to fetch/
update one full record at a time — good for `orders` app queries).
OLAP warehouses typically store data **column-by-column** (fast to scan
just the columns a query needs across millions of rows, e.g.
`SUM(revenue)` — doesn't need to touch unrelated columns) — this is a big
part of why warehouses are fast at analytics-style aggregation queries.

---

## 8. Indexing — Deeper Dive (builds on the System Design mention)

**What it is (recap):** A separate data structure (commonly a B-tree, or a
hash index for equality-only lookups) that lets the database find matching
rows without scanning the whole table.

**Why/effect (recap):** Turns O(n) scans into ~O(log n) lookups for indexed
columns — at the cost of slower writes (every insert/update must also
maintain the index) and extra storage.

### 8.1 What to index (a practical, interview-relevant checklist)
- Columns frequently used in `WHERE` clauses.
- Columns used in `JOIN ... ON` conditions (foreign keys — many databases
  don't auto-index FK columns, a common gotcha to mention).
- Columns used in `ORDER BY` (an index can let the DB skip a separate sort
  step).

### 8.2 Composite Indexes
An index can span multiple columns. **Column order matters** — an index on
`(user_id, created_at)` efficiently serves queries filtering on `user_id`
alone, or on `user_id` AND `created_at` together, but **not** efficiently
on `created_at` alone (this is a genuinely common interview gotcha —
"leftmost prefix" rule).

```sql
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at);

-- Uses the index efficiently:
SELECT * FROM orders WHERE user_id = 5 AND created_at > '2026-01-01';

-- Does NOT use this index efficiently (created_at isn't the leftmost column):
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

### 8.3 EXPLAIN — how to actually check if an index is being used
Most databases support an `EXPLAIN` (or `EXPLAIN ANALYZE`) command that
shows the **query plan** — whether the database used an index scan or fell
back to a full table scan. Mentioning "I'd check the query plan with
`EXPLAIN`" in an interview is a strong, practical signal.

---

## 9. Window Functions (intermediate/advanced — a good differentiator)

**What it is:** Functions that perform a calculation **across a set of rows
related to the current row**, without collapsing the result into a single
row per group (unlike `GROUP BY`).

**Why it's used:** Lets you compute things like "rank within a group" or
"running total" while still returning **every original row** — something
plain aggregation can't do.

```sql
-- Rank each user's orders by amount, per user
SELECT
    user_id,
    id AS order_id,
    amount,
    RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rank_in_user,
    SUM(amount) OVER (PARTITION BY user_id) AS user_total
FROM orders;
```

**Effect:** `PARTITION BY` divides rows into groups (like `GROUP BY`), but
`OVER (...)` keeps each individual row visible in the output, with the
window calculation attached alongside it — very useful for "top N per
group" queries, running totals, and moving averages, all common in
analytics use cases (again, relevant to a company shipping data into
analytical warehouses).

---

## 10. Common Query-Writing Patterns Worth Practicing

```sql
-- Find duplicate emails
SELECT email, COUNT(*) 
FROM users 
GROUP BY email 
HAVING COUNT(*) > 1;

-- Second highest value (a classic interview question)
SELECT MAX(amount) 
FROM orders 
WHERE amount < (SELECT MAX(amount) FROM orders);

-- Users with no orders (anti-join pattern)
SELECT u.* 
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;

-- Upsert (insert-or-update) -- directly relevant to idempotent pipeline
-- loads discussed in the System Design section
INSERT INTO users (id, name, email)
VALUES (1, 'Allaudin', 'a@example.com')
ON CONFLICT (id) DO UPDATE
SET name = EXCLUDED.name, email = EXCLUDED.email;
```

---

## 11. Common Interview Questions to Be Ready For

- What's the difference between `WHERE` and `HAVING`?
- Explain `INNER JOIN` vs `LEFT JOIN` with an example, including what
  happens to unmatched rows.
- What is normalization, and when would you intentionally denormalize?
- Explain ACID with a real transaction example (money transfer is the
  classic one — know it cold).
- What's the difference between a primary key and a unique key/constraint?
- How does indexing work, and what's the trade-off of adding one?
- What's the "leftmost prefix" rule for composite indexes?
- Difference between OLTP and OLAP, and where would a tool like Hevo sit
  relative to both?
- What is a window function, and how is it different from `GROUP BY`?
- How would you find duplicate rows in a table? The second-highest value?
- What's an upsert, and why does it matter for retry-safe data loading?

---

## 12. Quick Revision Table

| Concept | One-line definition |
|---|---|
| Primary Key | Uniquely identifies a row; not NULL, not duplicated |
| Foreign Key | Column referencing another table's primary key; models relationships |
| INNER JOIN | Only rows matching in both tables |
| LEFT JOIN | All rows from the left table + matches from the right (NULLs where unmatched) |
| GROUP BY / HAVING | Bucket rows into groups / filter on the aggregated result |
| Normalization | Split data to remove duplication and avoid update anomalies |
| Denormalization | Intentionally duplicate data to speed up reads (common in OLAP) |
| Transaction | A group of operations that succeed or fail together |
| ACID | Atomicity, Consistency, Isolation, Durability |
| Isolation levels | Trade-off between concurrency safety and performance |
| OLTP | Transactional, normalized, row-oriented — powers applications |
| OLAP | Analytical, denormalized, columnar — powers reporting/warehouses |
| Index | Speeds up reads on a column at the cost of slower writes + storage |
| Composite index | Multi-column index; leftmost-prefix rule applies |
| Window function | Per-row calculation over a related set of rows, without collapsing rows |
| Upsert | Insert-or-update by key; makes retries/loads idempotent |

---

# Operating Systems & Concurrency Basics — Complete Interview Notes (Python)

> Why this section matters: Hevo's own SDE I posting explicitly calls out
> "eagerness to learn concurrency and performance basics" and "debug and
> optimize code for data pipeline efficiency." A pipeline tool syncing many
> sources at once is fundamentally a concurrency problem — this section
> covers what you need to speak intelligently about it, with Python-specific
> detail since that's your language of choice.

---

## 1. Process vs Thread

**What a process is:** An independent running instance of a program, with
its **own memory space**, given its own resources by the OS (memory, file
handles, etc.). Two processes cannot directly read/write each other's
memory.

**What a thread is:** A unit of execution **within** a process. A single
process can have multiple threads, and all threads in the same process
**share the same memory space** (same variables, same heap).

**Why the distinction matters:**
| | Process | Thread |
|---|---|---|
| **Memory** | Isolated (own address space) | Shared (with other threads in the same process) |
| **Creation cost** | Expensive (OS allocates a full new environment) | Cheap (lightweight, shares existing process resources) |
| **Communication** | Needs explicit IPC (pipes, sockets, shared memory) | Direct — just shared variables (but needs synchronization, see Section 3) |
| **Crash isolation** | One process crashing doesn't crash another | One thread crashing can bring down the whole process |
| **Use case** | CPU-bound parallel work, isolation/safety needs | I/O-bound concurrent work, lightweight tasks needing shared state |

**Real-world example relevant to Hevo:** if a pipeline tool syncs 10 data
sources at once, it could spin up 10 **threads** (cheap, share config/
connection-pool state easily, good fit if the work is mostly waiting on
network I/O) or 10 **processes** (heavier, but fully isolated — one source's
crash/memory leak can't affect the others). Which one is "right" depends on
whether the work is I/O-bound or CPU-bound — covered next.

---

## 2. Concurrency vs Parallelism (a frequently confused pair — know this cold)

**Concurrency:** Structuring a program to **handle multiple tasks
overlapping in time** — doesn't necessarily mean they run at the exact same
instant. Example: a single chef juggling three dishes, switching between
them while things simmer.

**Parallelism:** Actually **executing multiple tasks at the literal same
instant**, which requires multiple CPU cores. Example: three chefs, each
cooking one dish simultaneously.

**Why the distinction matters:** You can have concurrency **without**
parallelism (e.g., a single-core machine can still run multiple threads
"concurrently" by rapidly switching between them — this is what Python's
GIL allows, see Section 4). You need multiple cores/processes for true
parallelism.

**Effect on design decisions:** If your bottleneck is **waiting** (network
calls, disk I/O), concurrency alone (even on one core) gives a huge speedup
— while one task waits, another can run. If your bottleneck is **computation**
(CPU-heavy work), you need true parallelism (multiple cores) to get a
speedup — concurrency alone won't help, since there's no "waiting" to
overlap with.

---

## 3. Race Conditions and Synchronization

**What a race condition is:** A bug that occurs when multiple
threads/processes **access and modify shared data at the same time**, and
the final result depends on the unpredictable timing/order of execution —
"racing" to read/write first.

**Why it's dangerous:** The bug is often **non-deterministic** — it might
not show up in testing, then appear randomly in production under load,
making it one of the hardest bug classes to reproduce and fix.

**Classic example:**
```python
# BAD: race condition on a shared counter
import threading

counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1          # NOT atomic! read-modify-write in 3 steps

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # Expected 200000, but often prints something LESS
                  # -- two threads can both read the same old value
                  # before either writes back the incremented result
```

**Why `counter += 1` isn't safe:** it's actually three separate steps —
read `counter`, add 1, write back. If Thread A reads `counter=5`, then
Thread B also reads `counter=5` before A writes back, both threads compute
6 and write 6 — one increment is silently lost.

### 3.1 The Fix: Locks (Mutexes)

**What it is:** A **lock** (mutual exclusion / "mutex") ensures only **one**
thread can execute a critical section of code at a time — other threads
must wait until the lock is released.

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100000):
        with lock:              # only one thread inside this block at a time
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()

print(counter)   # Now reliably 200000
```

**Effect/trade-off:** Locks fix correctness but introduce **contention** —
threads waiting on a lock aren't doing useful work, so overusing locks (or
locking too broad a section of code) can eliminate the performance benefit
you wanted from concurrency in the first place. A good design **locks the
smallest section possible** ("minimize the critical section").

### 3.2 Deadlocks

**What it is:** A situation where two or more threads are each waiting for
a resource the other holds, so **none of them can ever proceed**.

**Classic example:** Thread A holds Lock 1 and waits for Lock 2. Thread B
holds Lock 2 and waits for Lock 1. Neither ever releases what it holds —
permanent freeze.

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()

def task_1():
    with lock_a:
        with lock_b:        # if task_2 grabbed lock_b first, we deadlock here
            pass

def task_2():
    with lock_b:
        with lock_a:        # opposite order from task_1 -> classic deadlock setup
            pass
```

**How to avoid it (know this for interviews):** always **acquire locks in a
consistent, global order** across your whole codebase (e.g., always lock
`lock_a` before `lock_b`, everywhere) — this alone prevents most deadlocks.

---

## 4. Python's GIL (Global Interpreter Lock) — a very common Python-specific
interview question

**What it is:** The GIL is a lock in CPython (the standard Python
implementation) that ensures **only one thread executes Python bytecode at
a time**, even on a multi-core machine.

**Why it exists:** CPython's memory management (reference counting for
garbage collection) isn't thread-safe by default. The GIL is a
simplification that avoids needing fine-grained locks around every object,
at the cost of preventing true multi-core parallelism for pure-Python code.

**Effect — the practical takeaway that interviewers actually want:**
- **I/O-bound work** (network calls, file reads, waiting on a database) →
  Python `threading` still helps a lot, because the GIL is **released**
  while a thread waits on I/O — other threads can run during that wait.
- **CPU-bound work** (heavy computation, tight loops, data crunching) →
  Python `threading` does **NOT** give a speedup, because the GIL prevents
  two threads from executing Python bytecode simultaneously, even on 8
  cores. For this, you need `multiprocessing` (separate processes, each
  with its own GIL/interpreter) to get true parallelism.

```python
# I/O-bound: threading helps (GIL released during network/file wait)
import threading, requests

def fetch(url):
    requests.get(url)     # GIL released while waiting on the network

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
# ... this genuinely speeds things up

# CPU-bound: threading does NOT help -- use multiprocessing instead
from multiprocessing import Pool

def heavy_computation(n):
    return sum(i * i for i in range(n))

with Pool(processes=4) as pool:
    results = pool.map(heavy_computation, [10_000_000] * 4)
    # each process has its OWN GIL -- this actually uses all 4 cores
```

**Real-world tie-in for Hevo:** syncing data from a source is largely
**I/O-bound** (waiting on network calls to the source API/database) — so
`threading` (or `asyncio`, see below) is usually the right tool, not
`multiprocessing`. If a pipeline also does heavy in-memory transformation
of large datasets, that CPU-bound part might benefit from
`multiprocessing` instead.

---

## 5. Async/Await (asyncio) — the modern alternative for I/O-bound work

**What it is:** A concurrency model based on a **single thread** that
**cooperatively switches** between tasks whenever one task hits an `await`
point (e.g., waiting on a network response) — instead of the OS switching
between threads.

**Why it's used:** For very high numbers of concurrent I/O-bound tasks
(think: syncing hundreds of API endpoints), threads become expensive (each
thread still has memory/OS overhead). `asyncio` can juggle thousands of
concurrent tasks on a single thread far more cheaply, since there's no
OS-level thread-switching cost — just fast, cooperative function-call-like
switching.

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)   # runs all concurrently

# asyncio.run(fetch_all(list_of_source_urls))
```

**Effect/trade-off vs threading:**
- `asyncio` scales to far more concurrent I/O tasks with less overhead than
  `threading`.
- But it requires the **entire call chain** to be async-aware (a single
  blocking/synchronous call anywhere in the chain stalls the whole event
  loop) — this is the most common practical gotcha, and worth mentioning if
  asked to compare the two.
- `threading` is simpler to reason about for a moderate number of
  concurrent tasks and works fine with existing synchronous libraries.

**How to summarize this cleanly in an interview:** "For I/O-bound
concurrency in Python: `threading` is simple and works with existing
sync code, but has per-thread overhead; `asyncio` scales further with less
overhead but requires the whole call chain to be async. For CPU-bound work,
neither helps because of the GIL — you need `multiprocessing`."

---

## 6. Memory Management Basics (know the concepts, not implementation
internals)

**Stack vs Heap:**
- **Stack:** stores local variables and function call frames; memory is
  automatically reclaimed when a function returns — fast, but limited size
  (deep recursion can cause a "stack overflow").
- **Heap:** stores dynamically allocated objects (most Python objects live
  here); memory must be explicitly tracked and reclaimed — slower to
  allocate, but flexible size.

**Garbage Collection (GC):** the process of automatically freeing memory for
objects that are no longer reachable/used by the program, so developers
don't have to manually free memory (unlike C/C++).

**Python's approach (good to know at a high level):**
- **Reference counting:** every object tracks how many references point to
  it; when the count hits zero, it's freed immediately.
- **Cycle-detecting GC:** reference counting alone can't free objects that
  reference each other in a cycle (e.g., A references B, B references A,
  but nothing else references either) — Python runs a periodic cycle
  detector to catch and clean these up.

**Effect for you as a developer:** you rarely manage memory directly in
Python, but understanding reference cycles helps you reason about memory
leaks in long-running processes (relevant for a pipeline worker process
that runs continuously).

---

## 7. Producer-Consumer Pattern (ties directly to the System Design queue
section)

**What it is:** A concurrency pattern where one or more **producer**
threads/processes generate work items and put them into a shared queue,
while one or more **consumer** threads/processes take items off the queue
and process them.

**Why it's used:** Decouples the speed of producing work from the speed of
consuming it — the queue absorbs the difference. This is the same idea as
the message-queue pattern from the System Design section, just implemented
in-process with threads instead of across services.

```python
import threading
import queue
import time

work_queue = queue.Queue()     # thread-safe queue -- handles locking internally

def producer():
    for i in range(10):
        work_queue.put(f"job-{i}")
        time.sleep(0.1)
    work_queue.put(None)       # sentinel value signaling "no more work"

def consumer():
    while True:
        job = work_queue.get()
        if job is None:
            break
        print(f"Processing {job}")

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)
t1.start(); t2.start()
t1.join(); t2.join()
```

**Why `queue.Queue` specifically:** Python's built-in `queue.Queue` is
already **thread-safe** — it handles the locking internally, so you don't
need to manually wrap `put()`/`get()` calls with your own `Lock`. Mentioning
this in an interview signals you know not to reinvent synchronization when
a safe standard-library tool already exists.

---

## 8. Common Interview Questions to Be Ready For

- What's the difference between a process and a thread?
- What's the difference between concurrency and parallelism? Give an
  example of each.
- What is a race condition? How do you fix one?
- What is a deadlock, and how do you prevent it?
- What is Python's GIL, and how does it affect threading vs multiprocessing
  decisions?
- When would you choose `threading` vs `multiprocessing` vs `asyncio` in
  Python?
- What is the producer-consumer pattern, and where have you seen it used?
- What's the difference between the stack and the heap?
- How does Python manage memory / garbage collection?
- If you were syncing 50 data sources concurrently, how would you design
  that? (Great chance to combine this section with System Design Section 8
  — queue + worker pool + concurrency model choice.)

---

## 9. Quick Revision Table

| Concept | One-line definition |
|---|---|
| Process | Independent running program with its own isolated memory |
| Thread | Lightweight unit of execution sharing memory within a process |
| Concurrency | Multiple tasks overlapping in time (not necessarily simultaneous) |
| Parallelism | Multiple tasks executing at the literal same instant (needs multiple cores) |
| Race condition | Bug from unsynchronized concurrent access to shared data |
| Lock/Mutex | Ensures only one thread runs a critical section at a time |
| Deadlock | Threads waiting on each other's locks forever; prevent via consistent lock ordering |
| GIL | CPython lock allowing only one thread to run Python bytecode at a time |
| Threading (Python) | Good for I/O-bound work (GIL released during waits) |
| Multiprocessing (Python) | Needed for CPU-bound work (true multi-core parallelism) |
| asyncio | Single-thread cooperative concurrency; scales to many I/O tasks cheaply |
| Producer-Consumer | Pattern decoupling work generation from work processing via a shared queue |

---

# Behavioral / HR Round — Complete Interview Notes

> Why this section matters: from the Glassdoor/Blind reports gathered
> earlier, Hevo's process regularly includes a "work history" round (often
> with a VP Engineering or the CTO) and a separate HM/managerial round with
> behavioral questions. Candidates who ace DSA/LLD but stumble here lose
> offers — this round is graded just as seriously, and it's the most
> under-prepared-for round because people assume "just be yourself" is
> enough. It isn't; structure matters as much as substance.

---

## 1. What a behavioral round actually tests, and why it exists

**What it is:** A round where, instead of solving a technical problem live,
you're asked to describe **real past experiences** — how you handled
conflict, made a decision under uncertainty, dealt with failure, worked with
a difficult teammate, etc.

**Why companies run it:** Technical rounds test *can you do the work*.
Behavioral rounds test *how you'll actually behave day-to-day* — how you
communicate, whether you take ownership, how you handle ambiguity and
pressure, whether you're coachable, and whether you'll be someone the team
wants to work with. For a startup like Hevo (fast-moving, high-ownership
environment per the reviews gathered earlier), this matters even more than
at a larger, more process-heavy company.

**Effect of skipping preparation:** Even strong engineers ramble, give vague
answers ("I'm a team player"), or tell a story with no clear point — leaving
the interviewer unable to assess anything concrete. A well-prepared
candidate with slightly weaker technical rounds often outperforms a
technically stronger candidate who fumbles this round, because interviewers
weigh "will this person be effective and easy to work with" heavily.

---

## 2. The STAR Method — the core framework

**What it is:** A structure for answering any behavioral question so it
stays concise, concrete, and clearly demonstrates your role and impact.

- **S — Situation:** brief context. What was going on? (1-2 sentences)
- **T — Task:** what was YOUR specific responsibility or goal in that
  situation? (1 sentence)
- **A — Action:** what did YOU specifically do? (the bulk of your answer —
  be specific, use "I," not "we")
- **R — Result:** what happened? Quantify if possible. What did you learn?

**Why it's used:** Without structure, people either ramble for 5 minutes
with no clear point, or give a one-line non-answer. STAR forces a complete,
scannable story the interviewer can actually evaluate — they're listening
for exactly these four pieces, often literally taking notes against them.

**Effect of NOT specifying "I" vs "we":** A very common mistake is telling a
story entirely in "we" language ("we decided," "we fixed it") — this makes
it impossible for the interviewer to tell what **you** actually contributed
versus what the team did. Always be explicit: "I proposed X, then I
implemented Y, while my teammate handled Z."

### 2.1 Worked example of STAR in action

**Question:** "Tell me about a time you disagreed with a technical
decision."

**Weak answer (no structure):** "Yeah, there was this one time on a project
where me and another guy didn't agree on how to build something, so we
talked about it and eventually just went with his idea because he had more
experience I guess."

**Strong answer (STAR):**
> "**Situation:** During a college project, I was building a REST API for a
> group assignment, and a teammate wanted to store all configuration data
> as JSON blobs in a single database column instead of separate tables.
> **Task:** I was responsible for the data layer and was concerned this
> would make querying and validating data much harder later.
> **Action:** Instead of just pushing back, I built a small proof-of-concept
> showing both approaches querying the same sample data, and shared the
> query complexity difference concretely rather than arguing abstractly. I
> also asked what problem the JSON approach was solving for him — turned out
> he was worried about frequent schema changes.
> **Result:** We agreed on a middle ground — structured tables for stable
> fields, with one JSON column only for a few genuinely dynamic fields. The
> project shipped on time, and I learned that showing a concrete comparison
> beats arguing from opinion alone."

**Why this version works:** specific situation, clear ownership of "I,"
concrete action (not vague "we talked"), and a result with a lesson learned
— exactly the shape interviewers are scanning for.

---

## 3. Preparing Your Story Bank (do this BEFORE the interview, not during)

**What it is:** A set of 6-8 real experiences from your life (projects,
internships, coursework, personal projects) that you've pre-structured in
STAR format, each one flexible enough to answer multiple possible questions.

**Why you must prepare this in advance:** Trying to think of a good example
live, under pressure, while also structuring it into STAR is very hard —
most people either freeze or ramble. Having a small bank of solid stories
ready means you're just *recalling and adapting*, not *inventing*.

### 3.1 Map your stories to these common categories
You don't need a different story for every possible question — most
behavioral questions map to a handful of underlying themes. Prepare **one
strong story per category**, and be ready to angle it differently depending
on the exact question:

| Category | Typical questions it answers |
|---|---|
| **Conflict/disagreement** | "Tell me about a disagreement with a teammate/senior" |
| **Failure/mistake** | "Tell me about a time you failed" / "a mistake you made" |
| **Ownership/initiative** | "Tell me about a time you went above and beyond" |
| **Ambiguity/uncertainty** | "Tell me about a time requirements were unclear" |
| **Deadline pressure** | "Tell me about a tight deadline you had to hit" |
| **Learning something new** | "Tell me about learning a new technology quickly" |
| **Teamwork** | "Tell me about working with a difficult teammate" |
| **Leadership/influence** | "Tell me about leading without formal authority" |

**Since you're preparing a portfolio RAG project for interviews** (relevant
to your broader job search), that project is a genuinely strong source for
several of these — e.g., "learning something new" (RAG engineering itself),
"ambiguity" (deciding architecture with no single right answer), and
"ownership" (building something end-to-end on your own initiative). Have at
least one story from it ready, structured in STAR, before the interview.

---

## 4. Handling the "Tell me about a time you failed" Question Well

**Why this specific question deserves its own section:** it's asked
constantly and answered badly constantly. The two failure modes:
1. **Fake humility:** "My biggest weakness is I work too hard" — interviewers
   see through this immediately and it actively hurts you.
2. **A genuinely damaging failure** with no recovery/learning shown — picking
   a story that makes you look reckless or that you clearly haven't grown
   from.

**What makes a good answer:** pick a **real, moderate-stakes** failure
(not catastrophic, not fake), own it directly without excessive
self-flagellation OR excuse-making, and spend **more time on what you
changed afterward** than on the failure itself.

**Structure specifically for this question:**
- What happened (briefly, own it: "I underestimated X and it caused Y")
- What you did in the moment to address it
- **What you changed going forward** (this is the part that actually
  matters to the interviewer — they're testing whether you learn from
  mistakes, not whether you've ever made one)

---

## 5. Common Behavioral Questions — With Guidance on How to Answer

### "Tell me about yourself"
**What they're actually testing:** can you give a clear, concise
professional narrative — not your life story. This sets the tone for the
whole interview.
**Structure:** Present (what you're doing/building now) → Past (relevant
background that led here, briefly) → Future (why this role/company makes
sense next). Keep it under 90 seconds. Avoid reciting your resume line by
line — the interviewer already has it.

### "Why do you want to work at Hevo?"
**What they're testing:** did you actually research the company, or are you
applying everywhere with a copy-pasted answer?
**How to prepare a strong answer:** reference something specific and true
— e.g., that Hevo is a no-code ELT data pipeline platform serving companies
like Zepto, Curefit, and Groww, and connect that to something concrete about
your interest (e.g., wanting to work on infrastructure/data problems at
scale, or specific interest in the connector/integrations space mentioned
in the SDE I role). Avoid generic answers like "I heard it's a great
culture" with nothing to back it up.

### "Why should we hire you?"
**Structure:** connect 2-3 concrete things you bring (skills, a relevant
project, a specific strength) directly to what the role needs — don't just
list adjectives ("hardworking," "passionate"). Be specific: "I've built [X]
which required [skill directly relevant to connector/API work], and I'm
comfortable with [Y] which the role calls for."

### "Where do you see yourself in 3-5 years?"
**What they're testing:** realistic ambition and whether you'll stick
around long enough to be worth investing in, without sounding either
directionless or like you'll leave for the first bigger offer.
**Good framing:** talk about growing technically (going deeper into
systems/backend engineering, taking more ownership over time) rather than
a specific job title — titles sound presumptuous for an SDE I candidate and
can read as inauthentic.

### "Tell me about a time you had to learn something quickly"
Use your RAG portfolio project here if it fits — describe the actual
learning process (how you approached an unfamiliar area), not just the
end result.

### "How do you handle feedback/criticism?"
**What they're testing:** coachability — this matters enormously for an
SDE I role where you'll be mentored closely (the role description mentions
"under mentorship" for debugging/optimization). Give a real example where
feedback changed your approach, and be genuine about it — don't claim you
"love" all criticism; show that you take it seriously and act on it.

### "Do you have any questions for us?"
**Never skip this — always have 2-3 ready.** This is also being evaluated:
asking nothing signals low genuine interest. Good categories:
- About the role: "What does a typical first 90 days look like for an SDE
  I on this team?"
- About the team: "How is the connector/integrations team structured, and
  how do new engineers usually ramp up?"
- About growth: "What separates someone who's doing well at 6 months from
  someone who's struggling?"
Avoid asking about things easily found online (basic company info) or
leading with compensation/benefits questions in a technical round.

---

## 6. Company-Specific Context Worth Knowing (from real candidate reports)

Being aware of publicly reported patterns lets you prepare thoughtful
answers and ask informed questions — not to be negative, but to show you've
done real research and can speak to it maturely if it comes up.

- Multiple reviews describe Hevo as a **fast-paced, high-ownership**
  environment, with one report describing expectations as higher-pressure
  than some big tech companies for a comparable role. If asked "how do you
  handle a demanding pace," having a genuine, structured answer ready (see
  Section 5's feedback/pressure questions) is worth preparing rather than
  encountering the question cold.
- Interview process length and quality have been reported as **variable**
  across candidates — some had smooth, well-structured experiences; others
  reported inconsistent communication. If your own process feels
  disorganized at some point, that's a reported pattern, not necessarily a
  reflection on you — stay professional and keep performing well regardless.
- The work-history round is often taken by senior people (VP Engineering,
  CTO) — prepare your project/resume walkthrough (Section 7) especially
  carefully, since it's not just a formality.

---

## 7. Preparing Your Resume/Project Walkthrough (the "work history" round)

**What it is:** A round specifically focused on your past projects and
experience — reported explicitly in Hevo's process (a "Technical
experience round" / "Work history" round with a senior engineer, VP
Engineering, or CTO).

**Why it's a distinct skill from STAR-style Q&A:** here, YOU largely drive
the narrative — you'll be asked to "walk me through this project" and need
to explain it clearly to someone hearing about it for the first time,
including the "why," not just the "what."

**Structure for walking through any project (practice this out loud for
your RAG portfolio project specifically):**
1. **The problem** — what were you actually trying to solve, and why did it
   matter? (1-2 sentences)
2. **Your approach** — the key architectural/design decisions, and briefly
   *why* you chose them over alternatives (this is what interviewers really
   want — decision-making, not just "I used X library")
3. **The hardest part** — be ready for "what was the most challenging part
   of this?" — pick something real and explain how you solved it
4. **The result** — what works now, what you'd improve with more time
   (showing self-awareness about limitations is a strength, not a weakness)
5. **What you'd do differently** — a great signal of maturity; have a real
   answer ready, not "nothing, it was perfect"

**Common follow-up traps to prepare for:**
- "Why did you choose X over Y?" — for every major technical decision in
  your project, be ready to explain the alternative you considered and why
  you didn't pick it.
- "What would happen if [some edge case]?" — think through 2-3 realistic
  edge cases in your own project beforehand.
- "How would you scale this?" — a great chance to bridge into the System
  Design section's vocabulary (caching, queues, horizontal scaling) applied
  to your own project specifically.

---

## 8. Practical Preparation Checklist

- [ ] Write out 6-8 STAR stories covering the categories in Section 3.1
- [ ] Practice saying each one out loud, timed — aim for 60-90 seconds per
      story, not 5 minutes
- [ ] Prepare your RAG portfolio project walkthrough using the Section 7
      structure — practice it out loud, not just in your head
- [ ] Prepare 3 genuine questions to ask the interviewer (Section 5)
- [ ] Research Hevo's product specifically — be able to say in one sentence
      what Hevo does and who its customers are (see System Design Section
      10 for the ETL/ELT framing that ties directly into this)
- [ ] Prepare a concise, structured answer to "tell me about yourself"
      (Present → Past → Future), timed under 90 seconds

---

## 9. Common Interview Questions to Be Ready For (quick reference list)

- Tell me about yourself.
- Why do you want to work at Hevo? Why this role?
- Tell me about a time you disagreed with someone technically.
- Tell me about a time you failed, and what you learned.
- Tell me about a time you had to learn something new quickly.
- Tell me about a time you faced a tight deadline.
- Tell me about a time requirements were unclear or changed midway.
- How do you handle receiving critical feedback?
- Tell me about a time you took ownership of something beyond your
  explicit responsibility.
- Walk me through a project you're proud of. (See Section 7 structure.)
- Where do you see yourself in a few years?
- Do you have any questions for us?

---

## 10. Quick Revision Table

| Concept | One-line definition |
|---|---|
| STAR method | Situation, Task, Action, Result — structures any behavioral answer |
| "I" vs "we" | Always be explicit about YOUR specific contribution |
| Story bank | 6-8 pre-structured real stories mapped to common question categories |
| Failure question | Pick a real, moderate-stakes failure; spend more time on the fix than the failure |
| Work-history round | You drive the narrative; be ready to justify every major decision + edge cases |
| Closing questions | Always have 2-3 genuine ones ready; never skip this |

---

## 11. Personalized Mock-Interview Answers (Based on Allaudin's Resume)

> These are written in first person, as if you (Allaudin) are answering live
> in the interview. They're grounded in your actual resume — Qclairvoyance
> Quantum Labs internship, the AI-Powered Portfolio Optimization Engine,
> LexAI, and the URL Shortener project — so they hold up if an interviewer
> digs into specifics. Read each one, then **say it out loud from memory**,
> adjusting the wording to sound like you, not like a script. Where a
> detail isn't on your resume (e.g., a specific team conflict), it's flagged
> so you can swap in a true detail rather than reciting an invented one.

---

### 1. "Tell me about yourself"

"I'm a Computer Science student at SRM Institute of Science and Technology,
specializing in AI/ML with a minor focus on Quantum Computing, currently
maintaining a 9.09 GPA. Over the last year I've focused on building
production-grade systems rather than just notebooks: I built LexAI, a
RAG-based legal question-answering system for Indian law using hybrid
FAISS+BM25 retrieval and an async FastAPI backend; a reinforcement-learning
portfolio optimization engine using PPO and SAC agents with a full MLOps
pipeline; and a horizontally scalable URL shortener with Redis caching and
Celery-based async analytics. I also recently interned at Qclairvoyance
Quantum Labs, formulating QUBO optimization models for molecular docking.
Right now I'm looking for an SDE I role where I can apply that same
production-engineering rigor to real infrastructure problems — which is
what drew me to Hevo."

---

### 2. "Why do you want to work at Hevo? Why this role?"

"Hevo is a no-code ELT data pipeline platform, and what draws me
specifically is the connector and integrations problem space — reliable
extraction from dozens of heterogeneous sources, idempotent loading, schema
drift handling. In my URL Shortener project I already worked with a lot of
the underlying pieces — Redis caching, Celery for decoupling async work
from the request path, Prometheus/Grafana monitoring, GitHub Actions CI/CD
against real PostgreSQL and Redis containers. I want to apply that same
thinking to a real production data pipeline at scale, with a team, rather
than just in a personal project. The SDE I role specifically appeals to me
because it's backend/infrastructure-focused with mentorship — exactly where
I want to go deeper next."

---

### 3. "Tell me about a time you disagreed with someone technically"

*(Your resume doesn't show a team-conflict moment explicitly — this is
written around a real technical decision point from LexAI. If you actually
had a disagreement with a mentor/teammate on any project, swap in that real
detail; interviewers value a genuine interpersonal disagreement more than a
solo design decision.)*

"**Situation:** While building LexAI's retrieval pipeline, my first plan
was pure dense retrieval using FAISS alone, since that's the standard
approach for RAG systems. **Task:** I needed to lock in the retrieval
strategy before building the ingestion pipeline, since switching later
would mean re-indexing everything. **Action:** I tested it against sample
legal queries and noticed dense retrieval kept missing exact statute
numbers and case citations — embeddings are good at semantic similarity,
not exact-token matching. So I pushed back on my own initial plan and
added BM25 alongside FAISS, fused via Reciprocal Rank Fusion, plus a
CrossEncoder reranker. **Result:** the hybrid approach handled both
semantic queries and exact citation lookups reliably. It taught me not to
default to the 'standard' architecture without testing it against my
actual data first."

---

### 4. "Tell me about a time you failed, and what you learned"

"On the Portfolio Optimization Engine, my first version of the RL training
pipeline had subtle lookahead bias — the walk-forward evaluation windows
weren't strictly isolated from training data, which inflated my Sharpe
ratio numbers. I caught it when I tried to reproduce the results in a fresh
environment and the numbers didn't hold up. I rebuilt the evaluation
harness with strict rolling windows and re-validated against Markowitz and
Black-Litterman baselines with proper statistical significance testing
(p < 0.05) across 11 rolling windows before trusting the results again.
What I changed afterward: I now build the evaluation/validation harness
*before* trusting any headline metric, not after — a good-looking number is
the reason to dig deeper, not stop."

---

### 5. "Tell me about a time you had to learn something new quickly"

"Building LexAI required going deep on retrieval techniques I hadn't used
before in a few weeks — hybrid FAISS+BM25 fusion, HyDE query rewriting,
CrossEncoder reranking. I approached it by first building the simplest
possible pipeline end-to-end (pure FAISS retrieval), getting it fully
working, and only then adding one retrieval improvement at a time —
measuring its effect on real legal queries before adding the next
technique. That let me learn each piece in the context of an already-working
system instead of trying to implement the full state-of-the-art stack
blind and debugging five new components at once."

---

### 6. "Tell me about a time you faced a tight deadline"

*(Adapt the specific detail below to your actual internship timeline if it
differs — the shape is right: the Qclairvoyance internship ran Dec 2025-Jan
2026, a compressed window for a research-heavy deliverable.)*

"My internship at Qclairvoyance Quantum Labs ran over a compressed
one-month window, and I needed to formulate and benchmark QUBO models for
molecular docking — a 675-variable optimization problem — across four
different solvers (SCIP, CPLEX, Hercules, Qubolite) in that time. To hit
the deadline, I prioritized getting one solver pipeline fully correct and
validated first, then reused that validated formulation across the
remaining solvers rather than debugging all four in parallel — which would
have meant not fully trusting any of them by the deadline. That let me
deliver a comparative benchmark with a clear result: 83.3% structural
agreement between exact solvers, with heuristic approaches diverging —
which I could stand behind because the underlying formulation had already
been validated once, carefully, rather than four times, hastily."

---

### 7. "Tell me about a time requirements were unclear or changed midway"

*(Framed around a decision point in the Portfolio Optimization Engine
project, since building an RL trading system genuinely involves ambiguous,
self-directed scope decisions. Adjust if you have a more literal
'requirements changed' story from an internship or coursework.)*

"On the Portfolio Optimization Engine, there was no fixed spec — I had to
decide myself what 'outperforming the benchmark' actually meant, which
initially I treated as just 'a higher Sharpe ratio than equal-weight
allocation.' Partway through, I realized that framing was too weak to be
convincing — a single Sharpe ratio number over one time period can be
lucky. So I effectively redefined my own requirement mid-project: I added
classical baselines (Markowitz and Black-Litterman) as stronger comparison
points, and required statistical significance (p < 0.05) across 11 rolling
evaluation windows, not just one overall number. It meant more work
partway through, but the result was actually defensible — a 1.4 vs 0.6
Sharpe ratio backed by significance testing, not a single favorable
snapshot."

---

### 8. "How do you handle receiving critical feedback?"

"I try to treat it as information about the work, not about me — the
Portfolio Optimization Engine failure story is a good example: when my
walk-forward numbers didn't reproduce, that was essentially the harshest
feedback a project can give you, that the result was wrong. Rather than
defending the original approach, I rebuilt the evaluation methodology from
scratch and didn't trust the project again until I could reproduce the
result independently. In a team setting, I'd want that same directness —
I'd rather a mentor tell me a design is wrong early, when it's cheap to
fix, than be polite about it and let me find out after I've built on top of
a bad assumption."

---

### 9. "Tell me about a time you took ownership of something beyond your explicit responsibility"

"On the URL Shortener project, the original scope was just 'build a
working shortener,' but I pushed further — I decoupled click-analytics from
the redirect path using Celery and Redis specifically so analytics volume
couldn't degrade redirect latency, added Prometheus/Grafana monitoring,
structured JSON logging with request-ID tracing, and wrote 90+ tests
including concurrency tests under 50 simultaneous requests. Nobody asked
for observability or load testing — I added it because a shortener that's
fast in isolation but unmonitored and untested under load isn't actually
production-grade. I also went back and fixed production-only database
issues and added rate limiting and custom alias support after the core
feature was already 'done.'"

---

### 10. "Walk me through a project you're proud of"

*(Using the Section 7 project-walkthrough structure — this is the answer
you should be able to deliver fluently and extend with follow-up detail.)*

"I'd walk through LexAI. **The problem:** Indian legal research is
scattered across statutes and case law with no good conversational access.
**My approach:** I built a hybrid retrieval system — FAISS for semantic
search, BM25 for exact statute/citation matching, fused via Reciprocal Rank
Fusion, then reranked with a CrossEncoder — because pure dense retrieval
alone misses exact legal citations, something I verified by testing early.
I also used HyDE query rewriting, generating a hypothetical answer first to
improve the embedding match on ambiguous legal phrasing. **The hardest
part:** getting retrieval quality high enough on ambiguous queries — that's
what pushed me toward HyDE and the CrossEncoder reranking stage rather than
stopping at plain hybrid retrieval. **The result:** an async FastAPI
backend streaming responses via Server-Sent Events with source citations
and session-aware conversational memory, over indexes covering 18+ Indian
legal statutes and Supreme Court judgments, with a React + Tailwind
frontend rendering Markdown and interactive citation panels. **What I'd do
differently:** add a proper evaluation harness with a labeled legal QA
benchmark — right now retrieval quality is validated by manual spot-checks,
and I'd want quantitative retrieval metrics (e.g., recall@k) before calling
it fully production-ready."

**Likely follow-up: "Why FAISS + BM25 instead of just a bigger/better
embedding model?"**
"Because the failure mode I was solving for isn't semantic — it's exact
lexical matching. A statute number like 'Section 302 IPC' needs an exact
token match; embeddings alone can conflate semantically similar but legally
distinct provisions. Fusing both retrieval methods covers both failure
modes, and it's cheaper than trying to solve exact-match recall purely
through a fancier embedding model."

---

### 11. "Where do you see yourself in a few years?"

"I want to go deeper into backend and distributed systems engineering.
Right now I have decent breadth across ML infrastructure and backend
systems — FastAPI, Docker, MLOps tooling, async backends, caching and
queues. I want the next few years to be about depth: understanding how
systems like Hevo's connector infrastructure behave at real production
scale, and eventually owning larger pieces of that system with more
autonomy, rather than chasing a specific title."

---

### 12. "Do you have any questions for us?"

- "What does a typical first 90 days look like for an SDE I on the
  connectors/integrations team?"
- "How is reliability and idempotency handled today when a source sync
  fails partway through — is that the kind of problem an SDE I would work
  on early?"
- "What separates someone doing well at 6 months from someone struggling
  in this role?"

*(These deliberately connect back to Hevo's actual domain — idempotent
loads and sync reliability were covered in the System Design section's
ETL/ELT discussion — which signals you've thought about the product, not
just prepped generic interview questions.)*

---

### A note on the one gap worth fixing before the real interview

Your resume is genuinely strong on independent, self-directed project work
and one research internship, but light on a real interpersonal conflict or
teammate-disagreement story — questions 3 and 6-9 above lean on solo
project decisions because that's what's documented. If you have a real
group-project, hackathon, or internship story involving a disagreement with
another person (not just a design decision you made alone), prepare that
specifically — interviewers can often tell when a "disagreement" story is
actually just "I decided to do X instead of Y" with nobody else involved,
and a genuine interpersonal example will land noticeably better.

Also worth preparing proactively: if asked "why the shift from quantum/RL
research toward backend/data engineering," have a one-line bridge ready —
something like "the Qclairvoyance internship and my RL project both taught
me I enjoy the engineering rigor of validating and productionizing a
system as much as the research itself, which is what pulled me toward
backend/infra roles like this one" — since Hevo's SDE I role is backend-
focused and your resume leans research-heavy, addressing that gap head-on
will land better than leaving the interviewer to wonder about it.
