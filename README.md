# Sedky
SEDKY (Self-documenting Expressive Data-format — Kinda like YAML, and yes, the acronym is a tortured version of my surname 🙃) is a human-readable, strongly typed superset of JSON. Inspired by Haskell’s Algebraic Data Types (ADTs), it’s designed to make defining complex, structured data both expressive and safe — without sacrificing readability. The format allows you to model rich, type-safe data structures that could, in principle, be directly decoded into native types in a host language such as Python, Haskell, or Rust.

The project began as TSON (Typed JSON), but the format gradually took on more influence from YAML than from its original JSON roots, leading to a name change. Some parts of the documentation still refer to it as TSON.

## Quick Introduction
This started from a small idea I had while working on a project: I wanted to store a Python class outside of the program and load it back in later. While `pickle` seemed like an obvious option, it came with serious safety concerns—mainly because it can execute arbitrary code when loading data. The safer alternative was JSON, but I quickly hit its limitations in terms of how expressive it could be.

That led me to think about the tradeoff between safety and expressiveness in data formats. Then I remembered a core idea from functional programming: pure functions with no side effects. It made me wonder—could a data format based on functional principles be both expressive and safe?

That line of thinking eventually led me to combine ideas from Haskell and JSON, and that’s how I arrived at the idea behind TSON.

### Just a Note
This was something I put together out of curiosity one afternoon. I haven’t deeply researched whether similar formats already exist, and I didn’t set out to copy anyone’s work—just to explore an idea I found interesting. I’m a maths student who enjoys writing down ideas like this for fun.

At present this is just an idea, I have not built anything. Am open to any critisms or areas of improvment if you any other person has actually bothered to read this.

ChatGPT helped tidy up the grammar and writing in this README.

## Design Objectives
1. *JSON-Compatible*: SEDKY is designed as a superset of JSON—all valid JSON files are also valid TSON files.

2. *Readable*: SEDKY prioritizes human readability. Files should be easy to understand and manually edit.

3. *Expressive*: The format supports rich and structured data, making it easy to represent complex information.

4. *Strong Guarantees*: The use of strong, explicit types allows TSON to offer clear guarantees about the structure and validity of data.

## Design
### Primitive Types
Like JSON, SEDKY includes a set of primitive types:

1. *Strings*
2. *Integers and Floats*
3. *Lists*
4. *Dictionaries*
5. *Optionals*

### ADTs
SEDKY supports Algebraic Data Types, inspired by Haskell. These are declared using the `type` keyword and can represent enums, records, and more.
```python
type bool = true | false
```

Parameterized types are also supported:
```python
type maybe a = nothing | just a
```
#### Simple Example
Here’s an example of a custom ADT used to model a user:
```python
type user = user {username: str, phone_number: optional str, email: str}

user(
  username: "sedryx-sedky"
  phone_number: null
  email: "sedryx.sedky@tson.dev"
)
```

Note how, just like in Haskell, even something as simple as `bool` can be defined as a sum type rather than relying on a built-in primitive.
#### Extended Example
Let’s say we want to give more structure to an email address rather than just storing it as a string:
```python
type email = email {name: str, provider: str, domain: str}
type user = user {username: str, phone_number: optional str, email: str}

user(
  username: "sedryx-sedky"
  phone_number: null
  email: email (
      name: "sedryx.sedky"
      provider: "tson"
      domain: "dev"
    )
)
```

While this gives us richer structure, it also becomes harder to read and write by hand. That’s where the next feature comes in.

### Pattern Matching on Strings
TSON supports pattern matching on strings to allow cleaner, more concise representations of structured types.

```python
type email = email {name: str, provider: str, domain: str}
type user = user {username: str, phonenumber: optional str, email: email}

"{a}@{b}.{c}" => email (name: a, provider: b, domain: c)

user(
  username: "sedryx-sedky"
  phonenumber: null
  email: "sedryx.sedky@tson.dev"
)
```

This allows us to keep structural guarantees and retain human-friendly syntax.
You can even add conditions with `where` clauses and built-in functions:

```python
type email = email {name: str, provider: str, domain: str}
type user = user {username: str, phonenumber: optional str, email: email}

"{a}@{b}.{c}" => email (name: a, provider: b, domain: c)
  where remove_whitespace(a) != ""

user(
  username: "sedryx-sedky"
  phonenumber: null
  email: "sedryx.sedky@tson.dev"
)
```

## Syntax
### Algebratic Data Types (ADTs)
Algebraic Data Types (ADTs) can be defined using either *pipe-separated* or *newline-separated* constructors.
*Syntax (Pipe Style)*
```python
type <new_type> = <constructor_1> {<field>: <type>, ...} 
               | <constructor_2> {<field>: <type>, ...} 
               | ...
```

*Syntax (Newline Style)*
```python
type <new_type> =
  <constructor_1> {<field>: <type>, ...}
  <constructor_2> {<field>: <type>, ...}
  ...
```

