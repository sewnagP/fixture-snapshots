<h1 align="center">
  <!-- Logo -->
  <br/>
  @marko/fixture-snapshots
	<br/>

  <!-- Stability -->
  <a href="https://nodejs.org/api/documentation.html#documentation_stability_index">
    <img src="https://img.shields.io/badge/stability-stable-brightgreen.svg" alt="API Stability"/>
  </a>
  <!-- Language -->
  <a href="http://typescriptlang.org">
    <img src="https://img.shields.io/badge/%3C%2F%3E-typescript-blue.svg" alt="TypeScript"/>
  </a>
  <!-- Format -->
  <a href="https://github.com/prettier/prettier">
    <img src="https://img.shields.io/badge/styled_with-prettier-ff69b4.svg" alt="Styled with prettier"/>
  </a>
  <!-- CI -->
  <a href="https://travis-ci.org/marko-js/fixture-snapshots">
  <img src="https://img.shields.io/travis/marko-js/fixture-snapshots.svg" alt="Build status"/>
  </a>
  <!-- NPM Version -->
  <a href="https://npmjs.org/package/@marko/fixture-snapshots">
    <img src="https://img.shields.io/npm/v/@marko/fixture-snapshots.svg" alt="NPM Version"/>
  </a>
  <!-- Downloads -->
  <a href="https://npmjs.org/package/@marko/fixture-snapshots">
    <img src="https://img.shields.io/npm/dm/@marko/fixture-snapshots.svg" alt="Downloads"/>
  </a>
  <!-- Size -->
  <a href="https://npmjs.org/package/@marko/fixture-snapshots">
    <img src="https://img.shields.io/badge/size-1.21kb-green.svg" alt="Browser Bundle Size"/>
  </a>
</h1>

A tool for working with Marko component fixtures given the following directory structure:

```bash
src/
  components/
    app-carousel/
      fixtures/
        fixture1.js # exports data to render app-carousel/index.marko with
        fixture2.json # data to render app-carousel/index.marko with
        fixture3.marko # a template assumed to use <app-carousel>
      index.marko
```

## Installation

You probably already have `marko` installed, but you'll also need [`@marko/testing-library`](https://github.com/marko-js/testing-library) since it is used to render the fixtures.

```console
npm install marko @marko/fixture-snapshots @marko/testing-library
```

## Automatic Snapshot API

The automatic snapshot API can be used to generate and assert snapshots for the fixtures at a component level or for your entire project. The snapshots will be stored as `.html` files next to the fixture:

```bash
fixtures/
  fixture1.js
  fixture1.html # snapshot of app-carousel/index.marko rendered with data from fixture1.js
  fixture2.json
  fixture2.html # snapshot of app-carousel/index.marko rendered with data from fixture2.json
  fixture3.marko
  fixture3.html # snapshot of fixture3.marko
```

### `runProjectSnapshotTests(path: string, options?: SnapshotOptions)`

Loads the fixtures for all components under `path` and generates tests that render them and compare/generate snapshots.

```typescript
type SnapshotOptions = {
  normalizer: (container: Element | Fragment) => Element | Fragment;
  // a function the recieves a DOM container and returns a normalized DOM tree.
  // The normalizer function should not mutate the existing tree
  // (default: `require("@marko/fixture-snapshots").defaultNormalizer`)

  ignore: string[]; // directories to not search for fixtures in (default: ["node_modules"])
  fixtureDir: string; // the name of the fixture directory to search for (default: "fixtures")
};
```

#### Usage with Jest

```javascript
import path from "path";
import { runProjectSnapshotTests } from "@marko/fixture-snapshots/jest";
// const { runProjectSnapshotTests } = require("@marko/fixture-snapshots/jest");

describe("fixture snapshots", () => {
  // run snapshot tests for all components under src
  runProjectSnapshotTests(path.join(__dirname, "./src"));
});
```

> You can use `jest`'s built-in [snapshot updating](https://jestjs.io/docs/en/snapshot-testing#updating-snapshots) (`jest -u`) to update the fixture snapshots

#### Usage with Mocha

