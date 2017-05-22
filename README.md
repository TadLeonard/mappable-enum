# Enumap: a mappable Enum
An `Enum` that helps you manage named, ordered values in a strict but convenient way.
`Enumap` isn't a container, it's a just store of keys that makes ordered containers
from your data. This means:

# Usage
## Make ordered collections with Enum members as keys/fields
```
from enumap import Enumap

>>> class Pie(str, Enumap):
...    rhubarb = "tart"
...    cherry = "sweet"
...    mud = "savory"
...
>>> Pie.map(10, 23, mud=1)  # args and/or kwargs
OrderedDict([('rhubarb', 10), ('cherry', 23), ('mud', 1)])
>>> Pie.tuple(10, 23, 1000, cherry=1)  # override with kwargs
Pie_tuple(rhubarb=10, cherry=1, mud=1000)
```

## Use `Enumap` with type annotations for deserialization
```
>>> import arrow  # convenient datetime library
>>> from enum import auto
>>> class Order(Enumap):
...    index: int = "Order ID"
...    cost: Decimal = "Total pretax cost"
...    due_on: arrow.get = "Delivery date"
...
>>> serialized = "134,25014.99,2017-06-20"  # line from a CSV, for example
>>> Order.tuple_casted(*serialized.split(","))
Order_tuple(index=134, cost=Decimal('25014.99'), due_on=<Arrow [2017-06-20T00:00:00+00:00]>)
```

## `Enumap` keys are strictly required
```
>>> Pie.tuple(rhubarb=1, cherry=1, mud=3, blueberry=30)
...
KeyError: "Pie requires keys ('rhubarb', 'cherry', 'mud'); invalid: {'blueberry'}, missing: {}"
>>> Pie.map(1, 1)
...
KeyError: "Pie requires keys ('rhubarb', 'cherry', 'mud'); invalid: {}, missing: {'mud'}"
```

## Set `Enumap` types without annotations
Handy if you prefer the functional style of `Enum` construction.
```
>>> Part = Enumap("Part", "resistor capacitor inductor")
>>> Part.set_types(int, capacitor=float, inductor=float)
>>> some_raw_data = "10 90e-9 3.4e-30"
>>> Part.map_casted(*some_raw_data.split())
OrderedDict([('resistor', 10), ('capacitor', 90e-9), ('inductor', 0.0034)])
```

## Sparse mappings with the less strict `SparseEnumap`
`None` is provided for missing keys.
```
>>> from enumap import SparseEnumap
>>> SparsePie = SparseEnumap("SparsePie", "rhubarb cherry mud")
>>> SparsePie.tuple()
SparsePie_tuple(rhubarb=None, cherry=None, mud=None)
>>> SparsePie.tuple(2, cherry=1)
SparsePie_tuple(rhubarb=2, cherry=1, mud=None)
```

Still, invalid keys are not allowed:
```
>>> SparsePie.tuple(cherry=1, rhubarb=1, mud=3, blueberry=30)
...
KeyError: "SparsePie has keys ('rhubarb', 'cherry', 'mud'), got invalid keys {'blueberry'}"
```

# Why?
`Enumap` lets you define a set of keys or field names in your *once* in your code. This means:

1. You get a single place to declaratively define an
   immutable set of keys or field names
2. You can refer back to your keys and field names elsewhere without the
   uncertainty of using string literals or hard-to-debug global variables
3. You can make containers from your keys without worrying that you've
   omitted or mispelled a key or field name

## Why not just dictionaries with string keys?
String literals make fine dictionary keys for small projects.

```
data = dict(assembly="A1", reference="R3",
            name="resistor", subassembly=["U3", "W12"])
...

# later on
part_reference = data["reference"]
```

When a project grows beyond a certain size, you often see people keeping
field names bound to global variables so that they can be imported in other
modules. Usually the motivation for doing this is to improve clarity and
ease refactoring.

```
PART_ASSEMBLY = "assembly"
PART_REFERENCE = "reference"
PART_SUBASSEMBLY = "subassembly"
PART_NAME = "name"
...
```

...and later they might use these global variables as dictionary keys:

```
assembly = data[PART_ASSEMBLY]
subassembly = data.get(PART_SUBASSEMBLY, [])
...
```

After a while it might be tempting to group key variables in an empty class:

```
class Part:
    assembly = "assembly"
    reference = "reference"
    ...

assembly = data[Part.assembly]
...
```

Now the code is more refactorable and less prone to error,
but later on we may want a modified copy of our dictionary:

```
new_data = dict(data, asembly="A2")  # "assembly" is misspelled!
```

Now we've regressed to using plain strings and our code is prone to error
once more. We could get around this by using advanced dictionary unpacking:

```
new_data = {**data, **{Part.assembly: "A2"}}
```

... but we've sacrificed readability for the sake of correctness.


## How about plain `namedtuple`s?
Namedtuples are great for making your code correct. They're ordered,
immutable, and they insist on the field names they were born with.
```
Part = namedtuple("Part", "assembly reference subassembly name")
data = Part(assembly="A1", name="resistor", reference="R3", subassembly=[])
```

Looks great so far. We've got objects to pass around with immutable fields
and convenient, expressive attribute access. Let's say we want to JSONify
a `Part`. We'll want to convert it to a `dict`:
```
data_as_dict = data._as_dict()
```

So now we're left using a private `namedtuple` method just to
get a dictionary out of our data! Say we're not done yet and we want
to update a field in our dictionary before sending it out as JSON:
```
data_as_dict.update(asembly="A2")  # misspelled "assembly" again!
```

Often we'll want to access our field names programmatically. Sadly, this also
requires accessing a private `namedtuple` attribute. Say we're writing
`namedtuple`s to a CSV file:
```
csvwriter.write_header(Part._fields)
```

:toilet: Gross! Another private attribute!


## How about regular ol' `Enum`s as keys?
`Enum` makes your code more debugable. When you use `Enum` members as keys
and parameters in your project, you never again have to wonder where literal
strings like 'asembly' came from in a `KeyError` traceback. They're created
in a clean, declarative fashion and they're immutable.

```
Part = Enum("Part", "assembly reference subassembly name")
part = {Part.assembly: "A1", Part.name: "resistor", ...}
```

At this point we have a `part` dictionary whose keys are easily debuggable
`Enum` members. The problem is that you lose expressiveness when you create
collections out of their members:

```
part.update({Part.assembly: "A2"})  # Enum members can't be **kwarg keys!
part[Part.assembly]  # lots of repetition and typing for something so simple
```

Our collection is no longer very REPL-friendly:
```
>>> part
{<Part.assembly: 0>: 'A2', <Part.subassembly: 1>: []...}
```

Also, you may eventually want to get your collections' keys back into plain
string form (say, for JSONifying them):
```
jsonifyable_part = {key.name: value for key, value in part.items()}
```

... and nobody has time for that.


## `Enumap`: expressive *and* strict
With `Enumap`, you get an immutable collection of keys from which you can
create `dict`s and `namedtuple`s. This approach gives you the best of both
worlds: expressive, familiar data structures constructed by the same
object that holds the keys/field names.

```
Part = Enumap("Part", "assembly reference subassembly name")
part = Part.map("A1", "R3", subassembly=[], name="resistor")
part_tuple = Part.tuple("A1", "R3", [], name="resistor")
```

If you use `Part` every time you want a new collection, you'll never let an
invalid key pass silently through your code:

```
new_part = Part.map(**part, assembly="A2")  # override assembly
new_part = Part.tuple(**part, assembly="A2")
```
