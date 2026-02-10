# Day 4: Object-Oriented Programming in Python

## 🎯 Goal

Master Python's OOP model. Coming from Java, you already think in OOP — but Python's approach is **much less ceremonial**. No access modifiers, no interfaces keyword, no abstract keyword (well, sort of), and powerful dunder (magic) methods that let you customize everything. Coming from JS, you'll recognize the prototype-ish flexibility but with cleaner syntax.

---

## 1. Classes — The Basics

```python
class User:
    """A simple user class."""
    
    def __init__(self, name: str, age: int):
        """Constructor — like Java constructor or JS constructor()."""
        self.name = name    # instance attribute
        self.age = age
    
    def greet(self) -> str:
        """Instance method — 'self' is like 'this' but EXPLICIT."""
        return f"Hi, I'm {self.name}, {self.age} years old"

# Create instance (no 'new' keyword!)
user = User("Alice", 25)
print(user.greet())     # Hi, I'm Alice, 25 years old
print(user.name)        # Alice — all attributes are public by default
```

### `self` explained — the biggest difference from Java/JS

```python
class Dog:
    def bark(self):
        return "Woof!"

# When you call dog.bark(), Python translates it to:
dog = Dog()
dog.bark()       # This is actually Dog.bark(dog) under the hood

# That's why 'self' is the first parameter — it receives the instance
# In Java/JS, 'this' is implicit. In Python, 'self' is EXPLICIT.
# 'self' is just a convention — you could name it anything (but never do!)
```

### Comparison: same class in Java, JS, and Python

```java
// Java — lots of boilerplate
public class User {
    private String name;
    private int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    
    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + "}";
    }
    
    @Override
    public boolean equals(Object o) { /* ... */ }
    
    @Override
    public int hashCode() { /* ... */ }
}
```

```javascript
// JS — medium boilerplate
class User {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
    
    toString() {
        return `User{name='${this.name}', age=${this.age}}`;
    }
}
```

```python
# Python — minimal boilerplate
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age
    
    def __repr__(self):
        return f"User(name='{self.name}', age={self.age})"
```

---

## 2. Class vs Instance Attributes

```python
class User:
    # Class attribute — shared by ALL instances (like Java static field)
    species = "Human"
    user_count = 0
    
    def __init__(self, name: str):
        # Instance attribute — unique to each instance
        self.name = name
        User.user_count += 1  # modify class attribute via class name

alice = User("Alice")
bob = User("Bob")

print(alice.species)      # "Human" — reads from class
print(bob.species)        # "Human" — same
print(User.user_count)    # 2

# ⚠️ TRAP: assigning via instance creates a NEW instance attribute
alice.species = "Robot"    # creates instance attribute on alice only!
print(alice.species)       # "Robot" — instance attribute shadows class attribute
print(bob.species)         # "Human" — still reads from class
print(User.species)        # "Human" — class attribute unchanged

# This is similar to JS prototype chain behavior
```

---

## 3. Access Control — Python's "We're All Adults Here"

Python has **no real private/protected** like Java. Instead, it uses **naming conventions**.

```python
class BankAccount:
    def __init__(self, owner: str, balance: float):
        self.owner = owner          # public (convention: no prefix)
        self._balance = balance     # "protected" (convention: single underscore)
        self.__pin = 1234           # "private" (name mangling: double underscore)
    
    def get_balance(self):
        return self._balance

account = BankAccount("Alice", 1000)

# Public — anyone can access
print(account.owner)          # "Alice" ✅

# "Protected" — CAN access, but convention says "don't touch from outside"
print(account._balance)       # 1000 ✅ (works, but linters may warn)

# "Private" — name mangling makes it harder to access
# print(account.__pin)        # ❌ AttributeError!
print(account._BankAccount__pin)  # 1234 ✅ (you CAN access it, but DON'T)
# Python mangles __pin to _BankAccount__pin to avoid accidental access
```

### The philosophy