```javascript
import path from "path";
import { runProjectSnapshotTests } from "@marko/fixture-snapshots/mocha";
// const { runProjectSnapshotTests } = require("@marko/fixture-snapshots/mocha");

describe("fixture snapshots", () => {
  // run snapshot tests for all components under src
  runProjectSnapshotTests(path.join(__dirname, "./src"));
});
```

> You can set `UPDATE_SNAPSHOTS` as an [environment variable](https://en.wikipedia.org/wiki/Environment_variable) (`UPDATE_SNAPSHOTS=true mocha`) to update the fixture snapshots

### `runComponentSnapshotTests(options?: SnapshotOptions)`

Loads the fixtures for the closest component to the test file and generates tests that render it and compare/generate snapshots:

#### Usage with Jest/Mocha

```javascript
import { runComponentSnapshotTests } from "@marko/fixture-snapshots/jest"; // or mocha

describe("app-carousel", () => {
  runComponentSnapshotTests();

  it("another test", () => { ... });
});
```

## API

```js
import {
  findComponentFixtures,
  findClosestComponentFixtures,
  findProjectFixtures,
  defaultSerializer,
  defaultNormalizer
} from "@marko/fixture-snapshots";
```

### `findComponentFixtures(file: string, options?: ComponentOptions): ComponentFixtures`

Loads the fixtures for the component at `file`.

```typescript
type ComponentOptions = {
  fixtureDir: string; // the name of the fixture directory to search for (default: "fixtures")
};

type ComponentFixtures = {
  name: string; // the inferred name of the component (ex. app-carousel)
  path: string; // the absolute path to the component
  component: Template; // the loaded Marko template
  fixturesPath: string;
  fixtures: Record<
    string, // the inferred name of the fixture (ex. data)
    {
      name: string;
      path: string; // the absolute path to the fixture
      ext: ".js" | ".json" | ".marko";
      fixture: object | Template; // the loaded fixture
      render: () => RenderResult; // render the fixture, return type is the same as `@marko/testing-library`'s render function
      toString: (normalizer = defaultNormalizer) => Promise<string>;
    }
  >;
};
```

#### Example

```js
import { fireEvent } from "@marko/testing-library";
import { findComponentFixtures } from "@marko/fixture-snapshots";
const { fixtures } = findComponentFixtures(require.resolve("../index.marko"));

test("example", () => {
  const result = await fixtures.example.render();
  const button = result.getByRole("button");
  await fireEvent.click(button);
  expect(result.emitted("clicky-click")).toHaveLength(1);
});
```

### `findClosestComponentFixtures(options?: ClosestComponentOptions): ComponentFixtures`

Automatically finds and loads the fixtures for the closest component. Searches in the `cwd` and checks if `fixtureDir` exists, if not, it goes to the parent directory and checks. Once the `fixtureDir` is found, it begins searching in the directory in which it was found for an `index.marko` or `template.marko`, then the parent, and so on. The search will not leave the project root (the directory where the `package.json` lives).

```typescript
type ClosestComponentOptions = {
  cwd: string; // the directory to start searching from (default: call location)
  fixtureDir: string; // the name of the fixture directory to search for (default: "fixtures")
};
```

#### Example

```js
import { fireEvent } from "@marko/testing-library";
import { findClosestComponentFixtures } from "@marko/fixture-snapshots";
const { fixtures } = findClosestComponentFixtures();

test("example", () => {
  const result = await fixtures.example.render();
  const button = result.getByRole("button");
  await fireEvent.click(button);
  expect(result.emitted("clicky-click")).toHaveLength(1);
});
```

### `findProjectFixtures(dir: string, options?: ProjectOptions): ComponentFixtures[]`

Loads the fixtures for all components found under `dir`.

```typescript
type ProjectOptions = {
  ignore: string[]; // directories to not search for fixtures in (default: ["node_modules"])
  fixtureDir: string; // the name of the fixture directory to search for (default: "fixtures")
};
```

### `defaultSerializer(container: Element | Fragment): string`

Serializes the DOM container to a diffable HTML string

### `defaultNormalizer(container: Element | Fragment): Element | Fragment`

Returns a clone of the passed DOM container with Marko's internal markers removed (`data-marko`, etc.)

## Code of Conduct

This project adheres to the [eBay Code of Conduct](./.github/CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.
