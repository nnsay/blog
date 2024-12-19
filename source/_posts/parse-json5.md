---
title: parse json5
date: 2024-12-19 14:02:28
tags:
  - Code
  - 研究
excerpt: learn and research json5
---

# 1. What's [JSON5](https://spec.json5.org)

JSON5 is an extension to the popular [JSON](https://tools.ietf.org/html/rfc7159) file format that aims to be easier to **write and maintain _by hand_ (e.g. for config files)**. It is _not intended_ to be used for machine-to-machine communication. (Keep using JSON or other file formats for that. 🙂)

Summary of Features:

- Objects

  - Object keys may be an ECMAScript 5.1 _[IdentifierName](https://www.ecma-international.org/ecma-262/5.1/#sec-7.6)_.
  - Objects may have a single trailing comma.

- Arrays

  - Arrays may have a single trailing comma.

- Strings

  - Strings may be single quoted.
  - Strings may span multiple lines by escaping new line characters.
  - Strings may include character escapes.

- Numbers

  - Numbers may be hexadecimal.
  - Numbers may have a leading or trailing decimal point.
  - Numbers may be [IEEE 754](http://ieeexplore.ieee.org/servlet/opac?punumber=4610933) positive infinity, negative infinity, and NaN.
  - Numbers may begin with an explicit plus sign.

- Comments

  - Single and multi-line comments are allowed.

- White Space

  - Additional white space characters are allowed.

> [!TIP]
>
> [jsonc](https://github.com/onury/jsonc) is another json extension. You can add comments in jsonc file. This kind of file have their own file extension: .jsonc

# 2. How to parse json5 file?

In this blog, we only pay attention to parse json5 with javascript/nodejs language. The [json5](https://github.com/json5/json5) is the most popular NPM package which do this job.

Test the `json5` with a Typescript script and some test config files. The test code is as follows:

```typescript
import json5 from "json5";
import path from "path";
import { readFile } from "fs/promises";

const json5PackageTest = async () => {
  // json5 package
  const configFiles = ["config.json5", "config.jsonc"];
  for (const config of configFiles) {
    const json = await readFile(path.resolve(__dirname, config), "utf-8");
    const jsonObj = json5.parse<{ name: string; email: string }>(json);
    console.table(jsonObj);
  }
};

json5PackageTest().catch((err) => {
  console.error("catch error: ", err);
});
```

`config.json5`

```json
{
  // comments
  "unquoted": "and you can quote me on that",
  "singleQuotes": "I can use \"double quotes\" here",
  "lineBreaks": "Look, Mom! \
No \\n's!",
  "hexadecimal": 0xdecaf,
  "leadingDecimalPoint": 0.8675309,
  "andTrailing": 8675309,
  "positiveSign": +1,
  "trailingComma": "in objects",
  "andIn": ["arrays"],
  "backwardsCompatible": "with JSON"
}
```

`config.jsonc`

```json
{
  // uniquentity
  "name": "Jimmy Wang",
  "email": "jw@mail.com"
}
```

Run test script and chek the output. The result looks good:

```
┌─────────────────────┬──────────┬──────────────────────────────────┐
│       (index)       │    0     │              Values              │
├─────────────────────┼──────────┼──────────────────────────────────┤
│      unquoted       │          │  'and you can quote me on that'  │
│    singleQuotes     │          │ 'I can use "double quotes" here' │
│     lineBreaks      │          │      "Look, Mom! No \\n's!"      │
│     hexadecimal     │          │              912559              │
│ leadingDecimalPoint │          │            0.8675309             │
│     andTrailing     │          │             8675309              │
│    positiveSign     │          │                1                 │
│    trailingComma    │          │           'in objects'           │
│        andIn        │ 'arrays' │                                  │
│ backwardsCompatible │          │           'with JSON'            │
└─────────────────────┴──────────┴──────────────────────────────────┘
┌─────────┬────────────────────────┐
│ (index) │         Values         │
├─────────┼────────────────────────┤
│  name   │      'Jimmy Wang'      │
│  email  │ 'jimmy.wang@gmail.com' │
└─────────┴────────────────────────
```

# 3. Think more

In the [JSON5 specification](https://spec.json5.org/), the ECMAScript 5.1 features have been extended to JSON5. In antoher words, this tells us that the json5 config content is a javascript object. If we can export the object safely, the parse problem will been resolved.

The first tool that comes to my mind is the NodeJS core library [VM](https://nodejs.org/docs/latest/api/vm.html), it provides a sandbox for run unsafe code. We can run the json5 string in vm sandbox environment and export the object with vm context.

`json5.ts`

```typescript
import { readFileSync } from "node:fs";
import path from "node:path";
import vm from "node:vm";

export class JSON5 {
  static parse<T>(json: string) {
    const extName = path.extname(json);
    if (extName === ".json5" || extName === ".jsonc" || extName === ".json") {
      json = readFileSync(json, "utf-8");
    }
    const ctx = { __json5__: undefined };
    vm.runInContext(`__json5__=${json}`, vm.createContext(ctx));
    return ctx.__json5__ as T;
  }
}
```

Test code:

```typescript
import json5 from "json5";
import path from "path";
import { readFile } from "fs/promises";
import { JSON5 } from "./json5";

const json5Class = () => {
  // parse file: .jsonc or.json5

  const configFiles = ["config.json5", "config.jsonc"];
  for (const config of configFiles) {
    const jsonObj = JSON5.parse<{ name: string; email: string }>(
      path.resolve(__dirname, config)
    );
    console.log(`parse json file[${config}] result:`);
    console.log(jsonObj);
  }
  // parse json string
  console.log("parse json string result:");
  const jsonObj = JSON5.parse('{name: "json5", email: "json5@json5.com"}');
  console.log(jsonObj);
  console.log("parse json(array) string result:");
  const jsonArr = JSON5.parse(
    '[{name: "json5", email: "json5@json5.com"}, {name: "json5", email: "json5@json5.com"}]'
  );
  console.table(jsonArr);
};

json5Class().catch((err) => {
  console.error("catch error: ", err);
});
```

Run the above test code and check the outputs:

```
parse json file[config.json5] result:
{
  unquoted: 'and you can quote me on that',
  singleQuotes: 'I can use "double quotes" here',
  lineBreaks: "Look, Mom! No \\n's!",
  hexadecimal: 912559,
  leadingDecimalPoint: 0.8675309,
  andTrailing: 8675309,
  positiveSign: 1,
  trailingComma: 'in objects',
  andIn: [ 'arrays' ],
  backwardsCompatible: 'with JSON'
}
parse json file[config.jsonc] result:
{ name: 'Jimmy Wang', email: 'jimmy.wang@gmail.com' }
parse json string result:
{ name: 'json5', email: 'json5@json5.com' }
parse json(array) string result:
┌─────────┬─────────┬───────────────────┐
│ (index) │  name   │       email       │
├─────────┼─────────┼───────────────────┤
│    0    │ 'json5' │ 'json5@json5.com' │
│    1    │ 'json5' │ 'json5@json5.com' │
└─────────┴─────────┴───────────────────┘
```

The result looks good. So the vm way is simple solution for parse json5. Compared to [json5](https://github.com/json5/json5) package, export the javascript object with vm core pacakge is more simple. Applying more ECMAScript feature in json5 config also will be ok if you parse with the vm parse solution.

# 4. Summary

JSON5 make json config easier for humans to write and maintain by hand. For json5 config file, you can parse it by json5 npm pacakge or custom solution. The vm way which is tested in chapter 3 is a good custom solution for parse json5. You can copy/paste the `json5.ts` file in your project and import `JSON5` class to test and check.
