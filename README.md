# <span><img src="https://github.githubassets.com/images/icons/emoji/unicode/1f4d6.png" alt="Recursive DNS search"/></span> Practical tips for using io-ts

**Intro**

I have been working with [fp-ts ecosystem](https://gcanti.github.io/fp-ts/ecosystem) for a a couple of years and using it for several different projects. While working on different projects I have found some common patterns I would like to share in this document. This document is a collection of principles
and rules that have proven to be effective for me and the teams I have worked with.

I would like to outline patterns about:
* **General Patterns**
* **Application Structure**
* **Testing**


Take everything here as an **opinion**, not as a dogma.


**Table of Contents of the main stable features (version < 2.2)**

- [**General Patterns**](#general-patterns)
  - [Implementation of Refinement types using Branded Types](#implementation-of-refinement-types-using-branded-types)
    - [Definition of refinement type using io-ts types](#definition-of-refinement-type-using-io-ts-types)
    - [Refinement type must follow the rules](#refinement-type-must-follow-the-rules)
      - [Smart constructors](#smart-constructors) 
        - [General definition of smart constructor](#general-definition-of-smart-constructor)
        - [Definition of smart constructor from io-ts](#definition-of-smart-constructor-from-io-ts)
     - [Patterns for writting Branded types](#patterns-for-writting-branded-types)
        - [Use Branded types to define Domain types](#use-branded-types-to-define-domain-types)
        - [Initialization of Branded types](#initialization-of-branded-types)
        - [Export only smart constructor of Branded type](#use-branded-types-to-define-domain-types)
<!--   - Import Branded type after **[release 1.8.1](https://github.com/gcanti/io-ts/releases/tag/1.8.1)** -->

- [**Application Structure**](#application-structure)
  - [Use Default import to import io-ts](#use-default-import-to-import-io-ts)

**Table of Contents of the experimental features (version >= 2.2)**
- [**Examples that use experimental API**](#examples-that-use-experimental-api)
  
  
  
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
import { Either, fromPredicate as eitherFromPredicate, toError } from "fp-ts/lib/Either";
import { Branded } from "io-ts";

interface TrimmedStringBrand {
  readonly TrimmedString: unique symbol; 
}

type TrimmedString = Branded<string, TrimmedStringBrand>;

type Effect<S, E = Error> = Either<E, S>;

const isTrimmedString = (s: string): s is TrimmedString => s.length === s.trim().length;
const createTrimmedString = (s: string): Effect<TrimmedString> => pipe(s, eitherFromPredicate(isTrimmedString, toError));
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

#### Use Branded types to define Domain types
The point of having **Branded types** is to avoid a set of compile-time errors. You can combine **Branded types** via [combinators](https://github.com/gcanti/io-ts/blob/master/index.md#implemented-types--combinators) to build more complex types which represent domain entities, infrastructure types such as payloads.

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

# Examples that use experimental API