Both forms are equivalent. The pipe (`|`) syntax serves as syntactic sugar for newline-separated declarations. It is more concise for simple types, whereas the newline form provides improved readability, particularly when constructors span multiple lines or include `where` clauses.

*Example*
```python
# Pipe Style
type result = success {data: str} | failure {error: str}

# Newline Style
type result =
  success {data: str}
  failure {error: str}
```

A `where` clause can be used to impose constraints on constructor fields. These constraints may apply to individual constructors or to groups of constructors using the `for` and `except` keywords.
```python
type payment_method =
  cash {amount: money}
  check {amount: money, check_number: str, bank_routing: str}
  credit_card {amount: money, card_number: str, cvv: str, expiry: date}
  debit_card {amount: money, card_number: str, pin: str}
  gift_card {amount: money, card_code: str}
  store_credit {amount: money, credit_id: str}
  crypto {amount: money, wallet_address: str, transaction_hash: str}

    where amount > 0.99 except crypto
    where
      len(card_number) == 16
      len(cvv) == 3
    where amount <= 10_000 for cash
```

In the example above:

* The constraint `amount > 0.99` applies to all constructors except crypto.

* The constraint `amount <= 10_000` applies only to the cash constructor.

* The constraints on `card_number` and `cvv` apply to any constructor that defines those fields.

Each `where` clause is evaluated in the order it appears. If a constraint refers to a field that does *not* exist in a constructor, it is treated as vacuously true — the check passes and evaluation continues. This avoids unnecessary errors for constructors that do not contain certain fields.

`where` clauses support boolean operations such as `and`, `or`, and `not`. Multiple lines within a single where clause are implicitly joined with `and`.

You may also attach a `where` clause directly to an individual constructor:
```python
type example = foo {x: int} where x > 4 | bar {y: int}
```
This is simply syntactic sugar for:
```python
type example = foo {x: int} | bar {y: int}
  where x > 4 for foo
```
The attached `where` clause is treated as a scoped constraint on that constructor alone. This provides a concise way to declare simple, constructor-specific rules inline.

As in Haskell, type names and constructor names reside in separate namespaces. Field names are optional, and parameterized types are fully supported:

```python
type maybe a = nothing | just a
```
Constructors may be instantiated using commas, line breaks, or a combination of both:

```python
standard (days: 5)
```
For types with a single constructor, a concise anonymous constructor syntax is permitted:

```python
{<field>: <value>, ...}
```
*Example*

```python
type person = person {name: str, dob: str, email: optional str}
type book = book {title: str, price: float, author: person} where price > 0

book (
  title: "A Theory of Typed Objects",
  price: 24.99,
  author: {name: "Ada Lovelace", dob: "1815-12-10", email: nil}
)
```

The system will infer the correct type of the anonymous constructor (`person` in this case) and wrap it accordingly. Additional syntactic sugar is available to provide a more YAML-like syntax.
### References
TSON supports named references to improve modularity and express complex or recursive data structures.
```python
<name> := <data>   # Define a reference
!<name>            # Use '!' to reference the named value
```
This mechanism enables shared substructures and cyclic relationships.
*Example*
```python
type person = person {
  name: str,
  father: optional person,
  mother: optional person,
  children: list person
}

homer := person(
  name: "Homer"
  father: nil
  mother: nil
  children: [!bart, !lisa, !maggie]
)

marge := person(
  name: "Marge"
  father: nil
  mother: nil
  children: !homer.children
)

bart := person(
  name: "Bart"
  father: !homer
  mother: !marge
  children: []
)

lisa := person(
  name: "Lisa"
  father: !homer
  mother: !marge
  children: []
)

maggie := person(
  name: "Maggie"
  father: !homer
  mother: !marge
  children: []
)

```
This referencing model avoids duplication, promotes clarity, and facilitates expressive representations of linked or hierarchical data.
### Syntactic Sugar
#### Anonymous Constructors
For single-constructor types, TSON provides a YAML-like shorthand to improve readability:
```python
type_constructor:
  field: value
  other_field: other_value
```
This is syntactic sugar for:
```python
type_constructor {field: value, other_field: other_value}
```
#### Lists
TSON supports a compact and expressive syntax for lists, particularly useful for long or nested arrays inspired by YAML.
```python
type operating_systems = operating_systems (list str)

operating_systems:
  - "macOS"
  - "Linux"
  - "Windows"
```
This desugars to:
```python
operating_systems (["macOS", "Linux", "Windows"])
```
This notation aims to enhance clarity, especially when working with large or nested lists.
#### Lists of Anonymous Constructors
When a constructor expects a list of structured records, TSON supports a compact and expressive syntax combining list and anonymous constructor notation.
*Example*
Suppose we define a person type and a group type that holds a list of person values:
```python
type person = person {name: str, age: int}
type group = group {people: list person}
```
This can be written compactly as:

```python
group:
  -name: "Alice"
   age: 30

  -name: "Bob"
   age: 25

  -name: "Charlie"
   age: 40
```

