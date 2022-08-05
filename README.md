# <span><img src="https://github.githubassets.com/images/icons/emoji/unicode/1f4d6.png" alt="Recursive DNS search"/></span> Practical tips for using io-ts

**Intro**

I have been working with [fp-ts ecosystem](https://gcanti.github.io/fp-ts/ecosystem) for a a couple of years and using it for several different projects. While working on different projects I have found some common patterns I would like to share in this document. This document is a collection of principles
and rules that have proven to be effective for me and the teams I have worked with.

I would like to outline patterns about:
* **Application Structure**
* **General Patterns**
* **Testing**


Take everything here as an **opinion**, not as a dogma.


**Table of Contents**

- [**Application Structure**](#application-structure)
  - [Use Default import to import io-ts](#use-default-import-to-import-io-ts)
  - [Declare static types before implementation](#declare-static-types-before-implementation)
    
- [**General Patterns**](#general-patterns)
  - [Implementation of Refinement types using Branded Types](#implementation-of-refinement-types-using-branded-types)
    - [Definition of refinement type using io-ts types](#definition-of-refinement-type-using-io-ts-types)
    - [Refinement type must follow the rules](#refinement-type-must-follow-the-rules)
      - [Smart constructors](#smart-constructors) 
        - [General definition of smart constructor](#general-definition-of-smart-constructor)
        - [Definition of smart constructor from io-ts](#definition-of-smart-constructor-from-io-ts)
<!--   - Import Branded type after **[release 1.8.1](https://github.com/gcanti/io-ts/releases/tag/1.8.1)** -->

# Application Structure

## Use Default import to import io-ts
  Consider the code below:
```ts
import * as t from 'io-ts';

import { NanoId} from "types";

const ClientId = t.intersection([
  t.string,
  NanoId
], 'ClientId');

```
While **Namespace import** works, it is confusing and not a god practise to import all the contents of lib if we are not using them.
A better approach is to use **Default export** as seen below:
```ts
import {intersection, string} from 'io-ts';

import { NanoId} from "types";

const ClientId = intersection([
  string,
  NanoId
], 'ClientId');

```
With this approach we can destructure the content we need from the io-ts module instead of importing all the contents.

## Declare static types before implementation
Consider the code below:
```ts
import {
  TypeOf,
  OutputOf,
  intersection,
  string,
  Branded,
  brand,
} from "io-ts";

import { NanoId, isURL } from "types";


const ClientId = intersection([string, NanoId], "ClientId");
type ClientId = TypeOf<typeof ClientId>;


interface DomainBrand {
  readonly Domain: unique symbol;
}

const Domain = brand(
  string,
  (a): a is Branded<string, DomainBrand> => isURL(a),
  "Domain"
);

type Domain = TypeOf<typeof Domain>;
```
The code above can be cleaner and more readable especially when you define Branded types using interface with unique Symbol. If we seperate the runtime and compile-time declarations, it makes the code much easier to read from top to bottom without jumping and keeps the declarations and definitions seperate well: 
```ts
import {
  TypeOf,
  OutputOf,
  intersection,
  string,
  Branded,
  brand,
} from "io-ts";

import { NanoId, isURL } from "types";

type ClientId = TypeOf<typeof ClientId>;

const ClientId = intersection([string, NanoId], "ClientId");

interface DomainBrand {
  readonly Domain: unique symbol;
}

type Domain = TypeOf<typeof Domain>;

const Domain = brand(
  string,
  (a): a is Branded<string, DomainBrand> => isURL(a),
  "Domain"
);
```
We have seperated our compile-time declarations from our runtime declarations.

# General patterns

## Implementation of Refinement types using Branded Types

### Definition of refinement type using io-ts types
The refinement of a type `T` can be represented as `Branded<A, B>` where type `A` can be any member of the broad primitive types such as `string` or `number` and type `B` is defined as `interface Brand<B> {readonly _brand:  unique symbol B}` which represents the given refinement `R` and also ensures uniqueness.

Implementation of refinement type using Branded types: 
```ts
import { Branded } from "io-ts";

interface TrimmedStringBrand {
  readonly TrimmedString: unique symbol; 
}

type TrimmedString = Branded<string, TrimmedStringBrand>;
```
Type `Branded<A, B>` is **type constructor** which takes types as arguments and returns another type.

| **parameter of type constructor** | concrete type |
| ------------- | ------------------------------- |
| `A` | `string` - primitive data type |
| `B` | `TrimmedString` - refinement of primitive data type |

### Refinement type must follow the rules
If we return to the previous example of `TrimmedString`, this `TrimmedString`  only exists at type-level and we need to make sure that the given string does not containt whitespaces. We need some runtime validations. This is where **smart constructor** comes into play.

#### **Smart constructors** 
Smart constructors are functions that perform some extra checks when the values of required type are constructed.

##### General definition of smart constructor

| **signature of smart constructor** | |
| ------------- | ------------------------------- |
| `(a: A) => F<R>` |  Such signature models a function which accepts an input of the `A` and returns the type of `R`, coupled with an **effect** `F`, where `F` is some **type constructor**. |

Example: 
```ts
import { Either, fromPredicate as EitherFromPredicate, toError } from "fp-ts/lib/Either";
import { Branded } from "io-ts";

interface TrimmedStringBrand {
  readonly TrimmedString: unique symbol; 
}

type TrimmedString = Branded<string, TrimmedStringBrand>;

type Effect<S, E = Error> = Either<E, S>;

const isTrimmedString = (s: string): s is TrimmedString => s.length === s.trim().length;
const createTrimmedString = (s: string): Effect<TrimmedString> => pipe(s, EitherFromPredicate(isTrimmedString, toError));
```

| helpers | description |
| ------------- | ------------------------------- |
| isTrimmedString | Custom type guard to narrow the type - `TrimmedString` |
| createTrimmedString | **Smart constructor** |
| Effect | represents computation that can fail |




##### Definition of smart constructor from io-ts

| **signature of smart constructor from io-ts** | description |
| ------------- | ------------------------------- |
| `<I, A> = (i: I) => Validation<A>` |  Such signature models a function which accepts an input of the `I` and returns the `Validation` **effect**, coupled with type `A` which represents encapsulated **Branded type**. In our case, it is `TrimmedString`. |

All of the codecs([combinators](https://github.com/gcanti/io-ts/blob/master/index.md#implemented-types--combinators)) inherit from `Type<A, O, I>` 
class:
```ts
class Type<A, O = A, I = unknown> implements Decoder<I, A>, Encoder<A, O> {

  readonly is: Is<A>

  constructor(...)
  decode(i: I): Validation<A>
  ...
}
```

You could then define a codec `TrimmedString` which represents a string without leading and trailing whitespaces:
```ts
import {
  Either,
  fromPredicate as eitherFromPredicate,
  toError,
  mapLeft as eitherMapLeft,
} from "fp-ts/lib/Either";

interface TrimmedStringBrand {
  readonly TrimmedString: unique symbol;
}

type TrimmedString = Branded<string, TrimmedStringBrand>;

type Effect<E, S> = Either<E, S>;

type ValidationContext = { value: unknown; context: Context };

const isTrimmedString = (s: unknown): s is TrimmedString =>
  typeof s === "string" && s.length === s.trim().length;
const createTrimmedString = (s: string): Effect<Error, TrimmedString> =>
  pipe(s, eitherFromPredicate(isTrimmedString, toError));

const mapErrorToIOErrors =
  ({ value, context }: ValidationContext) =>
  ({ message }: Error): Errors =>
    [{ value, context, message }];

const TrimmedString = new Type<TrimmedString, string, string>(
  "TrimmedString",
  isTrimmedString,
  (value, context) =>
    pipe(
      value,
      createTrimmedString,
      eitherMapLeft(mapErrorToIOErrors({ value, context }))
    ),
  identity
);
```

| API **`Type<A, O, I>`** | description |
| ------------- | ------------------------------- |
| `is: Is<A>` |  Custom type guard to narrow the appropriate type -`isTrimmedString` |
| `decode(i: I): Validation<A>` |  This method can be used as **Smart guard** - `createTrimmedString` |



    