```
Java:   "I'll enforce access with private/protected/public"
Python: "I'll trust you to respect _conventions. We're all consenting adults."

_single_underscore  → "internal use" (like package-private in Java)
__double_underscore → "name mangled" (for avoiding conflicts in subclasses)
__dunder__          → "magic/special" (Python's own, never invent new ones)
```

---

## 4. Properties — Python's Getters/Setters

Instead of Java-style `getName()`/`setName()`, Python uses the `@property` decorator for clean attribute-style access with validation.

```python
class Temperature:
    def __init__(self, celsius: float):
        self._celsius = celsius  # underscore = internal storage
    
    @property
    def celsius(self) -> float:
        """Getter — accessed like an attribute, not a method."""
        return self._celsius
    
    @celsius.setter
    def celsius(self, value: float):
        """Setter — called when you assign to the property."""
        if value < -273.15:
            raise ValueError("Temperature below absolute zero!")
        self._celsius = value
    
    @celsius.deleter
    def celsius(self):
        """Deleter — called on 'del temp.celsius'."""
        print("Deleting temperature")
        del self._celsius
    
    @property
    def fahrenheit(self) -> float:
        """Computed property — read-only (no setter defined)."""
        return self._celsius * 9/5 + 32

temp = Temperature(25)
print(temp.celsius)      # 25 (calls getter — looks like attribute access!)
print(temp.fahrenheit)   # 77.0

temp.celsius = 30        # calls setter
# temp.celsius = -300    # ❌ ValueError: Temperature below absolute zero!
# temp.fahrenheit = 100  # ❌ AttributeError: can't set (no setter defined)
```

### Why properties are better than Java getters/setters

```python
# In Java, you start with:
#   private int age;
#   public int getAge() { return age; }
#   public void setAge(int age) { this.age = age; }
# Even if you need no validation. "Just in case."

# In Python, start simple:
class User:
    def __init__(self, age: int):
        self.age = age  # direct attribute

user = User(25)
user.age = 26  # direct access

# Later, if you NEED validation, add a property WITHOUT changing the API:
class User:
    def __init__(self, age: int):
        self._age = age
    
    @property
    def age(self):
        return self._age
    
    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("Age can't be negative")
        self._age = value

# All existing code still works! user.age = 26 — same syntax
# This is why Python doesn't need getters/setters from the start
```

---

## 5. Dunder (Magic) Methods — Customize Everything

Dunder methods (`__double_underscore__`) let you define how your objects behave with operators, built-in functions, and Python's protocols. Like Java's `toString()`, `equals()`, `compareTo()` — but for everything.

### String representation

```python
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    def __repr__(self) -> str:
        """For developers — unambiguous, ideally eval()-able.
        Called by repr(), in REPL, and in containers."""
        return f"Point({self.x}, {self.y})"
    
    def __str__(self) -> str:
        """For users — readable.
        Called by str(), print(), f-strings."""
        return f"({self.x}, {self.y})"

p = Point(3, 4)
print(p)          # (3, 4)        — calls __str__
print(repr(p))    # Point(3, 4)   — calls __repr__
print(f"Point: {p}")  # Point: (3, 4)  — calls __str__
print([p])        # [Point(3, 4)]  — containers use __repr__

# Rule: Always implement __repr__. Implement __str__ only if you want
# different user-facing output. If only __repr__ exists, it's used for both.

# Java equivalent: toString()
# JS equivalent: toString() and Symbol.toPrimitive
```

### Comparison operators

