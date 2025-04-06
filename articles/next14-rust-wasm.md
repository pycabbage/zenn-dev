---
title: "Next.js 15でWebAssemblyを使う"
emoji: "🦀"
type: "tech"
topics: ["rust", "nextjs", "frontend"]
published: false
---


## はじめに

Next.jsでWebAssemblyを使う方法についての記事はいくつかありますが、Next.js 15では使えない方法が多かったため、メモとして残します。

## 使用したライブラリ/フレームワークのバージョン

| ライブラリ/フレームワーク | バージョン |
|---|---|
| Node.js | `v23.8.0` |
| Next.js | `15.2.4` |
| React | `19.1.0` |
| Rust | `1.85.0` |
| wasm-pack | `0.13.1` |

## 手順

### 1. プロジェクトの作成

```shell
# Next.jsプロジェクトの作成
corepack pnpm create next-app --ts --tailwind --no-eslint --app --src-dir --turbopack --use-pnpm nextjs-wasm-test

# Rust プロジェクトの作成
cd nextjs-wasm-test
wasm-pack new nextjs-wasm-test/wasm
```

### 2. Rustコードを記述

今回は、非常に簡単な処理の例として足し算を行う関数を記述します。

```rust
// wasm/src/lib.rs

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
  return a + b;
}
```

### 3. WebAssemblyのビルド / Next.jsに依存関係を追加

```shell
# WebAssemblyのビルド
wasm-pack build wasm -s nextjs-wasm-test
```