Which desugars to:
```python
group (
  people: [
    {name: "Alice", age: 30},
    {name: "Bob", age: 25},
    {name: "Charlie", age: 40}
  ]
)
```
Each - introduces a new anonymous constructor block, and all indented fields beneath it are grouped into the same record. This form provides a clean and YAML-like structure for expressing lists of values with multiple fields, while retaining the safety and structure of typed data.

## Example

Here’s what a real-world TSON data structure might look like:
```python
type money = money {amount: float, currency: str}

"${amount} {currency}" => money(amount: float(amount), currency: "USD")
"€{amount}" => money(amount: float(amount), currency: "EUR")
"£{amount}" => money(amount: float(amount), currency: "GBP")

type shipping_method = 
  standard { days: int }
  express { days: int, cost: money }
  overnight { cost: money }
    where days > 0

type payment_status = pending | processing | completed | failed

type order_item = order_item {
  product_id: str,
  name: str,
  quantity: int,
  unit_price: money
} where quantity > 0

"{quantity}x {name} ({product_id}) @ {price}" where int(quantity) > 0
  => order_item(
    product_id: product_id,
    name: name,
    quantity: int(quantity),
    unit_price: price
  )

type order = order {
  order_id: str,
  customer_email: str,
  items: list order_item,
  shipping: shipping_method,
  total: money,
  payment: payment_status,
  notes: optional str
}

# Sample orders
order(
  order_id: "ORD-2024-001"
  customer_email: "alice@example.com"
  items:
    - "2x Super Widget (WIDGET-123) @ $29.99"
    - "1x Magic Gadget (GADGET-456) @ €45.50"
  shipping: express(days = 2, cost = "$15.00")
  total: "$105.48"
  payment: completed
  notes: "Gift wrap requested"
)

order(
  order_id: "ORD-2024-002"
  customer_email: "bob@example.com"
  items:
    - "1x Programming Guide (BOOK-789) @ £24.99"
  shipping: standard(days = 5),
  total: "£24.99"
  payment: pending
  notes: null
)
```
## Comparision to YAML
Below is a comparison of the same data stored in YAML and TSON.
*TSON*
```python
type date = date {year: int, month: int, day: int}
"{year}-{month}-{day}" => date(year = year, month = month, day = day)

type diagnosis = diagnosis {
  code: str,
  description: str,
  confirmed: bool
}

type treatment =
  medication {name: str, dosage_mg: float, frequency_per_day: int}
  surgery {procedure: str, date: date}
  therapy {type: str, sessions: int}

type patient = patient {
  id: str,
  name: str,
  date_of_birth: date,
  diagnoses: list diagnosis,
  current_treatments: list treatment,
  allergies: list str
}

patient:
  id: "P123456"
  name: "John Doe"
  date_of_birth: "1980-02-15"
  diagnoses:
      - code: "I10"
        description: "Hypertension"
        confirmed: true
      - code: "E11"
        description: "Type 2 Diabetes"
        confirmed: true
  current_treatments:
    - medication:
        name: "Lisinopril"
        dosage_mg: 10.0
        frequency_per_day: 1
    - therapy:
        type: "Physical Therapy"
        sessions: 12
  allergies:
    - "Penicillin"
```

*YAML*
```yaml
id: P123456
name: John Doe
date_of_birth: 1980-02-15
diagnoses:
  - code: I10
    description: Hypertension
    confirmed: true
  - code: E11
    description: Type 2 Diabetes
    confirmed: true
current_treatments:
  - type: medication
    name: Lisinopril
    dosage_mg: 10.0
    frequency_per_day: 1
  - type: therapy
    type_of_therapy: Physical Therapy
    sessions: 12
allergies:
  - Penicillin

```
## Return Structures
TSON isn’t just about defining structured data—it’s also meant to return that structure in your host language in a clean and natural way.

Let’s revisit the earlier example. Suppose we store the following in a `.tson` file:

```python
type email = email {name: str, provider: str, domain: str}
type user = user {username: str, phonenumber: optional str, email: email}

"{a}@{b}.{c}" => email (name: a, provider: b, domain: c)
  where remove_whitespace(a) != ""

user(
  username: "sedryx-sedky",
  phonenumber: null,
  email: "sedryx.sedky@tson.dev"
)
```

Now, in Python, we could parse and use this structured data like so:

```python
import tson

with open('/path/to/file.tson', 'r') as file:
  users = tson.load(file.read())

user = users[0]

user.username # "hamed sedky"
user.phonenumber # None
user.email # email(name = "sedryx-sedky", provider = "tson", domain = "dev")
```

Under the hood, the `tson` parser would generate or return classes that match the structure defined in the TSON file. For example, the above could result in the following Python classes:
```python
class Email(TsonType): pass
class email(Email):
  name: str
  provider: str
  domain: str

class User(TsonType): pass
class user(User):
  username: str
  phonenumber: Optional[str]
  email: Email
```