```python
class Money:
    def __init__(self, amount: float, currency: str = "USD"):
        self.amount = amount
        self.currency = currency
    
    def __eq__(self, other) -> bool:
        """== operator. Java: equals()"""
        if not isinstance(other, Money):
            return NotImplemented  # let Python try other.__eq__(self)
        return self.amount == other.amount and self.currency == other.currency
    
    def __lt__(self, other) -> bool:
        """< operator. Java: compareTo() < 0"""
        if not isinstance(other, Money):
            return NotImplemented
        if self.currency != other.currency:
            raise ValueError("Can't compare different currencies")
        return self.amount < other.amount
    
    def __le__(self, other) -> bool:
        """<= operator."""
        return self == other or self < other
    
    def __hash__(self) -> int:
        """Required if __eq__ is defined and you want to use in sets/dict keys."""
        return hash((self.amount, self.currency))
    
    def __repr__(self):
        return f"Money({self.amount}, '{self.currency}')"

a = Money(100, "USD")
b = Money(200, "USD")
print(a == b)         # False
print(a < b)          # True
print(sorted([b, a])) # [Money(100, 'USD'), Money(200, 'USD')]

# 💡 Shortcut: use @functools.total_ordering to only define __eq__ and __lt__
# and Python auto-generates __le__, __gt__, __ge__
from functools import total_ordering

@total_ordering
class Money:
    # ... only need __eq__ and __lt__, rest auto-generated
    pass
```

### Arithmetic operators

```python
class Vector:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    def __add__(self, other):
        """+ operator"""
        return Vector(self.x + other.x, self.y + other.y)
    
    def __sub__(self, other):
        """- operator"""
        return Vector(self.x - other.x, self.y - other.y)
    
    def __mul__(self, scalar):
        """* operator (vector * scalar)"""
        if isinstance(scalar, (int, float)):
            return Vector(self.x * scalar, self.y * scalar)
        return NotImplemented
    
    def __rmul__(self, scalar):
        """Reverse * (scalar * vector) — called when left operand doesn't support *"""
        return self.__mul__(scalar)
    
    def __abs__(self):
        """abs() function — returns magnitude"""
        return (self.x ** 2 + self.y ** 2) ** 0.5
    
    def __bool__(self):
        """bool() / truthiness — False if zero vector"""
        return self.x != 0 or self.y != 0
    
    def __len__(self):
        """len() — could return number of dimensions"""
        return 2
    
    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)       # Vector(4, 6)
print(v1 * 3)        # Vector(3, 6)
print(3 * v1)        # Vector(3, 6) — rmul
print(abs(v2))        # 5.0
print(bool(Vector(0, 0)))  # False
```

### Container protocol — make your class behave like a list/dict

```python
class Playlist:
    def __init__(self, name: str, songs: list[str] = None):
        self.name = name
        self._songs = songs or []
    
    def __len__(self) -> int:
        """len(playlist)"""
        return len(self._songs)
    
    def __getitem__(self, index):
        """playlist[0], playlist[1:3], enables iteration!"""
        return self._songs[index]
    
    def __setitem__(self, index, value):
        """playlist[0] = 'new song'"""
        self._songs[index] = value
    
    def __delitem__(self, index):
        """del playlist[0]"""
        del self._songs[index]
    
    def __contains__(self, item) -> bool:
        """'song' in playlist"""
        return item in self._songs
    
    def __iter__(self):
        """for song in playlist:"""
        return iter(self._songs)
    
    def __repr__(self):
        return f"Playlist('{self.name}', {self._songs})"

p = Playlist("Rock", ["Bohemian Rhapsody", "Stairway to Heaven", "Hotel California"])
print(len(p))            # 3
print(p[0])              # Bohemian Rhapsody
print(p[1:])             # ["Stairway to Heaven", "Hotel California"]
print("Hotel California" in p)  # True
for song in p:           # iteration works!
    print(song)
```

### Context manager protocol (preview — full coverage Day 7)

```python
class Timer:
    """Usage: with Timer('operation'): ..."""
    
    def __init__(self, label: str):
        self.label = label
    
    def __enter__(self):
        """Called when entering 'with' block."""
        import time
        self.start = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block."""
        import time
        elapsed = time.time() - self.start
        print(f"{self.label}: {elapsed:.4f}s")
        return False  # don't suppress exceptions

with Timer("sorting"):
    sorted(range(1_000_000, 0, -1))
# sorting: 0.0892s
```

### Complete dunder methods reference

