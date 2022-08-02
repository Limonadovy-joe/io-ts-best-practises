# <span><img src="https://github.githubassets.com/images/icons/emoji/unicode/1f4d6.png" alt="Recursive DNS search"/></span> Practical tips for using io-ts

**Intro**

I have been working with [fp-ts ecosystem](https://gcanti.github.io/fp-ts/ecosystem) for a a couple of years and using it for several different projects. While working on different projects I have found some common patterns I would like to share in this document. This document is a collection of principles
and rules that have proven to be effective for me and the teams I have worked with.

I would like to outline patterns about:
* **Application Structure**
* **Testing**
* **General Patterns**

Take everything here as an **opinion**, not as a dogma.


**Table of Contents**

- **Application Structure**
  - [Use Default import to import io-ts](#use-default-import-to-import-io-ts)
  - [Declare static types before implementation](#declare-static-types-before-implementation)
  - Import Branded type after **[release 1.8.1](https://github.com/gcanti/io-ts/releases/tag/1.8.1)**


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
  type,
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
  type,
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
