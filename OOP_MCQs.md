# OOP Concepts — 50 MCQs for NatWest SDE Assessment Prep

Organized by subtopic. Format: Question → 4 options → **Answer** → short explanation.

---

## Core Pillars (Q1–14)

**Q1.** The four pillars of OOP are:
A) Encapsulation, Inheritance, Polymorphism, Abstraction  B) Compilation, Execution, Debugging, Testing  C) Class, Object, Method, Variable  D) Public, Private, Protected, Default
**Answer: A**

**Q2.** Encapsulation refers to:
A) Creating multiple classes  B) Bundling data and methods together while restricting direct access to internal state  C) Inheriting from a parent class  D) Overloading methods
**Answer: B**

**Q3.** Abstraction focuses on:
A) Hiding implementation details and exposing only essential features  B) Hiding data using access modifiers  C) Creating multiple objects  D) Compiling code faster
**Answer: A** — Note the distinction from encapsulation: encapsulation is about bundling + access control; abstraction is about hiding complexity and showing only relevant behavior.

**Q4.** Which of these best demonstrates inheritance?
A) A `Car` class having an `Engine` object as a field  B) A `SportsCar` class extending a `Car` class  C) A class with private variables  D) A method with the same name but different parameters
**Answer: B**

**Q5.** Polymorphism allows:
A) A single interface to represent different underlying forms (data types) or a method to behave differently based on the object calling it  B) Only single inheritance  C) Direct memory access  D) Faster compilation
**Answer: A**

**Q6.** Method Overloading is an example of:
A) Compile-time (static) polymorphism  B) Runtime (dynamic) polymorphism  C) Inheritance  D) Encapsulation
**Answer: A**

**Q7.** Method Overriding is an example of:
A) Compile-time polymorphism  B) Runtime polymorphism  C) Encapsulation  D) Abstraction only
**Answer: B**

**Q8.** In Java, method overloading is resolved:
A) At compile time based on method signature (parameter list)  B) At runtime based on object type  C) Randomly  D) It's not allowed in Java
**Answer: A**

**Q9.** Which keyword in Java is used to prevent a class from being inherited?
A) static  B) final  C) private  D) abstract
**Answer: B**

**Q10.** Which keyword prevents a method from being overridden in a subclass?
A) static  B) final  C) const  D) sealed
**Answer: B**

**Q11.** Data hiding is primarily achieved through:
A) Public access modifiers  B) Private access modifiers combined with public getter/setter methods  C) Global variables  D) Static methods
**Answer: B**

**Q12.** Which OOP principle helps reduce code duplication by allowing a class to reuse fields/methods of another class?
A) Polymorphism  B) Inheritance  C) Abstraction  D) Encapsulation
**Answer: B**

**Q13.** An abstract class in most OOP languages:
A) Can be instantiated directly  B) Cannot be instantiated and may contain both abstract and concrete methods  C) Must have zero methods  D) Is the same as an interface
**Answer: B**

**Q14.** An interface differs from an abstract class primarily in that (in traditional OOP languages like Java pre-8):
A) An interface can have constructors  B) An interface only declares method signatures (no implementation), and a class can implement multiple interfaces  C) An interface can be instantiated  D) There's no difference
**Answer: B** — (Note: modern Java allows default/static methods with bodies in interfaces since Java 8, but the core distinction about multiple inheritance still holds.)

---

## Inheritance & Access Modifiers (Q15–26)

**Q15.** Which type of inheritance is NOT directly supported by Java classes (only via interfaces)?
A) Single inheritance  B) Multilevel inheritance  C) Multiple inheritance (of classes)  D) Hierarchical inheritance
**Answer: C** — Java avoids multiple class inheritance to prevent the "Diamond Problem."

**Q16.** The Diamond Problem refers to:
A) A memory leak issue  B) Ambiguity when a class inherits from two classes that both have a method with the same signature  C) A sorting algorithm issue  D) A database normalization issue
**Answer: B**

**Q17.** A `protected` member in Java/C++ is accessible from:
A) Only within the same class  B) The same class, subclasses, and (in Java) same package  C) Everywhere without restriction  D) Nowhere
**Answer: B**