| Category | Methods | Java Equivalent |
|----------|---------|-----------------|
| String | `__repr__`, `__str__`, `__format__` | `toString()` |
| Comparison | `__eq__`, `__lt__`, `__le__`, `__gt__`, `__ge__` | `equals()`, `compareTo()` |
| Hashing | `__hash__` | `hashCode()` |
| Arithmetic | `__add__`, `__sub__`, `__mul__`, `__truediv__`, `__mod__`, `__pow__` | — |
| Unary | `__neg__`, `__abs__`, `__bool__`, `__int__`, `__float__` | — |
| Container | `__len__`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__`, `__iter__` | `size()`, `get()`, `iterator()` |
| Callable | `__call__` | `apply()` in functional interfaces |
| Context | `__enter__`, `__exit__` | `try-with-resources` / `AutoCloseable` |
| Attribute | `__getattr__`, `__setattr__`, `__delattr__` | — |
| Creation | `__init__`, `__new__`, `__del__` | constructor, `finalize()` |

---

## 6. `__call__` — Making Instances Callable

```python
class Validator:
    """Callable class — instances can be used like functions."""
    
    def __init__(self, min_val: float, max_val: float):
        self.min_val = min_val
        self.max_val = max_val
    
    def __call__(self, value: float) -> bool:
        """Called when you use instance as a function."""
        return self.min_val <= value <= self.max_val

# Create reusable validators
is_percentage = Validator(0, 100)
is_temperature = Validator(-273.15, 1_000_000)

print(is_percentage(50))     # True — calling the instance like a function!
print(is_percentage(150))    # False
print(is_temperature(-300))  # False

# Why not just use a function? Because __call__ can carry STATE:
class Counter:
    def __init__(self):
        self.count = 0
    
    def __call__(self):
        self.count += 1
        return self.count

counter = Counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
# Cleaner than closure-based counters for complex cases
```

---

## 7. Inheritance

### Single inheritance

```python
class Animal:
    def __init__(self, name: str, sound: str):
        self.name = name
        self.sound = sound
    
    def speak(self) -> str:
        return f"{self.name} says {self.sound}!"
    
    def __repr__(self):
        return f"{type(self).__name__}('{self.name}')"

class Dog(Animal):
    def __init__(self, name: str, breed: str):
        super().__init__(name, "Woof")  # call parent constructor
        self.breed = breed              # new attribute
    
    def fetch(self, item: str) -> str:
        return f"{self.name} fetches the {item}!"

class Cat(Animal):
    def __init__(self, name: str):
        super().__init__(name, "Meow")
    
    def speak(self) -> str:
        """Override parent method."""
        return f"{self.name} reluctantly says {self.sound}..."

dog = Dog("Rex", "German Shepherd")
cat = Cat("Whiskers")

print(dog.speak())       # Rex says Woof!
print(dog.fetch("ball")) # Rex fetches the ball!
print(cat.speak())       # Whiskers reluctantly says Meow...

# Type checking
isinstance(dog, Dog)     # True
isinstance(dog, Animal)  # True
isinstance(cat, Dog)     # False
issubclass(Dog, Animal)  # True
```

### Multiple inheritance & MRO

Python supports multiple inheritance (Java doesn't — only interfaces). This is powerful but needs understanding.

```python
class Flyable:
    def fly(self):
        return f"{self.name} is flying!"

class Swimmable:
    def swim(self):
        return f"{self.name} is swimming!"

class Duck(Animal, Flyable, Swimmable):
    def __init__(self, name: str):
        super().__init__(name, "Quack")

donald = Duck("Donald")
print(donald.speak())  # Donald says Quack!
print(donald.fly())    # Donald is flying!
print(donald.swim())   # Donald is swimming!
```

### MRO — Method Resolution Order (the Diamond Problem)

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

d = D()
print(d.method())  # "B" — not "A", not "C"

# Why? Python uses C3 linearization (MRO):
print(D.__mro__)
# (D, B, C, A, object)
# Python searches left to right: D → B → C → A → object
# Found method in B first, so "B" wins

# Rule: left parent has priority, but each class appears only once
```

### Mixins — the Pythonic way to compose behavior

