---
title: "tsconfig/bases ใฎ็ดนไป๏ผ"
emoji: "๐ช"
type: "tech"
topics: ["typescript"]
published: true
---

## ็ต่ซ

```sh
$ yarn add -D @tsconfig/strictest
```

```json:tsconfig.json
{
    "extends": "@tsconfig/strictest/tsconfig.json",
}
```

ไปฅไธใงใ๏ผ

## `tsconfig.json` ใฃใฆใฉใใชใตใใซๆธใใฆใใพใใ๏ผ๏ผ

`tsconfig.json` ใใใใชๆใใงๆธใใฆใใไบบใฏใใชใใงใใใใ

```json:tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "allowUnusedLabels": false,
        "allowUnreachableCode": false,
        "exactOptionalPropertyTypes": true,
        "noFallthroughCasesInSwitch": true,
        "noImplicitOverride": true,
        "noImplicitReturns": true,
        "noPropertyAccessFromIndexSignature": true,
        "noUncheckedIndexedAccess": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "lib": ["dom"],
        "outDir": "./dist",
        // ...ไปฅไธ็ฅ
    }
}
```

ใใใฆใใใใ่คๆฐใฎใใญใธใงใฏใใงใณใใใใฆไฝฟใๅใใใใใใ

ใพใใๅจ็ถๅฅใซใใใจๆใใใงใใใใใใชใใฎใใใใ๏ผใฃใฆใใใฎใ็ดนไปใใใฆใใ ใใ๏ผ

## tsconfig/bases ใง `tsconfig.json` ใ็ฐกๆฝใซใใ

็ตๅฑๅใใฃใฆ `"strict": true` ใซใใใใ `"noUncheckedIndexedAccess": true` ใซใใใใใชใใงใใใใใฃใจใใใจๅใๅณใใใชใใฐใชใใปใฉๅใใใใใ M ใใใชใใงใใ๏ผ๏ผ

ใใใใใใจใใใใๅ ใใใฆใใใใ๏ผใใฃใฆใใ่จญๅฎใใใใฉใซใใซใใฆใใใไพฟๅฉใชใใคใใใใใงใใใ

https://github.com/tsconfig/bases

ใใใคใงใ๏ผ

็ดฐใใใใฎใซใคใใฆใฏ [README](https://github.com/tsconfig/bases#readme) ใ่ฆใฆใใใใใฎใ่ฏใใฎใงใใใ [@tsconfig/strictest](https://www.npmjs.com/package/@tsconfig/strictest) ใไฝฟใใจใๅใใงใใฏใๅ ใใใใชใใทใงใณใๅจ้จไนใใฆใใใใใงใใใญใ

ไฝฟใๆนใฏๆฎ้ใซใคใณในใใผใซใใฆใ

```sh
$ yarn add -D @tsconfig/strictest
```

`tsconfig.json` ใฎ `extends` ใซๆๅฎใใใ ใใงใ

```json:tsconfig.json
{
    "extends": "@tsconfig/strictest/tsconfig.json",
}
```

ใใฃใกใ็ฐกๅใงใใ๏ผ

ใใใๆๅฎใใใจใไปฅไธใฎๅๅฎนใ้ฉ็จใใใใใใซใชใใใงใใ

https://github.com/tsconfig/bases/blob/27f9097a02f960e351d69bca8dd4c7581d70e1b6/bases/strictest.json

ๅใใฎใใผใ ใซใฏๅณใใใใใชใใฃใฆใใใซใผใซใใใใฐใใใใ ใใๆ็คบ็ใซใชใใซใใฆใใใใฐ่ฏใใใงใใใญใ

ใจใใใใจใงใใใฒไฝฟใฃใฆใฟใฆใใ ใใ๏ผ

## ็ชๅค็ทจ: "importsNotUsedAsValues": "error"

`strictest` ใซใฏ `"importsNotUsedAsValues": "error"` ใจใใใชใใทใงใณใใใใใงใใใใใใฏๅๆๅ ฑใฎใใใ ใใซ `import` ใใฆใใๅ ดๅใซ `import type` ใจใใฆ `import` ใใใชใใใฐใชใใชใใจใใใซใผใซใงใใๅทไฝ็ใซ่จใใจใ

```ts
// ใใใฏใจใฉใผใซใชใ
import { Foo } from "./foo";

const foo: Foo = {};

// ใใฎๅ ดๅๆญฃใใใฎใฏ
import type { Foo } from "./foo";
```

```ts
// ใใใฏใจใฉใผใซใชใใชใ
import { Foo } from "./foo";

const foo = new Foo();
```

ใจใใๆใใชใฎใงใใใใใกใใกๆๅใง็ฎก็ใชใใใใฃใฆใใใชใใงใใใญใๅฎใฏ `@typescript-eslint/eslint-plugin` ใฎใซใผใซใซใใใ่ชๅใงไฟฎๆญฃใใฆใใใใซใผใซใใใใฎใงใใใใ่จญๅฎใใใจ่ฏใใงใใ

```json:.eslintrc
{
    "rules": {
        "@typescript-eslint/consistent-type-imports": "error",
    }
}
```

ใพใใใใฎใซใผใซใฏ `@typescript-eslint/recommended` ใจใใ่จญๅฎใซใๅซใพใใฆใใใฎใงใไธ่จใฎใใใซๆๅฎใใใ

```json:.eslintrc
{
    "extends": ["plugin:@typescript-eslint/recommended"]
}
```

ใฎใใใซใใฆใๅคงไธๅคซใงใใ