**Q18.** A `private` member is accessible:
A) From anywhere in the program  B) Only within the class it is declared in  C) From subclasses only  D) From the same package only
**Answer: B**

**Q19.** In Java, if no access modifier is specified, the default access is:
A) Public  B) Private  C) Package-private (accessible within the same package)  D) Protected
**Answer: C**

**Q20.** Constructor chaining refers to:
A) Calling one constructor from another (via `this()` or `super()`)  B) Creating multiple objects at once  C) Overloading a method  D) Deleting a constructor
**Answer: A**

**Q21.** In Java, `super()` inside a subclass constructor:
A) Calls the subclass's own constructor  B) Calls the immediate parent class's constructor  C) Deletes the parent class  D) Is only used in interfaces
**Answer: B**

**Q22.** If a subclass does not define any constructor, Java automatically:
A) Throws a compile error  B) Provides a default constructor that implicitly calls the parent's no-arg constructor  C) Prevents object creation  D) Uses the grandparent's constructor
**Answer: B**

**Q23.** Which of the following is true about static methods and polymorphism?
A) Static methods can be overridden and support runtime polymorphism  B) Static methods are resolved at compile time and are NOT polymorphic (they can be hidden, not overridden)  C) Static methods must always be abstract  D) Static methods cannot exist in Java
**Answer: B**

**Q24.** A `final` variable in Java:
A) Can be reassigned any number of times  B) Can be assigned only once (becomes a constant after initialization)  C) Is always static  D) Cannot be used in a class
**Answer: B**

**Q25.** In multilevel inheritance, `Class C extends Class B extends Class A`. Object of C can access:
A) Only C's members  B) Members of A, B, and C (subject to access modifiers)  C) Only A's members  D) None
**Answer: B**

**Q26.** Hierarchical inheritance describes:
A) One subclass inheriting from multiple parent classes  B) Multiple subclasses inheriting from a single parent class  C) A class inheriting from itself  D) No inheritance at all
**Answer: B**

---

## Constructors, Destructors & Object Lifecycle (Q27–34)

**Q27.** A constructor in OOP is:
A) A method used to destroy objects  B) A special method automatically called when an object is created, typically used to initialize it  C) A static utility method  D) The same as a regular method with any name
**Answer: B**

**Q28.** Can a constructor be overloaded?
A) No, never  B) Yes, a class can have multiple constructors with different parameter lists  C) Only in abstract classes  D) Only if the class has no fields
**Answer: B**

**Q29.** In Java, garbage collection primarily handles:
A) Manual memory deallocation like C++'s `delete`  B) Automatic reclamation of memory for objects no longer referenced  C) Compiling the code  D) Managing the stack only
**Answer: B**

**Q30.** A destructor (or `finalize`/`dispose` pattern) is used to:
A) Create an object  B) Perform cleanup before an object is destroyed/garbage collected  C) Overload a method  D) Enforce encapsulation
**Answer: B**

**Q31.** In C++, which of the following triggers a copy constructor call?
A) Only explicit assignment (`=`) after both objects already exist  B) Passing an object by value to a function, returning an object by value, or initializing one object from another  C) Never, unless explicitly written  D) Only static objects
**Answer: B**

**Q32.** A "default constructor" is:
A) A constructor with parameters  B) A no-argument constructor, either written explicitly or auto-generated if no constructor is defined  C) The last constructor written  D) Not allowed in OOP
**Answer: B**

**Q33.** Object cloning (e.g., Java's `Cloneable`/`clone()`) is used to:
A) Delete an object  B) Create a copy of an existing object  C) Compare two objects  D) Serialize an object to disk
**Answer: B**

**Q34.** The `this` keyword in Java/C++ refers to:
A) The parent class  B) The current instance of the class  C) A static variable  D) An interface
**Answer: B**

---

## Interfaces, Abstract Classes & Design (Q35–44)

**Q35.** A class implementing an interface in Java MUST:
A) Override all abstract methods of the interface (unless the class itself is abstract)  B) Extend another class too  C) Have a constructor  D) Be declared final
**Answer: A**