```python
# Mixins are small classes that add specific functionality
# Convention: suffix with "Mixin"

class JsonMixin:
    """Adds JSON serialization to any class with __dict__."""
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__, default=str)
    
    @classmethod
    def from_json(cls, json_str: str):
        import json
        data = json.loads(json_str)
        return cls(**data)

class TimestampMixin:
    """Adds created_at timestamp."""
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        original_init = cls.__init__
        
        def new_init(self, *args, **kwargs):
            from datetime import datetime
            original_init(self, *args, **kwargs)
            self.created_at = datetime.now()
        
        cls.__init__ = new_init

class LoggableMixin:
    """Adds logging capability."""
    def log(self, message: str):
        print(f"[{type(self).__name__}] {message}")

# Compose multiple behaviors
class User(JsonMixin, LoggableMixin):
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

user = User("Alice", 25)
print(user.to_json())  # {"name": "Alice", "age": 25}
user.log("User created")  # [User] User created

# This is like TypeScript mixins or Java interfaces with default methods
```

---

## 8. Class Methods and Static Methods

```python
class Date:
    def __init__(self, year: int, month: int, day: int):
        self.year = year
        self.month = month
        self.day = day
    
    # Instance method — has access to instance (self)
    def format(self) -> str:
        return f"{self.year}-{self.month:02d}-{self.day:02d}"
    
    # Class method — has access to class (cls), not instance
    # Like Java static factory methods
    @classmethod
    def from_string(cls, date_str: str) -> "Date":
        """Alternative constructor — parse from string."""
        year, month, day = map(int, date_str.split("-"))
        return cls(year, month, day)  # cls, not Date — works with subclasses!
    
    @classmethod
    def today(cls) -> "Date":
        """Another alternative constructor."""
        from datetime import date
        d = date.today()
        return cls(d.year, d.month, d.day)
    
    # Static method — no access to instance or class
    # Just a regular function that lives in the class namespace
    @staticmethod
    def is_valid(year: int, month: int, day: int) -> bool:
        """Utility function — doesn't need instance or class."""
        return 1 <= month <= 12 and 1 <= day <= 31
    
    def __repr__(self):
        return f"Date({self.year}, {self.month}, {self.day})"

# Usage
d1 = Date(2025, 2, 10)          # regular constructor
d2 = Date.from_string("2025-02-10")  # classmethod constructor
d3 = Date.today()                # classmethod constructor
print(Date.is_valid(2025, 13, 1))    # False — staticmethod
```

### When to use which?

```
Instance method (def method(self)):
  → Needs access to instance data
  → Most common type
  
Class method (@classmethod, def method(cls)):
  → Alternative constructors (from_string, from_json, from_dict)
  → Factory methods that should work with subclasses
  → Accessing/modifying class-level state

Static method (@staticmethod, def method()):
  → Utility/helper that logically belongs to the class
  → Doesn't need instance or class
  → Could be a standalone function, but grouping it makes sense
```

---

## 9. Abstract Base Classes (ABC)