今回は依存関係を追加する方法として [pnpm Workspaces](https://pnpm.io/ja/workspaces) を使用します。
npm ( [npm workspaces](https://docs.npmjs.com/cli/v11/using-npm/workspaces) ) や yarn ( [yarn workspaces](https://yarnpkg.com/features/workspaces) ) でも代替できます。

```yaml
# pnpm-workspace.yaml
packages:
  - "wasm/pkg"
```

```jsonc
{
  // ...
  "devDependencies": {
    // ...
    "@nextjs-wasm-test/wasm": "workspace:^",
  }
}
```

### 4. WebAssemblyをNext.jsで使用

Client ComponentとServer Componentの両方で使用できることを確かめます。
まずは、Server Componentを記述します。

```tsx
// src/app/page.tsx

import Add from "@/components/Add"
import { add } from "@nextjs-wasm-test/wasm"

export default async function Home() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2 className="text-2xl">
        example: 2 + 3 = {add(2, 3)}
      </h2>
      <Add />
    </div>
  )
}
```

次に、Client Componentを記述します。

```tsx
// src/components/Add.tsx
"use client"

import { add } from "@nextjs-wasm-test/wasm"
import { useState } from "react"

const inputCN = "border border-gray-300 rounded-md p-2 outline-none not-read-only:focus:border-blue-500 not-read-only:focus:ring-1 not-read-only:focus:ring-blue-500 transition duration-200"
const buttonCN = "bg-blue-500 text-white rounded-md p-2 hover:bg-blue-600 transition duration-200 cursor-pointer"

export default function Add() {
  const initialValue = {
    a: 5,
    b: 7,
  }
  const [result, setResult] = useState(() => add(initialValue.a, initialValue.b))
  return <form className="flex flex-col gap-2" action={formData => {
    const {a, b} = Object.fromEntries(formData.entries().map(([key, value]) => [key, Number(value)])) as {
      a: number
      b: number
    }
    setResult(add(a, b))
  }}>
    <label>
      <span>First number</span>
      <input type="number" name="a" className={`${inputCN}`} defaultValue={initialValue.a} />
    </label>
    <label>
      <span>Second number</span>
      <input type="number" name="b" className={`${inputCN}`} defaultValue={initialValue.b} />
    </label>
    <label>
      <span>Result</span>
      <input type="number" name="c" readOnly={true} value={result} className={`${inputCN}`} />
    </label>
    <input type="submit" value="Add" className={`${buttonCN}`} />
  </form>
}
```

開発サーバーでの表示結果は以下の通りです。

![表示結果](/images/next14-rust-wasm/1.png)

### 5. Next.jsの設定

ここまでは問題なく動作しますが、Productionビルドを行うと、以下のエラーが発生します。

:::details エラー全文

```shell
$ corepack pnpm run next build --no-lint

> nextjs-wasm-test@0.1.0 next /home/ubuntu/nextjs-wasm-test
> next build --no-lint

 ⚠ Linting is disabled.
   ▲ Next.js 15.2.4

   Creating an optimized production build ...
socket hang up

Retrying 1/3...
Failed to compile.

./wasm/pkg/wasm_bg.wasm
Module parse failed: Unexpected character '' (1:0)
The module seem to be a WebAssembly module, but module is not flagged as WebAssembly module for webpack.
BREAKING CHANGE: Since webpack 5 WebAssembly is not enabled by default and flagged as experimental feature.
You need to enable one of the WebAssembly experiments via 'experiments.asyncWebAssembly: true' (based on async modules) or 'experiments.syncWebAssembly: true' (like webpack 4, deprecated).
For files that transpile to WebAssembly, make sure to set the module type in the 'module.rules' section of the config (e. g. 'type: "webassembly/async"').
(Source code omitted for this binary file)

Import trace for requested module:
./wasm/pkg/wasm_bg.wasm
./wasm/pkg/wasm.js
./src/app/page.tsx

./wasm/pkg/wasm_bg.wasm
Module parse failed: Unexpected character '' (1:0)
The module seem to be a WebAssembly module, but module is not flagged as WebAssembly module for webpack.
BREAKING CHANGE: Since webpack 5 WebAssembly is not enabled by default and flagged as experimental feature.
You need to enable one of the WebAssembly experiments via 'experiments.asyncWebAssembly: true' (based on async modules) or 'experiments.syncWebAssembly: true' (like webpack 4, deprecated).
For files that transpile to WebAssembly, make sure to set the module type in the 'module.rules' section of the config (e. g. 'type: "webassembly/async"').
(Source code omitted for this binary file)

Import trace for requested module:
./wasm/pkg/wasm_bg.wasm
./wasm/pkg/wasm.js
./src/components/Add.tsx

> Build failed because of webpack errors
 ELIFECYCLE  Command failed with exit code 1.
```

:::

Next.js が使用している webpack で WebAssembly を扱う場合、特殊な対処が必要なようです。
エラーの指示通りに `experiments.asyncWebAssembly` のみを有効化すると以下のエラーが発生します。

:::details エラー全文

```shell
$ corepack pnpm run next build --no-lint

> nextjs-wasm-test@0.1.0 next /home/ubuntu/nextjs-wasm-test
> next build --no-lint

 ⚠ Linting is disabled.
   ▲ Next.js 15.2.4

   Creating an optimized production build ...
 ⚠ Compiled with warnings

./wasm/pkg/wasm_bg.wasm
The generated code contains 'async/await' because this module is using "asyncWebAssembly".
However, your target environment does not appear to support 'async/await'.
As a result, the code may not run as expected or may cause runtime errors.

Import trace for requested module:
./wasm/pkg/wasm_bg.wasm
./wasm/pkg/wasm.js
./src/components/Add.tsx

 ✓ Checking validity of types    
   Collecting page data  ...[Error: Failed to collect configuration for /] {
  [cause]: [Error: ENOENT: no such file or directory, open '/home/ubuntu/nextjs-wasm-test/.next/server/static/wasm/ffe94b3915243d96.wasm'] {
    errno: -2,
    code: 'ENOENT',
    syscall: 'open',
    path: '/home/ubuntu/nextjs-wasm-test/.next/server/static/wasm/ffe94b3915243d96.wasm'
  }
}

> Build error occurred
[Error: Failed to collect page data for /] { type: 'Error' }
 ELIFECYCLE  Command failed with exit code 1.
```

:::

Next.js が WebAssembly ファイルのパスを正しく処理できないため、手動でシンボリックリンクを作成する必要があります。
そのため、`next.config.ts` に以下のような設定を追加します。

```ts
// next.config.ts

import type { NextConfig } from "next"
import { access, symlink } from "node:fs/promises"
import { join } from "node:path"
import type { Compiler, Configuration, WebpackPluginInstance } from "webpack"

const nextConfig = {
  webpack(config: Configuration, { isServer }) {
    config.experiments = {
      ...config.experiments,
      asyncWebAssembly: true,
      layers: true,
    }

    // define plugin
    class SymlinkWebpackPlugin implements WebpackPluginInstance {
      apply(compiler: Compiler) {
        compiler.hooks.afterEmit.tapPromise(
          "SymlinkWebpackPlugin",
          async (compiler) => {
            if (isServer) {
              const from = join(compiler.options.output.path || "", "../static")
              const to = join(compiler.options.output.path || "", "static")

              try {
                await access(from)
                return
              } catch (error: any) {
                if (error?.code !== "ENOENT") {
                  throw error
                }
              }

              await symlink(to, from, "junction")
              console.log(`created symlink ${from} -> ${to}`)
            }
          }
        )
      }
    }

    // add plugin
    if (!config.plugins) config.plugins = []
    config.plugins.push(new SymlinkWebpackPlugin())
    return config
  },
} satisfies NextConfig

export default nextConfig
```

### 6. Productionビルド・実行

```shell
# ビルド
corepack pnpm run build
# 実行
corepack pnpm run start
```

開発サーバーと同じ画面が表示できることを確認しました。

## まとめ

Next.js 15でWebAssemblyを使用する場合、変なバグを回避するためにいくつかの設定が必要です。
特にwebpack周りの設定に関しては不安定な部分が多く、次のメジャーリリースで追加の対応が必要になる可能性があります。
これらのバグはTurbopackでは確認できなかったため、開発環境では気付きにくいと感じました。

## 参考

https://bkbkb.net/ja/articles/nextjs-wasm

https://github.com/vercel/next.js/issues/34763

https://github.com/vercel/next.js/issues/72036