**Q36.** Can an abstract class have a constructor in Java?
A) No, never  B) Yes — it's called when a subclass object is instantiated (via `super()`)  C) Only if it has no abstract methods  D) Only static constructors allowed
**Answer: B**

**Q37.** Which of these is true about interfaces in Java (pre-Java 8)?
A) They can contain instance variables with any value  B) All methods are implicitly `public abstract`, and all fields are implicitly `public static final`  C) They can have constructors  D) They support multiple inheritance of implementation
**Answer: B**

**Q38.** Composition ("has-a" relationship) is generally preferred over inheritance ("is-a") when:
A) You want tight coupling and to expose the parent's entire interface  B) You want flexibility and looser coupling by building objects from other objects rather than extending classes  C) Never — inheritance is always better  D) Composition and inheritance are identical
**Answer: B** — Known as the "favor composition over inheritance" design principle.

**Q39.** The Liskov Substitution Principle (part of SOLID) states that:
A) A subclass should be replaceable for its superclass without breaking the program's correctness  B) Every class must implement an interface  C) Classes should never inherit from abstract classes  D) Objects must always be immutable
**Answer: A**

**Q40.** The Single Responsibility Principle (SOLID) states:
A) A class should have multiple unrelated responsibilities  B) A class should have only one reason to change (one responsibility)  C) A class should implement every interface available  D) A class should never be extended
**Answer: B**

**Q41.** Which SOLID principle states "software entities should be open for extension but closed for modification"?
A) Single Responsibility  B) Open/Closed Principle  C) Liskov Substitution  D) Dependency Inversion
**Answer: B**

**Q42.** The Dependency Inversion Principle suggests:
A) High-level modules should depend on low-level modules directly  B) Both high-level and low-level modules should depend on abstractions (interfaces), not concrete implementations  C) There should be no dependencies at all  D) All classes must be static
**Answer: B**

**Q43.** A marker interface (like Java's `Serializable`) is:
A) An interface with many methods  B) An empty interface used to convey metadata/a capability to the JVM or a framework  C) The same as an abstract class  D) Not a real concept
**Answer: B**

**Q44.** Which design pattern ensures a class has only one instance and provides a global point of access to it?
A) Factory Pattern  B) Singleton Pattern  C) Observer Pattern  D) Builder Pattern
**Answer: B**

---

## Miscellaneous OOP (Q45–50)

**Q45.** Operator overloading (e.g., in C++) allows:
A) Redefining the behavior of operators like `+` for user-defined types  B) Overloading only built-in types  C) Changing the syntax of the language  D) Removing operators
**Answer: A** — (Note: Java does not support operator overloading, unlike C++.)

**Q46.** In Java, all classes implicitly inherit from:
A) `Object` class  B) `Main` class  C) No class by default  D) `Abstract` class
**Answer: A**

**Q47.** A class with all abstract methods and no implementation at all (in older Java) is functionally closest to:
A) A concrete class  B) An interface  C) A final class  D) A static class
**Answer: B**

**Q48.** Aggregation (a weaker form of "has-a" relationship than composition) means:
A) The contained object's lifecycle is tied strictly to the container  B) The contained object can exist independently of the container (e.g., a `Department` has `Employees`, but employees can exist without that specific department object)  C) It's identical to inheritance  D) It requires multiple inheritance
**Answer: B**

**Q49.** Which of the following is NOT a valid reason to use an abstract class over an interface (in Java 7 and earlier)?
A) You want to provide some default/shared implementation across subclasses  B) You want to define constructors or instance fields with state  C) You want the class to support multiple inheritance of implementation from unrelated hierarchies  D) You want partial implementation with some abstract methods left for subclasses
**Answer: C** — A class can only extend one abstract class (single inheritance), so this is exactly what abstract classes CANNOT give you — that's the interface's advantage instead.

**Q50.** Runtime polymorphism in Java is implemented internally using:
A) Compile-time method binding only  B) Dynamic method dispatch (virtual method table lookup at runtime)  C) Static binding always  D) It isn't implemented at all
**Answer: B**
