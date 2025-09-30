
### `from dataclasses import dataclass`

This line is importing the **`dataclass` decorator** from Python’s built-in [`dataclasses`](https://docs.python.org/3/library/dataclasses.html) module (introduced in Python 3.7).

A **dataclass** is a Python class that’s mainly used to store data, and the `@dataclass` decorator automatically generates a lot of the boilerplate code for you.

---

### Without `dataclass`

If you write a regular class to hold data, you usually need:

```python
class Person:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"Person(name={self.name}, age={self.age})"
```

That’s a lot of typing just to hold two fields.

---

### With `@dataclass`

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int
```

✅ That’s it! Python automatically creates:

* `__init__` → so you can do `Person("Alice", 30)`
* `__repr__` → gives a nice string output like `Person(name='Alice', age=30)`
* `__eq__` → allows comparison like `Person("Alice", 30) == Person("Alice", 30)`

---

### Example in action

```python
p1 = Person("Alice", 30)
p2 = Person("Alice", 30)
p3 = Person("Bob", 25)

print(p1)          # Person(name='Alice', age=30)
print(p1 == p2)    # True (auto-generated equality check)
print(p1 == p3)    # False
```

---

### Why use `dataclass`?

* Less boilerplate (no need to write init, repr, eq manually).
* Type annotations are built-in.
* You can make them **immutable** (`frozen=True`).
* You can add default values easily.

Example with defaults:

```python
@dataclass
class Car:
    brand: str
    year: int = 2024   # default value
```

---

👉 So in short:
`from dataclasses import dataclass` lets you write **clean, concise data-holding classes** with less code.

