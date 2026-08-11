# Type hints, dataclasses, Pydantic models

type hints -> static contracts for developers and tooling
dataclasses -> boilerplate-free data containers
Pydantic Models -> runtime validation + serialization + parsing

> boilerplate -> code that you have to write repeatedly even though it doesn't contain much unique logic

A common evolution in Python projects is:

```
dicts
  ↓
type hints
  ↓
dataclasses
  ↓
Pydantic
```


## Type hints

consider:

```python
def calculate_price(price, tax):
    return price + tax
```

with type hints:

```python
def calculate_price(price: float, tax: float) -> float:
    return price + tax
```


Now:

- IDE understands intent
- Static analyzers (mypy, pyright) can detect bugs
- Refactoring becomes safer
- Documentation becomes self-contained


Type hints do not enforce types at runtime!

```python
def add(a: int, b: int) -> int:
    return a + b

print(add("1", "2"))
```

output:

```
12
```

Python executes successfully.

Type hints are mostly for:

- Humans
- IDEs
- Static type checkers


### Optional Return
```python
def find_user(user_id: int) -> str | None:
    ...
```

Older syntax:
```python
from typing import Optional

def get_username(user_id: int) -> Optional[str]:
    if user_id == 1:
        return "Alice"
    return None
```

### Examples

#### List

```python
numbers: list[int]

numbers: list[int] = [1, 2, 3]
```

#### Dictionary

```python
scores: dict[str, int]


# example
scores = {
    "John": 90,
    "Jane": 95
}
```


#### Set

```python
tags: set[str]
```

#### Tuple

```python
point: tuple[float, float]
```
example:
```python
point = (10.5, 20.2)
```

#### Union type
Value can be multiple types.

modern syntax:
```python
value: int | str
```

equivalent:

```python
from typing import Union

value: Union[int, str]
```


#### Any

Disables type safety.

```python
from typing import Any

payload: Any
```

-> not good idea


#### Literal Types

restrict values.

```python
from typing import Literal

def connect(env: Literal["dev", "prod", "test"]):
    pass
```

#### typedDict

For strongly typed dictionaries.

Without:

```python
user = {
    "id": 1,
    "name": "John"
}
```

with:

```python
from typing import TypedDict

class UserDict(TypedDict):
    id: int
    name: str
```

usage:

```python
user: UserDict = {
    "id": 1,
    "name": "John"
}
```

### Type Alias

Python 3.12+

```python
UserId = int
Email = str


# or
type UserId = int
type Email = str
```

### Generics

TypeVar is used to create a type variable for generic code.

```python
from typing import TypeVar

MyType = TypeVar("MyType")

def identity(value: MyType) -> MyType:
    return value
```

This function accepts any type and returns the same type.


#### for classes

Generic[T] tells the type system: "T is a type parameter of the entire Box class."

```python
class Box(Generic[T]):
    def __init__(self, value: T):
        self.value = value

    def get(self) -> T:
        return self.value
```


### Protocols (Duck Typing)

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None:
        ...



class Circle:
    def draw(self) -> None:
        print("Drawing a circle")


class Square:
    def draw(self) -> None:
        print("Drawing a square")


class Person:
    def speak(self) -> None:
        print("Hello")


def render(obj: Drawable) -> None:
    obj.draw()


circle = Circle()
square = Square()
person = Person()

render(circle)   # Drawing a circle
render(square)   # Drawing a square

render(person)   # ❌ Type checker error
```


### Advanced Type Hints


#### Callable

```python
from typing import Callable

Processor = Callable[[str], int]


def process(text: str, processor: Processor) -> int:
    return processor(text)
```

This means -> Processor is an alias for a function that takes a str and returns an int.


#### Self

self -> "The type of the current class."

```python
from typing import Self

class User:
    def set_name(self, name: str) -> Self:
        self.name = name
        return self
```


#### ClassVar

-> "This variable belongs to the class, not to each individual object."

```python
from typing import ClassVar

class User:
    count: ClassVar[int] = 0
```

> count is a class variable because it is shared by the class.

complete example:

```python
from typing import ClassVar

class User:
    count: ClassVar[int] = 0

    def __init__(self, name: str):
        self.name = name
        User.count += 1
```

---

## Dataclasses

Traditional Classes has Lots of boilerplate:

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        ...
```


Dataclass Equivalent:

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
```

Python automatically creates:

- init
- repr
- eq

Usage:
```python
user = User(
    name="John",
    age=30
)

print(user)
```

Output:

```
User(name='John', age=30)
```

unnecessory tip: we can add method to dataclass in this way:
```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    trusted: bool = True # default value

    def introduce(self) -> str:
        return f"My name is {self.name} and I am {self.age} years old."
```


### Default Factory; 

danger:
```python
@dataclass
class User:
    tags: list[str] = []
```

because if:
```
user1 = User()
user2 = User()
```

Neither user1 nor user2 creates a new tags list. Python looks for tags on the object, doesn't find it, and then finds it on the classs. so if:
```python
user1.tags.append("Python")
```

modifies the same list that user2.tags refers to.
```python
print(user1.tags)
# ['Python']

print(user2.tags)
# ['Python']
```


correct:
```python
from dataclass import field

@dataclass
class User:
    tags: list[str] = field(default_factory=list)
```


### Frozen Dataclasses

Immutable object:

```python
@dataclass(frozen=True)
class User:
    name: str
    age: int
```

Trying to modify:

```python
user.age = 50
```

Raises:

```
FrozenInstanceError
```

### Ordering

```python
@dataclass(order=True)
class Product:
    price: float
```
in this way it creates these methods:
```python
__lt__   # <
__le__   # <=
__gt__   # >
__ge__   # >=
```


so we can compare :
```python
p1 = Product(price=20)
p2 = Product(price=25)

print(p1 < p2)
```


### Post Initialization

Validation/computation after construction.

```python
@dataclass
class User:
    name: str
    email: str

    def __post_init__(self):
        self.email = self.email.lower()
```

usage:

```python
User("John", "TEST@MAIL.COM")
```

so stored will be as this:

```python
test@mail.com
```


### Derived Fields

Do not include this field in the automatically generated `__init__()`

```python
@dataclass
class Product:
    price: float
    quantity: int

    total: float = field(init=False)

    def __post_init__(self):
        self.total = self.price * self.quantity
```

### Slots

normal dataclass:

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
```

you can normally do:
```python
user = User("Alice", 25)

user.city = "Berlin"  # ✅
```

With slots=True :

```python

from dataclasses import dataclass

@dataclass(slots=True)
class User:
    name: str
    age: int

```

now:

```python

user = User("Alice", 25)

user.city = "Berlin"  # ❌ AttributeError

```

Because city wasn't declared in the class.


> slots=True => more memory-efficient and restricted attributes.