Python's version of Java interfaces/abstract classes.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    """Abstract class — can't be instantiated directly."""
    
    @abstractmethod
    def area(self) -> float:
        """Subclasses MUST implement this."""
        pass
    
    @abstractmethod
    def perimeter(self) -> float:
        pass
    
    # Concrete method — inherited as-is
    def description(self) -> str:
        return f"{type(self).__name__}: area={self.area():.2f}"

# shape = Shape()  # ❌ TypeError: Can't instantiate abstract class

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def area(self) -> float:
        from math import pi
        return pi * self.radius ** 2
    
    def perimeter(self) -> float:
        from math import pi
        return 2 * pi * self.radius

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height
    
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

# If you forget to implement an abstract method:
# class Triangle(Shape):
#     def area(self): return 0
#     # Missing perimeter!
# t = Triangle()  # ❌ TypeError: Can't instantiate — perimeter is abstract

circle = Circle(5)
rect = Rectangle(3, 4)
print(circle.description())  # Circle: area=78.54
print(rect.description())    # Rectangle: area=12.00

# Polymorphism
shapes: list[Shape] = [Circle(5), Rectangle(3, 4), Circle(2)]
total_area = sum(s.area() for s in shapes)
```

---

## 10. Dataclasses — Modern Python Classes

Dataclasses auto-generate `__init__`, `__repr__`, `__eq__`, and more. Like Java records or Lombok's `@Data`.

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    email: str = ""  # default value
    tags: list[str] = field(default_factory=list)  # mutable default — safe!
    # Remember Day 2: never use mutable default args
    # field(default_factory=list) creates a NEW list for each instance

alice = User("Alice", 25, "alice@example.com")
bob = User("Bob", 30)

print(alice)           # User(name='Alice', age=25, email='alice@example.com', tags=[])
print(alice == bob)    # False (auto-generated __eq__ compares all fields)
```

### Dataclass features

```python
from dataclasses import dataclass, field, asdict, astuple

@dataclass
class Product:
    name: str
    price: float
    quantity: int = 0
    tags: list[str] = field(default_factory=list)
    _internal: str = field(default="", repr=False, compare=False)  # excluded from repr and eq
    
    @property
    def total_value(self) -> float:
        """Computed property — not a field."""
        return self.price * self.quantity
    
    def __post_init__(self):
        """Called after __init__ — for validation/transformation."""
        if self.price < 0:
            raise ValueError("Price can't be negative")
        self.name = self.name.strip().title()

p = Product("  widget  ", 9.99, 100, ["sale"])
print(p)            # Product(name='Widget', price=9.99, quantity=100, tags=['sale'])
print(p.total_value) # 999.0

# Convert to dict/tuple
asdict(p)    # {"name": "Widget", "price": 9.99, "quantity": 100, "tags": ["sale"]}
astuple(p)   # ("Widget", 9.99, 100, ["sale"])
```

### Frozen dataclasses — immutable (like Java records)

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(3, 4)
# p.x = 10  # ❌ FrozenInstanceError!

# Can be used as dict keys and in sets (auto-generates __hash__)
points = {Point(0, 0): "origin", Point(1, 1): "unit"}
```

### Dataclass ordering

```python
@dataclass(order=True)
class Student:
    gpa: float
    name: str = field(compare=False)  # exclude from ordering
    age: int = field(compare=False)

students = [
    Student(3.5, "Alice", 20),
    Student(3.9, "Bob", 22),
    Student(3.2, "Charlie", 21),
]

print(sorted(students))
# Sorted by gpa: [Student(3.2, ...), Student(3.5, ...), Student(3.9, ...)]
```

### Dataclass vs NamedTuple vs Regular Class

```
Regular class:
  ✅ Full control
  ❌ Lots of boilerplate
  Use when: complex behavior, custom __init__, many methods

@dataclass:
  ✅ Auto __init__, __repr__, __eq__, __hash__ (if frozen)
  ✅ Mutable by default, can be frozen
  ✅ Can have methods, properties, __post_init__
  Use when: data-carrying classes (most common choice!)

NamedTuple:
  ✅ Immutable, lightweight, tuple-based
  ✅ Can be dict key
  ❌ Can't add mutable behavior easily
  Use when: simple immutable records, function return values

For Data Engineering: dataclasses + Pydantic models (Day 5) will be your main tools.
```

---

## 11. Encapsulation — Real-World Pattern

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class BankAccount:
    owner: str
    _balance: float = field(default=0, repr=False)
    _transactions: list = field(default_factory=list, repr=False)
    
    @property
    def balance(self) -> float:
        return self._balance
    
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount
        self._transactions.append({
            "type": "deposit",
            "amount": amount,
            "timestamp": datetime.now(),
            "balance_after": self._balance,
        })
    
    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal must be positive")
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount
        self._transactions.append({
            "type": "withdrawal",
            "amount": amount,
            "timestamp": datetime.now(),
            "balance_after": self._balance,
        })
    
    @property
    def statement(self) -> str:
        lines = [f"Statement for {self.owner}"]
        for t in self._transactions:
            lines.append(f"  {t['type']:12s} ${t['amount']:>8.2f}  →  ${t['balance_after']:>8.2f}")
        lines.append(f"  {'Balance':12s} ${self._balance:>8.2f}")
        return "\n".join(lines)

acc = BankAccount("Alice")
acc.deposit(1000)
acc.deposit(500)
acc.withdraw(200)
print(acc.statement)
# Statement for Alice
#   deposit      $ 1000.00  →  $ 1000.00
#   deposit      $  500.00  →  $ 1500.00
#   withdrawal   $  200.00  →  $ 1300.00
#   Balance      $ 1300.00
```

---

## 📝 Practice Tasks

### Task 1: Vector Class
Build a `Vector2D` class with: addition, subtraction, scalar multiplication, dot product, magnitude, equality, and string representation. Use dunder methods for all operators.

### Task 2: Deck of Cards
Create `Card` (dataclass) and `Deck` classes. Deck should support: `len()`, `[]` indexing, iteration, `in` checking, `shuffle()`, `deal(n)`. Use dunder methods to make Deck behave like a collection.

### Task 3: Linked List
Implement a singly linked list with a `Node` class and `LinkedList` class. Support: `append`, `prepend`, `len()`, iteration (`__iter__`), `in` checking, `[]` indexing, and `__repr__`.

### Task 4: Shape Hierarchy
Create an abstract `Shape` class with `area()` and `perimeter()` abstract methods. Implement `Circle`, `Rectangle`, `Triangle`. Add a `ShapeCollection` class that stores shapes and can compute total area, find the largest, and sort by area.

### Task 5: Employee System
Build using inheritance + dataclasses:
- Base `Employee` (name, salary)
- `Developer(Employee)` — adds language, has bonus calculation
- `Manager(Employee)` — adds team (list of employees), has different bonus
- `Department` — manages employees, calculates total salary, finds highest paid

### Task 6: Custom Dict
Create a `CaseInsensitiveDict` that works like a regular dict but treats keys as case-insensitive. Implement `__getitem__`, `__setitem__`, `__contains__`, `__delitem__`, `__len__`, `__iter__`, `__repr__`.

```python
d = CaseInsensitiveDict()
d["Name"] = "Alice"
print(d["name"])    # Alice
print(d["NAME"])    # Alice
print("name" in d)  # True
```

---

## 📚 Resources

- [Python Official — Classes](https://docs.python.org/3/tutorial/classes.html)
- [Real Python — OOP in Python](https://realpython.com/python3-object-oriented-programming/)
- [Real Python — Python @property](https://realpython.com/python-property/)
- [Real Python — Dataclasses](https://realpython.com/python-data-classes/)
- [Real Python — Dunder Methods](https://realpython.com/operator-function-overloading/)
- [Real Python — Inheritance and Composition](https://realpython.com/inheritance-composition-python/)
- [Real Python — Abstract Base Classes](https://realpython.com/python-abc/)
- [Python Data Model — Official Docs](https://docs.python.org/3/reference/datamodel.html) (the definitive reference for all dunders)

---

## 🔑 Key Takeaways

1. **`self` is explicit** — always the first parameter of instance methods.
2. **No real private** — use `_convention` and trust, not enforcement.
3. **Properties replace getters/setters** — start with plain attributes, add `@property` only when needed.
4. **Dunder methods** customize everything — operators, `len()`, iteration, comparison, string representation.
5. **Dataclasses** eliminate boilerplate — use them as your default for data-carrying classes.
6. **`@classmethod`** for alternative constructors, **`@staticmethod`** for utilities.
7. **ABC** for interfaces/contracts — use when you need to enforce method implementation.
8. **Multiple inheritance** works via MRO (C3 linearization) — prefer mixins for composing behavior.
9. **`__repr__`** > `__str__` — always implement `__repr__` first.

---

> **Tomorrow (Day 5):** Exceptions, Modules, Type Hints & Pydantic — the building blocks for professional Python code and data validation.
