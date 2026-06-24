# Object-Oriented Programming in C++

A comprehensive guide to OOP concepts in C++ with detailed explanations, examples, and gotchas.

---

## Table of Contents
1. [Introduction to OOP](#introduction-to-oop)
2. [Classes and Objects](#classes-and-objects)
3. [Encapsulation](#encapsulation)
4. [Inheritance](#inheritance)
5. [Polymorphism](#polymorphism)
6. [Abstraction](#abstraction)
7. [Access Specifiers](#access-specifiers)
8. [Constructors and Destructors](#constructors-and-destructors)
9. [Virtual Functions and Dynamic Binding](#virtual-functions-and-dynamic-binding)
10. [Multiple Inheritance](#multiple-inheritance)
11. [Complete Practical Example](#complete-practical-example)

---

## Introduction to OOP

### What is Object-Oriented Programming?

OOP is a programming paradigm based on the concept of "objects" that contain data (attributes) and code (methods).

### Four Pillars of OOP
```
┌──────────────────────────────────────────┐
│                 OOP PILLARS              │
├──────────────────────────────────────────┤
│                                          │
│  ┌──────────────┐  ┌──────────────┐      │
│  │ Encapsulation│  │  Abstraction │      │
│  │              │  │              │      │
│  │ Hide details │  │ Expose only  │      │
│  │ Bundle data  │  │ essential    │      │
│  └──────────────┘  └──────────────┘      │
│                                          │
│  ┌──────────────┐  ┌───────────────┐     │
│  │ Inheritance  │  │ Polymorphism  │     │
│  │              │  │               │     │
│  │ Reuse code   │  │ Many forms    │     │
│  │ "IS-A"       │  │ Same interface│     │
│  └──────────────┘  └───────────────┘     │
│                                          │
└──────────────────────────────────────────┘
```

---

## Classes and Objects

### What are Classes and Objects?

**Class**: A blueprint or template for creating objects
**Object**: An instance of a class

```
┌─────────────────────────────────────────┐
│          CLASS vs OBJECT                │
├─────────────────────────────────────────┤
│                                         │
│  Class: Car                             │
│  ┌─────────────────────┐                │
│  │ Attributes:         │                │
│  │  - color            │  Blueprint     │
│  │  - speed            │                │
│  │  - model            │                │
│  │                     │                │
│  │ Methods:            │                │
│  │  - start()          │                │
│  │  - accelerate()     │                │
│  │  - brake()          │                │
│  └─────────────────────┘                │
│           │                             │
│           │ instantiate                 │
│           ▼                             │
│  ┌─────────────────────┐                │
│  │ Object: myCar       │                │
│  │  color = "red"      │  Instance      │
│  │  speed = 0          │                │
│  │  model = "Tesla"    │                │
│  └─────────────────────┘                │
│                                         │
└─────────────────────────────────────────┘
```

### Basic Class Definition

```cpp
#include <iostream>
#include <string>

class Car {
private:
    std::string brand_;
    int speed_;
    
public:
    // Constructor
    Car(std::string brand) : brand_(brand), speed_(0) {
        std::cout << "Car created: " << brand_ << "\n";
    }
    
    // Methods
    void accelerate(int increment) {
        speed_ += increment;
        std::cout << brand_ << " speed: " << speed_ << " km/h\n";
    }
    
    void brake() {
        speed_ = 0;
        std::cout << brand_ << " stopped\n";
    }
    
    // Getter
    int get_speed() const { return speed_; }
};

int main() {
    Car tesla("Tesla");      // Create object
    tesla.accelerate(50);    // Call method
    tesla.accelerate(30);
    tesla.brake();
    
    return 0;
}
```

### 💡 Hunch
- Always initialize member variables in constructor
- Use initializer lists for efficiency
- Const member functions don't modify object state

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Uninitialized members
class Bad {
    int value_;  // Not initialized!
public:
    Bad() {}  // value_ contains garbage!
};

// GOOD: Always initialize
class Good {
    int value_;
public:
    Good() : value_(0) {}  // Initialized in initializer list
};

// GOTCHA 2: Most vexing parse
Car myCar();  // Not an object! This is a function declaration!
Car myCar;    // Correct: creates an object
```

---

## Encapsulation

### What is Encapsulation?

Bundling data and methods that operate on that data within a single unit (class), and restricting direct access to some components.

```
┌────────────────────────────────────────────┐
│           ENCAPSULATION                    │
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │         BankAccount Class            │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │ Private (Hidden):              │  │  │
│  │  │  - balance_                    │  │  │
│  │  │  - account_number_             │  │  │
│  │  │  - validate_pin()              │  │  │
│  │  └────────────────────────────────┘  │  │
│  │           ▲                          │  │
│  │           │                          │  │
│  │           │                          │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │ Public (Interface):            │  │  │
│  │  │  + deposit()                   │  │  │
│  │  │  + withdraw()                  │  │  │
│  │  │  + get_balance()               │  │  │
│  │  └────────────────────────────────┘  │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  Users can only access through public      │
│  interface, not directly manipulate data   │
└────────────────────────────────────────────┘
```

### Example: Bank Account

```cpp
#include <iostream>
#include <string>

class BankAccount {
private:
    // Encapsulated data - hidden from outside
    std::string account_number_;
    double balance_;
    std::string owner_;
    
    // Private helper method
    bool validate_amount(double amount) const {
        return amount > 0 && amount <= 1000000;
    }
    
public:
    // Constructor
    BankAccount(std::string owner, std::string account_number)
        : owner_(owner)
        , account_number_(account_number)
        , balance_(0.0) {}
    
    // Public interface - controlled access
    bool deposit(double amount) {
        if (!validate_amount(amount)) {
            std::cout << "Invalid deposit amount\n";
            return false;
        }
        balance_ += amount;
        std::cout << "Deposited: $" << amount << "\n";
        return true;
    }
    
    bool withdraw(double amount) {
        if (!validate_amount(amount)) {
            std::cout << "Invalid withdrawal amount\n";
            return false;
        }
        if (amount > balance_) {
            std::cout << "Insufficient funds\n";
            return false;
        }
        balance_ -= amount;
        std::cout << "Withdrawn: $" << amount << "\n";
        return true;
    }
    
    // Getter (read-only access)
    double get_balance() const {
        return balance_;
    }
    
    std::string get_owner() const {
        return owner_;
    }
};

int main() {
    BankAccount account("John Doe", "ACC123456");
    
    account.deposit(1000);
    account.withdraw(200);
    
    std::cout << "Balance: $" << account.get_balance() << "\n";
    
    // account.balance_ = 1000000;  // ERROR! Private member
    
    return 0;
}
```

### 💡 Hunch
- Make data members private by default
- Provide public getters/setters only when needed
- Use const for methods that don't modify state
- Validate all inputs in setters

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Returning non-const reference breaks encapsulation
class Bad {
private:
    std::vector<int> data_;
public:
    std::vector<int>& get_data() { return data_; }  // BAD!
};

Bad obj;
obj.get_data().clear();  // Oops! Can modify private data!

// GOOD: Return const reference or copy
class Good {
private:
    std::vector<int> data_;
public:
    const std::vector<int>& get_data() const { return data_; }
};

// GOTCHA 2: Public data members
struct Data {
    int value;  // Public by default in struct! No validation possible
};
```

---

## Inheritance

### What is Inheritance?

A mechanism where a new class (derived/child) inherits properties and behaviors from an existing class (base/parent).

```
┌───────────────────────────────────────────────┐
│            INHERITANCE HIERARCHY              │
├───────────────────────────────────────────────┤
│                                               │
│           ┌──────────────┐                    │
│           │   Vehicle    │  Base Class        │
│           │──────────────│                    │
│           │ + speed      │                    │
│           │ + start()    │                    │
│           └──────┬───────┘                    │
│                  │                            │
│         ┌────────┴────────┐                   │
│         │                 │                   │
│    ┌────▼─────┐     ┌────▼─────┐              │
│    │   Car    │     │   Bike   │ Derived      │
│    │──────────│     │──────────│ Classes      │
│    │ + doors  │     │ + pedals │              │
│    │ + honk() │     │ + ring() │              │
│    └──────────┘     └──────────┘              │
│                                               │
│  "Car IS-A Vehicle"                           │
│  "Bike IS-A Vehicle"                          │
│                                               │
└───────────────────────────────────────────────┘
```

### Types of Inheritance

```cpp
#include <iostream>
#include <string>

// Base class
class Animal {
protected:  // Accessible in derived classes
    std::string name_;
    int age_;
    
public:
    Animal(std::string name, int age) 
        : name_(name), age_(age) {
        std::cout << "Animal constructor: " << name_ << "\n";
    }
    
    virtual ~Animal() {
        std::cout << "Animal destructor: " << name_ << "\n";
    }
    
    void eat() const {
        std::cout << name_ << " is eating\n";
    }
    
    void sleep() const {
        std::cout << name_ << " is sleeping\n";
    }
    
    // Virtual function (can be overridden)
    virtual void make_sound() const {
        std::cout << name_ << " makes a sound\n";
    }
    
    std::string get_name() const { return name_; }
};

// Derived class 1: Dog
class Dog : public Animal {
private:
    std::string breed_;
    
public:
    Dog(std::string name, int age, std::string breed)
        : Animal(name, age), breed_(breed) {
        std::cout << "Dog constructor: " << name_ << "\n";
    }
    
    ~Dog() {
        std::cout << "Dog destructor: " << name_ << "\n";
    }
    
    // Override virtual function
    void make_sound() const override {
        std::cout << name_ << " says: Woof! Woof!\n";
    }
    
    // Dog-specific method
    void fetch() const {
        std::cout << name_ << " is fetching the ball\n";
    }
    
    void show_breed() const {
        std::cout << name_ << " is a " << breed_ << "\n";
    }
};

// Derived class 2: Cat
class Cat : public Animal {
private:
    bool indoor_;
    
public:
    Cat(std::string name, int age, bool indoor)
        : Animal(name, age), indoor_(indoor) {
        std::cout << "Cat constructor: " << name_ << "\n";
    }
    
    ~Cat() {
        std::cout << "Cat destructor: " << name_ << "\n";
    }
    
    void make_sound() const override {
        std::cout << name_ << " says: Meow! Meow!\n";
    }
    
    void scratch() const {
        std::cout << name_ << " is scratching\n";
    }
};

int main() {
    std::cout << "=== Creating Animals ===\n";
    Dog buddy("Buddy", 3, "Golden Retriever");
    Cat whiskers("Whiskers", 2, true);
    
    std::cout << "\n=== Common Behaviors ===\n";
    buddy.eat();
    whiskers.eat();
    
    std::cout << "\n=== Polymorphic Behavior ===\n";
    buddy.make_sound();    // Dog's version
    whiskers.make_sound(); // Cat's version
    
    std::cout << "\n=== Specific Behaviors ===\n";
    buddy.fetch();
    buddy.show_breed();
    whiskers.scratch();
    
    std::cout << "\n=== Destruction ===\n";
    return 0;
}
```

### Inheritance Access Modes

```
┌─────────────────────────────────────────────────────────────┐
│        INHERITANCE ACCESS CONTROL                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Base Class    │  Public      │ Protected  │ Private        │
│  Member        │ Inheritance  │            │                │
│────────────────┼──────────────┼────────────┼────────────────┤
│  public        │  public      │ protected  │ private        │
│  protected     │  protected   │ protected  │ private        │
│  private       │ inaccessible │inaccessible│ inaccessible   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Most common: public inheritance (IS-A relationship)
```

### 💡 Hunch
- Use public inheritance for "IS-A" relationships
- Make destructors virtual in base classes
- Use protected for members that derived classes need
- Call base constructor explicitly in derived constructor
- Use `override` keyword for clarity and safety

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Slicing - derived object copied to base
Dog dog("Max", 5, "Bulldog");
Animal animal = dog;  // SLICED! Only Animal part copied
animal.make_sound();  // Calls Animal version, not Dog!

// GOOD: Use pointers or references
Animal* ptr = &dog;
ptr->make_sound();  // Calls Dog version (polymorphism)

// GOTCHA 2: Non-virtual destructor
class Base {
public:
    ~Base() { /* cleanup */ }  // Non-virtual!
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() { delete[] data_; }  // Might not be called!
};

Base* ptr = new Derived();
delete ptr;  // UB! Only Base destructor called, memory leak!

// GOOD: Virtual destructor
class Base {
public:
    virtual ~Base() { /* cleanup */ }
};

// GOTCHA 3: Hiding base class functions
class Base {
public:
    void func() { }
    void func(int x) { }
};

class Derived : public Base {
public:
    void func(double x) { }  // Hides ALL Base::func overloads!
};

Derived d;
// d.func();     // ERROR! Base::func() hidden
// d.func(5);    // ERROR! Base::func(int) hidden
d.func(5.0);     // OK

// GOOD: Use 'using' to bring base functions into scope
class Derived : public Base {
public:
    using Base::func;  // Bring all overloads
    void func(double x) { }
};
```

---

## Polymorphism

### What is Polymorphism?

The ability of objects of different types to be accessed through the same interface. "Many forms."

### Types of Polymorphism

```
┌──────────────────────────────────────────────┐
│         POLYMORPHISM TYPES                   │
├──────────────────────────────────────────────┤
│                                              │
│  1. COMPILE-TIME (Static)                    │
│     ┌────────────────────────────┐           │
│     │ Function Overloading       │           │
│     │ Operator Overloading       │           │
│     │ Template Polymorphism      │           │
│     └────────────────────────────┘           │
│                                              │
│  2. RUNTIME (Dynamic)                        │
│     ┌────────────────────────────┐           │
│     │ Virtual Functions          │           │
│     │ Function Overriding        │           │
│     │ Abstract Classes           │           │
│     └────────────────────────────┘           │
│                                              │
└──────────────────────────────────────────────┘
```

### Compile-Time Polymorphism

```cpp
#include <iostream>

// Function Overloading
class Calculator {
public:
    // Same name, different parameters
    int add(int a, int b) {
        return a + b;
    }
    
    double add(double a, double b) {
        return a + b;
    }
    
    int add(int a, int b, int c) {
        return a + b + c;
    }
};

// Operator Overloading
class Complex {
private:
    double real_;
    double imag_;
    
public:
    Complex(double real, double imag) : real_(real), imag_(imag) {}
    
    // Overload + operator
    Complex operator+(const Complex& other) const {
        return Complex(real_ + other.real_, imag_ + other.imag_);
    }
    
    // Overload << operator (friend function)
    friend std::ostream& operator<<(std::ostream& os, const Complex& c) {
        os << c.real_ << " + " << c.imag_ << "i";
        return os;
    }
};

int main() {
    // Function overloading
    Calculator calc;
    std::cout << calc.add(5, 3) << "\n";         // int version
    std::cout << calc.add(5.5, 3.2) << "\n";     // double version
    std::cout << calc.add(1, 2, 3) << "\n";      // three parameters
    
    // Operator overloading
    Complex c1(3, 4);
    Complex c2(1, 2);
    Complex c3 = c1 + c2;  // Uses overloaded +
    std::cout << c3 << "\n";  // Uses overloaded <<
    
    return 0;
}
```

### Runtime Polymorphism

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <cmath>   // std::sqrt used by Triangle::area()

// Base class with virtual functions
class Shape {
protected:
    std::string name_;
    
public:
    Shape(std::string name) : name_(name) {}
    
    virtual ~Shape() = default;
    
    // Pure virtual function (abstract)
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
    
    // Virtual function with implementation
    virtual void display() const {
        std::cout << "Shape: " << name_ << "\n";
        std::cout << "Area: " << area() << "\n";
        std::cout << "Perimeter: " << perimeter() << "\n";
    }
    
    std::string get_name() const { return name_; }
};

class Rectangle : public Shape {
private:
    double width_;
    double height_;
    
public:
    Rectangle(double width, double height)
        : Shape("Rectangle"), width_(width), height_(height) {}
    
    double area() const override {
        return width_ * height_;
    }
    
    double perimeter() const override {
        return 2 * (width_ + height_);
    }
};

class Circle : public Shape {
private:
    double radius_;
    static constexpr double PI = 3.14159265359;
    
public:
    Circle(double radius)
        : Shape("Circle"), radius_(radius) {}
    
    double area() const override {
        return PI * radius_ * radius_;
    }
    
    double perimeter() const override {
        return 2 * PI * radius_;
    }
};

class Triangle : public Shape {
private:
    double a_, b_, c_;  // sides
    
public:
    Triangle(double a, double b, double c)
        : Shape("Triangle"), a_(a), b_(b), c_(c) {}
    
    double area() const override {
        // Heron's formula
        double s = (a_ + b_ + c_) / 2;
        return std::sqrt(s * (s - a_) * (s - b_) * (s - c_));
    }
    
    double perimeter() const override {
        return a_ + b_ + c_;
    }
};

int main() {
    // Polymorphic container - holds different shapes
    std::vector<std::unique_ptr<Shape>> shapes;
    
    shapes.push_back(std::make_unique<Rectangle>(5, 3));
    shapes.push_back(std::make_unique<Circle>(4));
    shapes.push_back(std::make_unique<Triangle>(3, 4, 5));
    
    std::cout << "=== Shape Information ===\n";
    
    // Polymorphic behavior - same interface, different implementations
    for (const auto& shape : shapes) {
        shape->display();  // Calls appropriate version
        std::cout << "\n";
    }
    
    // Calculate total area
    double total_area = 0;
    for (const auto& shape : shapes) {
        total_area += shape->area();  // Polymorphic call
    }
    std::cout << "Total area: " << total_area << "\n";
    
    return 0;
}
```

### Virtual Function Table (VTable)

```
┌─────────────────────────────────────────────────┐
│          VIRTUAL FUNCTION MECHANISM             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Object in Memory:                              │
│  ┌──────────────────┐                           │
│  │ Circle Object    │                           │
│  ├──────────────────┤                           │
│  │ vptr  ───────────┼─┐  Hidden pointer         │
│  ├──────────────────┤ │                         │
│  │ name_ = "Circle" │ │                         │
│  │ radius_ = 4.0    │ │                         │
│  └──────────────────┘ │                         │
│                       │                         │
│                       └──> VTable (Circle)      │
│                            ┌──────────────────┐ │
│                            │ area()       ────┼─┼─> Circle::area()
│                            │ perimeter()  ────┼─┼─> Circle::perimeter()
│                            │ display()    ────┼─┼─> Shape::display()
│                            └──────────────────┘ │
│                                                 │
│  When calling: ptr->area()                      │
│  1. Follow vptr to VTable                       │
│  2. Lookup area() in VTable                     │
│  3. Call the function pointed to                │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 💡 Hunch
- Use `virtual` for functions that should have different behavior in derived classes
- Use `override` keyword to catch mistakes
- Use `final` to prevent further overriding
- Pure virtual functions make a class abstract
- Virtual functions have small runtime cost (vtable lookup)

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Virtual functions don't work in constructors/destructors
class Base {
public:
    Base() {
        init();  // Calls Base::init(), NOT Derived::init()!
    }
    virtual void init() {
        std::cout << "Base init\n";
    }
};

class Derived : public Base {
public:
    void init() override {
        std::cout << "Derived init\n";  // Never called from Base constructor!
    }
};

// GOTCHA 2: Forgetting to make destructor virtual
class Base {
public:
    ~Base() { }  // NOT virtual!
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() { delete[] data_; }  // May not be called!
};

Base* ptr = new Derived();
delete ptr;  // Only Base destructor called - memory leak!

// GOTCHA 3: Default parameters don't work with virtual functions
class Base {
public:
    virtual void func(int x = 10) {
        std::cout << "Base: " << x << "\n";
    }
};

class Derived : public Base {
public:
    void func(int x = 20) override {
        std::cout << "Derived: " << x << "\n";
    }
};

Derived d;
Base* ptr = &d;
ptr->func();  // Prints "Derived: 10" - uses Base's default value!

// GOTCHA 4: Calling virtual function from initializer list
class Base {
    int value_;
public:
    Base() : value_(get_value()) {}  // BAD! Virtual function in initializer
    virtual int get_value() { return 0; }
};
```

---

## Abstraction

### What is Abstraction?

Hiding complex implementation details and showing only essential features. Using abstract classes and interfaces.

```
┌─────────────────────────────────────────────┐
│           ABSTRACTION LEVELS                │
├─────────────────────────────────────────────┤
│                                             │
│  HIGH LEVEL (User sees)                     │
│  ┌─────────────────────────┐                │
│  │  car.start()            │  Simple        │
│  │  car.accelerate(50)     │  Interface     │
│  │  car.brake()            │                │
│  └─────────────────────────┘                │
│            │                                │
│            │ Abstraction                    │
│            │                                │
│  LOW LEVEL (Hidden complexity)              │
│  ┌─────────────────────────┐                │
│  │ - Check fuel            │                │
│  │ - Engage ignition       │  Hidden        │
│  │ - Start engine          │  Details       │
│  │ - Check oil pressure    │                │
│  │ - Warm up engine        │                │
│  └─────────────────────────┘                │
│                                             │
└─────────────────────────────────────────────┘
```

### Abstract Classes and Interfaces

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <vector>

// Abstract class (has pure virtual functions)
class Database {
public:
    virtual ~Database() = default;
    
    // Pure virtual functions - must be implemented by derived classes
    virtual bool connect(const std::string& connection_string) = 0;
    virtual void disconnect() = 0;
    virtual bool execute_query(const std::string& query) = 0;
    virtual std::string fetch_result() = 0;
    
    // Concrete function - shared implementation
    void log(const std::string& message) const {
        std::cout << "[LOG] " << message << "\n";
    }
};

// Concrete implementation 1: MySQL
class MySQLDatabase : public Database {
private:
    bool connected_;
    std::string connection_string_;
    
public:
    MySQLDatabase() : connected_(false) {}
    
    bool connect(const std::string& connection_string) override {
        connection_string_ = connection_string;
        connected_ = true;
        log("MySQL: Connected to " + connection_string);
        return true;
    }
    
    void disconnect() override {
        if (connected_) {
            log("MySQL: Disconnected");
            connected_ = false;
        }
    }
    
    bool execute_query(const std::string& query) override {
        if (!connected_) {
            log("MySQL: Not connected!");
            return false;
        }
        log("MySQL: Executing query: " + query);
        return true;
    }
    
    std::string fetch_result() override {
        return "MySQL Result Set";
    }
};

// Concrete implementation 2: PostgreSQL
class PostgreSQLDatabase : public Database {
private:
    bool connected_;
    
public:
    PostgreSQLDatabase() : connected_(false) {}
    
    bool connect(const std::string& connection_string) override {
        connected_ = true;
        log("PostgreSQL: Connected to " + connection_string);
        return true;
    }
    
    void disconnect() override {
        if (connected_) {
            log("PostgreSQL: Disconnected");
            connected_ = false;
        }
    }
    
    bool execute_query(const std::string& query) override {
        if (!connected_) {
            log("PostgreSQL: Not connected!");
            return false;
        }
        log("PostgreSQL: Executing query: " + query);
        return true;
    }
    
    std::string fetch_result() override {
        return "PostgreSQL Result Set";
    }
};

// Interface (pure abstract class - only pure virtual functions)
class Serializable {
public:
    virtual ~Serializable() = default;
    virtual std::string serialize() const = 0;
    virtual void deserialize(const std::string& data) = 0;
};

// Class implementing multiple interfaces
class User : public Serializable {
private:
    int id_;
    std::string name_;
    std::string email_;
    
public:
    User(int id, std::string name, std::string email)
        : id_(id), name_(std::move(name)), email_(std::move(email)) {}
    
    std::string serialize() const override {
        return std::to_string(id_) + "," + name_ + "," + email_;
    }
    
    void deserialize(const std::string& data) override {
        // Simple parsing (in real code, use proper parser)
        size_t pos1 = data.find(',');
        size_t pos2 = data.find(',', pos1 + 1);
        
        id_ = std::stoi(data.substr(0, pos1));
        name_ = data.substr(pos1 + 1, pos2 - pos1 - 1);
        email_ = data.substr(pos2 + 1);
    }
    
    void display() const {
        std::cout << "User[" << id_ << "]: " << name_ << " <" << email_ << ">\n";
    }
};

// High-level application using abstraction
class Application {
private:
    std::unique_ptr<Database> db_;
    
public:
    // Dependency injection - works with any Database implementation
    Application(std::unique_ptr<Database> db) : db_(std::move(db)) {}
    
    void run() {
        // Use database through abstract interface
        db_->connect("server=localhost;database=test");
        db_->execute_query("SELECT * FROM users");
        std::cout << "Result: " << db_->fetch_result() << "\n";
        db_->disconnect();
    }
};

int main() {
    std::cout << "=== Using MySQL ===\n";
    {
        Application app1(std::make_unique<MySQLDatabase>());
        app1.run();
    }
    
    std::cout << "\n=== Using PostgreSQL ===\n";
    {
        Application app2(std::make_unique<PostgreSQLDatabase>());
        app2.run();
    }
    
    std::cout << "\n=== Serialization ===\n";
    User user(1, "Alice", "alice@example.com");
    std::string data = user.serialize();
    std::cout << "Serialized: " << data << "\n";
    
    User user2(0, "", "");
    user2.deserialize(data);
    user2.display();
    
    return 0;
}
```

### 💡 Hunch
- Use abstract classes to define interfaces
- Pure virtual functions (= 0) force derived classes to implement
- Abstract classes can't be instantiated
- Use abstraction for dependency injection
- Prefer composition over inheritance when possible

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Can't instantiate abstract class
class Abstract {
public:
    virtual void func() = 0;
};

// Abstract a;  // ERROR! Can't create instance

// GOTCHA 2: Forgot to implement pure virtual function
class Derived : public Abstract {
    // Oops! Forgot to implement func()
};

// Derived d;  // ERROR! Still abstract!

// GOTCHA 3: Deleting through base pointer without virtual destructor
class Base {
public:
    virtual void func() = 0;
    ~Base() { }  // NOT virtual!
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() { delete[] data_; }
    void func() override {}
};

Base* ptr = new Derived();
delete ptr;  // Memory leak! Derived destructor not called
```

---

## Access Specifiers

### Understanding Access Control

```
┌─────────────────────────────────────────────────┐
│          ACCESS SPECIFIERS                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐      │
│  │ private  │  │ protected │  │  public  │      │
│  └──────────┘  └───────────┘  └──────────┘      │
│       │             │              │            │
│       │             │              │            │
│  Only class    Class +         Everyone         │
│    itself      derived         can access       │
│               classes                           │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │          Class MyClass                   │   │
│  ├──────────────────────────────────────────┤   │
│  │ private:                                 │   │
│  │   int secret_;      ◄─── Only MyClass    │   │
│  │                                          │   │  
│  │ protected:                               │   │
│  │   int internal_;    ◄─── MyClass +       │   │
│  │                          derived         │   │
│  │ public:                                  │   │
│  │   int data_;        ◄─── Everyone        │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Example with All Access Levels

```cpp
#include <iostream>

class Base {
private:
    int private_data_;  // Only Base can access
    
protected:
    int protected_data_;  // Base and derived classes
    
public:
    int public_data_;  // Everyone can access
    
    Base() : private_data_(1), protected_data_(2), public_data_(3) {}
    
    void demonstrate_access() {
        std::cout << "Inside Base:\n";
        std::cout << "  private: " << private_data_ << "\n";     // OK
        std::cout << "  protected: " << protected_data_ << "\n"; // OK
        std::cout << "  public: " << public_data_ << "\n";       // OK
    }
};

class Derived : public Base {
public:
    void demonstrate_access() {
        std::cout << "Inside Derived:\n";
        // std::cout << private_data_;  // ERROR! Can't access private
        std::cout << "  protected: " << protected_data_ << "\n"; // OK
        std::cout << "  public: " << public_data_ << "\n";       // OK
    }
};

int main() {
    Base b;
    Derived d;
    
    std::cout << "From main():\n";
    // std::cout << b.private_data_;    // ERROR! Can't access
    // std::cout << b.protected_data_;  // ERROR! Can't access
    std::cout << "  public: " << b.public_data_ << "\n";  // OK
    
    b.demonstrate_access();
    d.demonstrate_access();
    
    return 0;
}
```

### 💡 Hunch
- Default for `class` is private
- Default for `struct` is public
- Use private for implementation details
- Use protected for data derived classes need
- Use public for the interface
- Prefer private data with public accessors

### ⚠️ Gotchas
```cpp
// GOTCHA 1: struct vs class defaults
struct MyStruct {
    int x;  // Public by default!
};

class MyClass {
    int x;  // Private by default!
};

// GOTCHA 2: Friend functions break encapsulation
class Secret {
private:
    int data_;
    
public:
    friend void hack(Secret& s);  // Friend can access private!
};

void hack(Secret& s) {
    s.data_ = 0;  // Can access private member!
}
```

---

## Constructors and Destructors

The example below manages a raw `char*` by hand to *demonstrate* the copy and
move special members. In real code you would store a `std::string` and let the
compiler generate these (the **Rule of Zero**). When you *do* manage a resource
directly, you must obey the **Rule of Five** — see
[Advanced Features → Rule of Five / Rule of Zero](12_advanced_features.md) for
the full treatment, and [Exceptions](17_exceptions.md) for why destructors
should be `noexcept`.

### Types of Constructors

```cpp
#include <iostream>
#include <string>
#include <cstring>

class String {
private:
    char* data_;
    size_t size_;
    
public:
    // 1. Default Constructor
    String() : data_(nullptr), size_(0) {
        std::cout << "Default constructor\n";
    }
    
    // 2. Parameterized Constructor
    String(const char* str) {
        std::cout << "Parameterized constructor\n";
        size_ = std::strlen(str);
        data_ = new char[size_ + 1];
        std::strcpy(data_, str);
    }
    
    // 3. Copy Constructor
    String(const String& other) {
        std::cout << "Copy constructor\n";
        size_ = other.size_;
        data_ = new char[size_ + 1];
        std::strcpy(data_, other.data_);
    }
    
    // 4. Move Constructor (C++11)
    String(String&& other) noexcept {
        std::cout << "Move constructor\n";
        data_ = other.data_;
        size_ = other.size_;
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // 5. Copy Assignment Operator
    String& operator=(const String& other) {
        std::cout << "Copy assignment\n";
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new char[size_ + 1];
            std::strcpy(data_, other.data_);
        }
        return *this;
    }
    
    // 6. Move Assignment Operator (C++11)
    String& operator=(String&& other) noexcept {
        std::cout << "Move assignment\n";
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
    
    // Destructor
    ~String() {
        std::cout << "Destructor\n";
        delete[] data_;
    }
    
    const char* c_str() const { return data_ ? data_ : ""; }
};

int main() {
    std::cout << "1. Default:\n";
    String s1;
    
    std::cout << "\n2. Parameterized:\n";
    String s2("Hello");
    
    std::cout << "\n3. Copy:\n";
    String s3 = s2;
    
    std::cout << "\n4. Move:\n";
    String s4 = std::move(s2);
    
    std::cout << "\n5. Copy assignment:\n";
    s1 = s3;
    
    std::cout << "\n6. Move assignment:\n";
    s1 = std::move(s4);
    
    std::cout << "\n7. Destruction:\n";
    return 0;
}
```

### Constructor Initialization List

```cpp
class Example {
private:
    const int constant_;
    int& reference_;
    int value_;
    
public:
    // MUST use initializer list for const and references
    Example(int val, int& ref) 
        : constant_(42)        // const must be initialized here
        , reference_(ref)       // reference must be initialized here
        , value_(val)          // more efficient than assignment
    {
        // value_ = val;  // This is assignment, not initialization!
    }
};
```

### Delegating Constructors (C++11)

```cpp
class Point {
private:
    int x_, y_;
    
public:
    // Main constructor
    Point(int x, int y) : x_(x), y_(y) {
        std::cout << "Point(" << x_ << ", " << y_ << ")\n";
    }
    
    // Delegate to main constructor
    Point() : Point(0, 0) {
        std::cout << "Default point created\n";
    }
    
    Point(int x) : Point(x, x) {
        std::cout << "Square point created\n";
    }
};
```

### 💡 Hunch
- Always initialize members in initializer list
- Order in initializer list doesn't matter (initialization follows declaration order)
- Use `= default` for compiler-generated versions
- Use `= delete` to prevent copying/moving
- Make constructors `explicit` to prevent implicit conversions
- Make move operations `noexcept`

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Initialization order follows declaration, not initializer list
class Bad {
    int y_;
    int x_;
public:
    Bad(int val) : x_(val), y_(x_ * 2) {}  // y_ initialized first (garbage * 2)!
    // Declaration order matters: y_ is declared before x_!
};

// GOTCHA 2: Most vexing parse
class MyClass {
public:
    MyClass(int x) {}
};

MyClass obj(5);     // OK: creates object
MyClass obj2();     // NOT an object! Function declaration!
MyClass obj3{};     // OK: uniform initialization

// GOTCHA 3: Implicit conversions
class String {
public:
    String(const char* s) {}  // Not explicit!
};

void func(String s) {}

func("hello");  // Implicitly converts const char* to String

// Better:
class String {
public:
    explicit String(const char* s) {}
};

// func("hello");  // ERROR! Must explicitly convert

// GOTCHA 4: Forgot to implement virtual destructor
class Base {
public:
    Base() {}
    ~Base() {}  // NOT virtual!
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() { delete[] data_; }
};

Base* ptr = new Derived();
delete ptr;  // Memory leak! Only Base destructor called!

// GOTCHA 5: Copy-and-swap idiom needed for exception safety
class Resource {
    int* data_;
public:
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete[] data_;  // What if new throws?
            data_ = new int[100];
            // ... copy data
        }
        return *this;
    }
};

// Better: copy-and-swap
class Resource {
    int* data_;
public:
    Resource& operator=(Resource other) {  // Pass by value (copy)
        swap(*this, other);  // noexcept swap
        return *this;        // old data destroyed when 'other' goes out of scope
    }
    
    friend void swap(Resource& a, Resource& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
    }
};
```

---

## Virtual Functions and Dynamic Binding

### How Virtual Functions Work

```
┌─────────────────────────────────────────────────────┐
│     DYNAMIC DISPATCH WITH VIRTUAL FUNCTIONS         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Code:                                              │
│  ─────                                              │
│  Shape* ptr = new Circle(5);                        │
│  ptr->area();  // Which area()?                     │
│                                                     │
│  Without virtual (static binding):                  │
│  ┌──────────────────────────────┐                   │
│  │ Compile time decision        │                   │
│  │ Calls Shape::area()          │ ◄── Wrong!        │
│  └──────────────────────────────┘                   │
│                                                     │
│  With virtual (dynamic binding):                    │
│  ┌────────────────────────────────┐                 │
│  │ Runtime decision through VTable│                 │
│  │ Calls Circle::area()           │ ◄── Correct!    │
│  └────────────────────────────────┘                 │
│                                                     │
│  Memory Layout:                                     │
│  ┌─────────────────┐                                │
│  │ Circle object   │                                │
│  ├─────────────────┤                                │
│  │ vptr  ──────────┼──┐                             │
│  ├─────────────────┤  │                             │
│  │ radius = 5.0    │  │                             │
│  └─────────────────┘  │                             │
│                       │                             │
│                       └──> VTable                   │
│                            ┌────────────────┐       │
│                            │ Circle::area   │       │
│                            │ Circle::perim  │       │
│                            │ Shape::display │       │
│                            └────────────────┘       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Override, Final, and Virtual

```cpp
#include <iostream>

class Base {
public:
    virtual void func1() {
        std::cout << "Base::func1\n";
    }
    
    virtual void func2() final {  // Can't be overridden
        std::cout << "Base::func2 (final)\n";
    }
    
    virtual void func3() {
        std::cout << "Base::func3\n";
    }
};

class Derived : public Base {
public:
    void func1() override {  // Explicitly overriding
        std::cout << "Derived::func1\n";
    }
    
    // void func2() override {}  // ERROR! Base::func2 is final
    
    void func3() override final {  // Override and make final
        std::cout << "Derived::func3 (final)\n";
    }
    
    // void func4() override {}  // ERROR! Nothing to override
};

class MoreDerived : public Derived {
public:
    // void func3() override {}  // ERROR! Derived::func3 is final
};

// Final class - can't be inherited from
class Final final {
public:
    void func() {
        std::cout << "Final class\n";
    }
};

// class CannotDeriveFromFinal : public Final {};  // ERROR!

int main() {
    Base* ptr = new Derived();
    ptr->func1();  // Calls Derived::func1
    ptr->func2();  // Calls Base::func2
    ptr->func3();  // Calls Derived::func3
    delete ptr;
    
    return 0;
}
```

### Pure Virtual and Abstract Classes

```cpp
#include <iostream>
#include <vector>
#include <memory>

// Abstract class - cannot be instantiated
class Vehicle {
public:
    virtual ~Vehicle() = default;
    
    // Pure virtual functions - must be overridden
    virtual void start() = 0;
    virtual void stop() = 0;
    virtual double get_speed() const = 0;
    
    // Non-pure virtual - can be overridden
    virtual void display() const {
        std::cout << "Vehicle speed: " << get_speed() << " km/h\n";
    }
};

class Car : public Vehicle {
private:
    double speed_;
    
public:
    Car() : speed_(0) {}
    
    void start() override {
        std::cout << "Car: Starting engine\n";
        speed_ = 0;
    }
    
    void stop() override {
        std::cout << "Car: Stopping\n";
        speed_ = 0;
    }
    
    double get_speed() const override {
        return speed_;
    }
    
    void accelerate(double amount) {
        speed_ += amount;
    }
};

class Bicycle : public Vehicle {
private:
    double speed_;
    
public:
    Bicycle() : speed_(0) {}
    
    void start() override {
        std::cout << "Bicycle: Start pedaling\n";
        speed_ = 0;
    }
    
    void stop() override {
        std::cout << "Bicycle: Stop pedaling\n";
        speed_ = 0;
    }
    
    double get_speed() const override {
        return speed_;
    }
    
    void pedal(double effort) {
        speed_ = effort * 10;
    }
};

int main() {
    // Vehicle v;  // ERROR! Can't instantiate abstract class
    
    std::vector<std::unique_ptr<Vehicle>> vehicles;
    
    auto car = std::make_unique<Car>();
    car->start();
    car->accelerate(60);
    
    auto bike = std::make_unique<Bicycle>();
    bike->start();
    bike->pedal(2);
    
    vehicles.push_back(std::move(car));
    vehicles.push_back(std::move(bike));
    
    std::cout << "\nVehicle speeds:\n";
    for (const auto& v : vehicles) {
        v->display();  // Polymorphic call
    }
    
    return 0;
}
```

---

## Multiple Inheritance

### Diamond Problem

```
┌───────────────────────────────────────────────┐
│           THE DIAMOND PROBLEM                 │
├───────────────────────────────────────────────┤
│                                               │
│            ┌──────────┐                       │
│            │  Animal  │                       │
│            │ - age    │                       │
│            └────┬─────┘                       │
│                 │                             │
│         ┌───────┴───────┐                     │
│         │               │                     │
│    ┌────▼────┐      ┌───▼─────┐               │
│    │  Mammal │      │  Bird   │               │
│    └────┬────┘      └───┬─────┘               │
│         │               │                     │
│         └──────┬────────┘                     │
│                │                              │
│           ┌────▼────┐                         │
│           │   Bat   │                         │
│           └─────────┘                         │
│                                               │
│  Problem: Bat has TWO copies of Animal::age!  │
│                                               │
│  Solution: Virtual inheritance                │
│                                               │
└───────────────────────────────────────────────┘
```

### Multiple Inheritance Example

```cpp
#include <iostream>
#include <string>

// Without virtual inheritance - Diamond Problem
namespace BadExample {
    class Animal {
    protected:
        int age_;
    public:
        Animal(int age) : age_(age) {
            std::cout << "Animal constructor (age=" << age << ")\n";
        }
        int get_age() const { return age_; }
    };
    
    class Mammal : public Animal {
    public:
        Mammal(int age) : Animal(age) {
            std::cout << "Mammal constructor\n";
        }
    };
    
    class Bird : public Animal {
    public:
        Bird(int age) : Animal(age) {
            std::cout << "Bird constructor\n";
        }
    };
    
    class Bat : public Mammal, public Bird {
    public:
        Bat(int age) : Mammal(age), Bird(age) {
            std::cout << "Bat constructor\n";
        }
        
        // ERROR: Ambiguous!
        // int age = get_age();  // Which get_age()? Mammal's or Bird's?
    };
}

// With virtual inheritance - Solves Diamond Problem
namespace GoodExample {
    class Animal {
    protected:
        int age_;
    public:
        Animal(int age) : age_(age) {
            std::cout << "Animal constructor (age=" << age << ")\n";
        }
        int get_age() const { return age_; }
    };
    
    // Virtual inheritance
    class Mammal : virtual public Animal {
    public:
        Mammal(int age) : Animal(age) {
            std::cout << "Mammal constructor\n";
        }
    };
    
    class Bird : virtual public Animal {
    public:
        Bird(int age) : Animal(age) {
            std::cout << "Bird constructor\n";
        }
    };
    
    class Bat : public Mammal, public Bird {
    public:
        // Must call Animal constructor directly
        Bat(int age) : Animal(age), Mammal(age), Bird(age) {
            std::cout << "Bat constructor\n";
        }
        
        // No ambiguity - only ONE Animal subobject
        int age() const { return get_age(); }
    };
}

// Multiple interfaces (no diamond problem)
class Printable {
public:
    virtual ~Printable() = default;
    virtual void print() const = 0;
};

class Serializable {
public:
    virtual ~Serializable() = default;
    virtual std::string serialize() const = 0;
};

class Document : public Printable, public Serializable {
private:
    std::string content_;
    
public:
    Document(std::string content) : content_(std::move(content)) {}
    
    void print() const override {
        std::cout << "Printing: " << content_ << "\n";
    }
    
    std::string serialize() const override {
        return "DOC:" + content_;
    }
};

int main() {
    std::cout << "=== Bad Example (Diamond Problem) ===\n";
    // BadExample::Bat bat1(5);
    // std::cout << "Age: " << bat1.get_age() << "\n";  // Ambiguous!
    
    std::cout << "\n=== Good Example (Virtual Inheritance) ===\n";
    GoodExample::Bat bat2(5);
    std::cout << "Age: " << bat2.age() << "\n";  // No ambiguity!
    
    std::cout << "\n=== Multiple Interfaces ===\n";
    Document doc("Hello, World!");
    doc.print();
    std::cout << "Serialized: " << doc.serialize() << "\n";
    
    return 0;
}
```

### 💡 Hunch
- Avoid multiple implementation inheritance when possible
- Multiple interface inheritance is usually fine
- Use virtual inheritance to solve diamond problem
- Virtual inheritance has performance cost
- Prefer composition over multiple inheritance

### ⚠️ Gotchas
```cpp
// GOTCHA 1: Ambiguity with multiple inheritance
class A {
public:
    void func() {}
};

class B {
public:
    void func() {}
};

class C : public A, public B {
};

C c;
// c.func();  // ERROR! Ambiguous - A::func or B::func?
c.A::func();  // OK: explicitly specify

// GOTCHA 2: Constructor calling order with virtual inheritance
class Base {
public:
    Base(int x) { std::cout << "Base(" << x << ")\n"; }
};

class A : virtual public Base {
public:
    A() : Base(1) {}
};

class B : virtual public Base {
public:
    B() : Base(2) {}
};

class C : public A, public B {
public:
    C() : Base(3), A(), B() {}  // Base(3) is called, not Base(1) or Base(2)!
};
// Most derived class calls virtual base constructor

// GOTCHA 3: Downcasting ambiguity
class Base1 {
public:
    virtual ~Base1() = default;
    int x;
};

class Base2 {
public:
    virtual ~Base2() = default;
    int y;
};

class Derived : public Base1, public Base2 {
};

Derived d;
Base1* p1 = &d;  // OK
Base2* p2 = &d;  // OK, but different address!

// Derived* pd = static_cast<Derived*>(p2);  // Potentially wrong!
Derived* pd = dynamic_cast<Derived*>(p2);    // Correct - adjusts pointer
```

---

## Complete Practical Example

Here's a comprehensive banking system demonstrating all OOP concepts:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <stdexcept>
#include <iomanip>

// 1. ABSTRACTION - Abstract base class
class Account {
protected:
    std::string account_number_;
    std::string owner_;
    double balance_;
    
    // Protected constructor - can only be called by derived classes
    Account(std::string account_number, std::string owner, double initial_balance)
        : account_number_(std::move(account_number))
        , owner_(std::move(owner))
        , balance_(initial_balance) {
        if (balance_ < 0) {
            throw std::invalid_argument("Initial balance cannot be negative");
        }
    }
    
public:
    virtual ~Account() = default;
    
    // Pure virtual functions - ABSTRACTION
    virtual bool withdraw(double amount) = 0;
    virtual bool deposit(double amount) = 0;
    virtual std::string get_account_type() const = 0;
    virtual double get_monthly_fee() const = 0;
    
    // Concrete methods - shared by all accounts
    double get_balance() const { return balance_; }
    std::string get_account_number() const { return account_number_; }
    std::string get_owner() const { return owner_; }
    
    // Virtual method with default implementation
    virtual void display() const {
        std::cout << "Account: " << account_number_ << "\n"
                  << "Owner: " << owner_ << "\n"
                  << "Type: " << get_account_type() << "\n"
                  << "Balance: $" << std::fixed << std::setprecision(2) 
                  << balance_ << "\n";
    }
};

// 2. ENCAPSULATION - Checking Account
class CheckingAccount : public Account {
private:
    double overdraft_limit_;
    static constexpr double MONTHLY_FEE = 10.0;
    
    // Private helper method - ENCAPSULATION
    bool is_within_limit(double amount) const {
        return (balance_ - amount) >= -overdraft_limit_;
    }
    
public:
    CheckingAccount(std::string account_number, std::string owner, 
                   double initial_balance, double overdraft_limit = 500.0)
        : Account(std::move(account_number), std::move(owner), initial_balance)
        , overdraft_limit_(overdraft_limit) {}
    
    // POLYMORPHISM - Override virtual methods
    bool withdraw(double amount) override {
        if (amount <= 0) {
            std::cout << "Invalid withdrawal amount\n";
            return false;
        }
        
        if (!is_within_limit(amount)) {
            std::cout << "Withdrawal denied: exceeds overdraft limit\n";
            return false;
        }
        
        balance_ -= amount;
        std::cout << "Withdrew $" << amount << " from checking account\n";
        return true;
    }
    
    bool deposit(double amount) override {
        if (amount <= 0) {
            std::cout << "Invalid deposit amount\n";
            return false;
        }
        
        balance_ += amount;
        std::cout << "Deposited $" << amount << " to checking account\n";
        return true;
    }
    
    std::string get_account_type() const override {
        return "Checking";
    }
    
    double get_monthly_fee() const override {
        return MONTHLY_FEE;
    }
    
    double get_overdraft_limit() const { return overdraft_limit_; }
};

// 3. INHERITANCE - Savings Account
class SavingsAccount : public Account {
private:
    double interest_rate_;
    int withdrawals_this_month_;
    static constexpr int FREE_WITHDRAWALS = 3;
    static constexpr double WITHDRAWAL_FEE = 2.50;
    static constexpr double MONTHLY_FEE = 5.0;
    
public:
    SavingsAccount(std::string account_number, std::string owner,
                  double initial_balance, double interest_rate = 0.02)
        : Account(std::move(account_number), std::move(owner), initial_balance)
        , interest_rate_(interest_rate)
        , withdrawals_this_month_(0) {}
    
    bool withdraw(double amount) override {
        if (amount <= 0) {
            std::cout << "Invalid withdrawal amount\n";
            return false;
        }
        
        double total = amount;
        if (withdrawals_this_month_ >= FREE_WITHDRAWALS) {
            total += WITHDRAWAL_FEE;
            std::cout << "Withdrawal fee: $" << WITHDRAWAL_FEE << "\n";
        }
        
        if (total > balance_) {
            std::cout << "Withdrawal denied: insufficient funds\n";
            return false;
        }
        
        balance_ -= total;
        withdrawals_this_month_++;
        std::cout << "Withdrew $" << amount << " from savings account\n";
        return true;
    }
    
    bool deposit(double amount) override {
        if (amount <= 0) {
            std::cout << "Invalid deposit amount\n";
            return false;
        }
        
        balance_ += amount;
        std::cout << "Deposited $" << amount << " to savings account\n";
        return true;
    }
    
    std::string get_account_type() const override {
        return "Savings";
    }
    
    double get_monthly_fee() const override {
        return MONTHLY_FEE;
    }
    
    void apply_interest() {
        double interest = balance_ * interest_rate_;
        balance_ += interest;
        std::cout << "Applied interest: $" << interest << "\n";
    }
    
    void reset_monthly_withdrawals() {
        withdrawals_this_month_ = 0;
    }
};

// 4. MULTIPLE INTERFACES
class Transferable {
public:
    virtual ~Transferable() = default;
    virtual bool transfer_to(Account& destination, double amount) = 0;
};

class Reportable {
public:
    virtual ~Reportable() = default;
    virtual void generate_statement() const = 0;
};

// 5. MULTIPLE INHERITANCE - Premium Account
class PremiumAccount : public CheckingAccount, public Transferable, public Reportable {
private:
    std::vector<std::string> transaction_history_;
    double cashback_rate_;
    
    void log_transaction(const std::string& transaction) {
        transaction_history_.push_back(transaction);
    }
    
public:
    PremiumAccount(std::string account_number, std::string owner,
                  double initial_balance, double cashback_rate = 0.01)
        : CheckingAccount(std::move(account_number), std::move(owner), 
                         initial_balance, 1000.0)  // Higher overdraft
        , cashback_rate_(cashback_rate) {}
    
    // Override to add cashback
    bool deposit(double amount) override {
        bool success = CheckingAccount::deposit(amount);
        if (success) {
            double cashback = amount * cashback_rate_;
            balance_ += cashback;
            std::cout << "Cashback applied: $" << cashback << "\n";
            log_transaction("Deposit: $" + std::to_string(amount) + 
                          " + Cashback: $" + std::to_string(cashback));
        }
        return success;
    }
    
    bool withdraw(double amount) override {
        bool success = CheckingAccount::withdraw(amount);
        if (success) {
            log_transaction("Withdrawal: $" + std::to_string(amount));
        }
        return success;
    }
    
    // Implement Transferable interface
    bool transfer_to(Account& destination, double amount) override {
        if (withdraw(amount)) {
            if (destination.deposit(amount)) {
                log_transaction("Transfer: $" + std::to_string(amount) + 
                              " to " + destination.get_account_number());
                std::cout << "Transfer successful\n";
                return true;
            } else {
                // Rollback withdrawal
                deposit(amount);
                log_transaction("Transfer failed (rollback)");
            }
        }
        std::cout << "Transfer failed\n";
        return false;
    }
    
    // Implement Reportable interface
    void generate_statement() const override {
        std::cout << "\n=== PREMIUM ACCOUNT STATEMENT ===\n";
        display();
        std::cout << "Cashback Rate: " << (cashback_rate_ * 100) << "%\n";
        std::cout << "Monthly Fee: $" << get_monthly_fee() << " (WAIVED)\n";
        std::cout << "\nTransaction History:\n";
        for (size_t i = 0; i < transaction_history_.size(); ++i) {
            std::cout << "  " << (i + 1) << ". " 
                      << transaction_history_[i] << "\n";
        }
        std::cout << "================================\n\n";
    }
    
    std::string get_account_type() const override {
        return "Premium";
    }
    
    double get_monthly_fee() const override {
        return 0.0;  // Premium accounts have no monthly fee
    }
};

// 6. Bank class - manages accounts
class Bank {
private:
    std::vector<std::unique_ptr<Account>> accounts_;
    std::string bank_name_;
    
public:
    explicit Bank(std::string name) : bank_name_(std::move(name)) {}
    
    void add_account(std::unique_ptr<Account> account) {
        std::cout << "Account " << account->get_account_number() 
                  << " added to " << bank_name_ << "\n";
        accounts_.push_back(std::move(account));
    }
    
    Account* find_account(const std::string& account_number) {
        for (auto& account : accounts_) {
            if (account->get_account_number() == account_number) {
                return account.get();
            }
        }
        return nullptr;
    }
    
    void display_all_accounts() const {
        std::cout << "\n=== " << bank_name_ << " - All Accounts ===\n";
        for (const auto& account : accounts_) {
            account->display();
            std::cout << "---\n";
        }
    }
    
    void apply_monthly_fees() {
        std::cout << "\n=== Applying Monthly Fees ===\n";
        for (auto& account : accounts_) {
            double fee = account->get_monthly_fee();
            if (fee > 0) {
                std::cout << "Charging $" << fee << " to account " 
                          << account->get_account_number() << "\n";
                account->withdraw(fee);
            }
        }
    }
};

// Main demonstration
int main() {
    try {
        Bank bank("First National Bank");
        
        std::cout << "=== Creating Accounts ===\n";
        
        // Create different account types - POLYMORPHISM
        auto checking = std::make_unique<CheckingAccount>(
            "CHK001", "Alice Johnson", 1000.0, 500.0
        );
        
        auto savings = std::make_unique<SavingsAccount>(
            "SAV001", "Bob Smith", 5000.0, 0.03
        );
        
        auto premium = std::make_unique<PremiumAccount>(
            "PRM001", "Charlie Brown", 10000.0, 0.02
        );
        
        std::cout << "\n=== Testing Operations ===\n";
        
        // Test checking account
        checking->deposit(500);
        checking->withdraw(200);
        checking->withdraw(1500);  // Should use overdraft
        
        // Test savings account
        savings->deposit(1000);
        savings->withdraw(200);
        savings->withdraw(300);
        savings->withdraw(400);  // 4th withdrawal - fee charged
        savings->apply_interest();
        
        // Test premium account
        premium->deposit(2000);  // Gets cashback
        premium->withdraw(500);
        
        // Add accounts to bank
        bank.add_account(std::move(checking));
        bank.add_account(std::move(savings));
        
        // Keep premium for transfer test
        PremiumAccount* prem_ptr = premium.get();
        bank.add_account(std::move(premium));
        
        // Display all accounts - POLYMORPHISM
        bank.display_all_accounts();
        
        // Test transfer - MULTIPLE INHERITANCE
        std::cout << "\n=== Testing Transfer ===\n";
        Account* destination = bank.find_account("CHK001");
        if (destination) {
            prem_ptr->transfer_to(*destination, 1000.0);
        }
        
        // Generate statement - INTERFACE IMPLEMENTATION
        prem_ptr->generate_statement();
        
        // Apply monthly fees
        bank.apply_monthly_fees();
        
        // Final balances
        std::cout << "\n=== Final Balances ===\n";
        bank.display_all_accounts();
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    
    std::cout << "\n=== Program Complete ===\n";
    return 0;
}
```

### Concepts Demonstrated in Example:

**Encapsulation:**
- Private member variables (balance_, account_number_)
- Public interface methods (deposit, withdraw)
- Protected members for derived classes
- Private helper methods (is_within_limit)

**Inheritance:**
- Base class (Account)
- Derived classes (CheckingAccount, SavingsAccount, PremiumAccount)
- Constructor chaining
- Protected members accessible to derived classes

**Polymorphism:**
- Virtual functions (withdraw, deposit, get_account_type)
- Override keyword
- Runtime polymorphism through base class pointers
- Pure virtual functions (abstract methods)

**Abstraction:**
- Abstract base class (Account)
- Pure virtual functions forcing implementation
- Interface classes (Transferable, Reportable)
- Hiding implementation details

**Additional Concepts:**
- Multiple inheritance (PremiumAccount)
- Multiple interfaces
- Smart pointers (unique_ptr)
- Exception handling
- RAII (automatic resource management)
- Const correctness
- Static constexpr for constants

---

## Summary of OOP Gotchas

### Top 10 Common Mistakes

```cpp
// 1. Forgot virtual destructor
class Base { ~Base() {} };  // Should be: virtual ~Base() {}

// 2. Object slicing
Derived d;
Base b = d;  // Sliced! Use pointers/references

// 3. Not using override
void func() {}  // Might not actually override! Use: void func() override {}

// 4. Returning reference to local
const std::string& bad() {
    std::string local = "test";
    return local;  // Dangling reference!
}

// 5. Initialization order
class Bad {
    int y_, x_;
public:
    Bad() : x_(5), y_(x_) {}  // y_ initialized first with garbage!
};

// 6. Most vexing parse
Widget w();  // Function declaration, not object!

// 7. Missing explicit on constructors
class String {
    String(int size) {}  // Should be: explicit String(int size) {}
};

// 8. Calling virtual from constructor
class Base {
    Base() { init(); }  // Calls Base::init, not Derived::init!
    virtual void init() {}
};

// 9. Not checking self-assignment
T& operator=(const T& other) {
    delete data_;  // If this == &other, disaster!
    data_ = new int(*other.data_);
    return *this;
}

// 10. Public inheritance when should be private/protected
class Stack : public std::vector<int> {};  // Wrong! Breaks encapsulation
```

---

## Related Topics
- [Advanced Features](12_advanced_features.md) — move semantics, smart pointers, RAII, and the Rule of Five/Zero that make resource-owning classes safe.
- [Templates](09_templates.md) — generic programming, the *other* major C++ paradigm that complements OOP.
- [Exceptions](17_exceptions.md) — exception safety, `noexcept` destructors, and why RAII is the backbone of error handling.
- [Best Practices](13_best_practices.md) — when to prefer composition over inheritance, and modern idioms.
- [Utility Containers](08_utility_containers.md) — `std::variant` + `std::visit` offer a closed-set alternative to runtime polymorphism.
- [Quick Reference](99_quick_reference.md) — concise reminders of the special members and virtual rules.

## Next Steps
- **Next**: [Sequence Containers →](01_sequence_containers.md)
- **Back to**: [Main README](README.md)

---
*Chapter 0 — Object-Oriented Programming Fundamentals*


