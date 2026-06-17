# Python Iterables, Iterators, Generators, and `yield` — The Mental Model That Finally Made It Click

I kept getting confused between:

* Iterable
* Iterator
* Generator
* Generator expression
* Generator function
* `yield`
* `iter()`
* `next()`

The breakthrough came when I realized that almost everything in Python iteration is built on just two functions:

```python
iter(obj)
next(iterator)
```

---

# The Core Iteration Model

Whenever Python sees:

```python
for x in something:
    ...
```

it roughly does:

```python
it = iter(something)

while True:
    try:
        x = next(it)
        ...
    except StopIteration:
        break
```

This is the foundation of iteration in Python.

---

# Iterable

An iterable is:

> An object from which Python can obtain an iterator.

Examples:

```python
list
tuple
dict
set
string
range
generator
```

These work:

```python
iter(obj)
```

Examples:

```python
iter([1,2,3])
iter((1,2,3))
iter("abc")
```

A list is iterable.

A tuple is iterable.

A dict is iterable.

---

# Iterator

An iterator is:

> An object that knows how to produce the next value and remembers where it currently is.

It supports:

```python
next(iterator)
```

Example:

```python
arr = [10,20,30]

it = iter(arr)

next(it)  # 10
next(it)  # 20
next(it)  # 30
```

Then:

```python
next(it)
```

raises:

```python
StopIteration
```

because the iterator is exhausted.

---

# Why Isn't a List an Iterator?

This confused me.

A list is NOT an iterator.

This fails:

```python
arr = [1,2,3]

next(arr)
```

Error:

```python
TypeError
```

Because a list doesn't store iteration state.

Instead:

```python
it = iter(arr)
```

creates a separate iterator object.

Think:

```text
List = book
Iterator = bookmark
```

The book doesn't remember where you stopped reading.

The bookmark does.

---

# Why Does Python Need iter()?

At first I wondered:

> If for-loops already work with lists and generators, why do we even need iter()?

Answer:

Most of the time, we don't.

Python uses it internally.

For example:

```python
for x in arr:
```

Python automatically does:

```python
it = iter(arr)
```

Similarly:

```python
sum(arr)
max(arr)
list(arr)
tuple(arr)
```

all use iterators internally.

`iter()` exists because Python needs a standard way to obtain an iterator from any iterable.

---

# Generator

A generator is:

> A special kind of iterator that generates values lazily (on demand).

Example:

```python
g = (x*x for x in range(5))
```

This is a generator.

It already supports:

```python
next(g)
```

Unlike lists:

```python
next([1,2,3])   # Error
next(g)         # Works
```

A generator is both:

```text
Iterable
Iterator
```

at the same time.

---

# Generator Expression

Generator expression:

```python
(x*x for x in arr)
```

List comprehension:

```python
[x*x for x in arr]
```

Difference:

List comprehension creates:

```python
[1,4,9,16]
```

immediately.

Generator expression creates:

```python
<generator object>
```

and produces values only when asked.

---

# Generator Function

A generator function is:

> A function containing yield.

Example:

```python
def squares():
    yield 1
    yield 4
    yield 9
```

Notice:

```python
yield
```

instead of:

```python
return
```

Calling:

```python
g = squares()
```

does NOT run the function.

Instead it creates a generator object.

---

# What Does yield Actually Do?

The biggest confusion.

I kept thinking:

> Is yield the generator?

No.

They are different.

---

## yield

`yield` is just an instruction:

```text
Return this value.
Pause here.
Resume later.
```

Example:

```python
def f():
    yield 10
    yield 20
```

---

## Generator Object

The generator object stores:

```text
Current line number
Local variables
Paused state
```

Example:

```python
g = f()
```

State lives in:

```python
g
```

NOT in:

```python
f
```

---

# Why next(f()) Behaves Differently

This confused me a lot.

Consider:

```python
def f():
    yield 10
    yield 20
```

---

Works as expected:

```python
g = f()

next(g)
next(g)
```

Output:

```text
10
20
```

---

But:

```python
next(f())
next(f())
```

returns:

```text
10
10
```

Why?

Because:

```python
f()
```

creates a NEW generator each time.

Equivalent to:

```python
g1 = f()
next(g1)

g2 = f()
next(g2)
```

Each generator starts from the beginning.

---

# How Does Python Know Something Is a Generator Function?

Python checks whether the function contains:

```python
yield
```

Example:

```python
def f():
    return 10
```

Normal function.

---

```python
def g():
    yield 10
```

Generator function.

Presence of yield changes everything.

---

# yield vs return

Normal function:

```python
def f():
    return 10
```

Behavior:

```text
Return value
Function dies
```

---

Generator:

```python
def f():
    yield 10
```

Behavior:

```text
Return value
Pause
Resume later
```

---

# Why Is It Called a Generator?

Because it generates values on demand.

List approach:

```python
[0,1,4,9,16]
```

Everything exists immediately.

Generator:

```python
0
pause

1
pause

4
pause
```

Produces values only when requested.

---

# Memory Optimization

List version:

```python
sum([x*x for x in arr])
```

Creates a full intermediate list.

Generator version:

```python
sum(x*x for x in arr)
```

No intermediate list.

Values are generated and consumed one at a time.

Much better for large datasets.

---

# How Does sum() Consume a Generator?

Conceptually:

```python
def my_sum(iterable):
    total = 0

    for x in iterable:
        total += x

    return total
```

Internally:

```python
next(generator)
next(generator)
next(generator)
...
```

until:

```python
StopIteration
```

---

# Why Generators Get Exhausted

Example:

```python
g = (x*x for x in range(5))

sum(g)
sum(g)
```

Output:

```text
30
0
```

The first sum consumed all values.

The generator has nothing left.

---

# The Final Mental Model

```text
Iterable
    ↓
Can create iterator via iter()

Iterator
    ↓
Produces values via next()
Remembers position

Generator
    ↓
Special iterator
Produces values lazily

Generator Function
    ↓
Function containing yield

yield
    ↓
Pause instruction
```

The one sentence that finally made everything click:

> Iterable = object that can create an iterator.
>
> Iterator = object that remembers position and produces the next value.
>
> Generator = a special iterator that generates values lazily using yield.
>
> yield = pause here and continue later.
 
