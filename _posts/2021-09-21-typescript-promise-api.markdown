---
layout: post
title: "[Typescript] Promise API"
date: 2021-09-21T22:00:00-08:00
---

Typescript Promise API 不知道為什麼，只有提供 callback style 的 API ，對於用過 Scala Promise 的用戶來說。使用上很難用，可讀性又很糟糕

```typescript
return new Promise<number>((resolve, reject) => {
    try {
        //do something
        resolve(1)
    } catch (error) {
        reject(error)
    }
});
```

這樣子的寫法，把主要的邏輯包在 callback 內，可讀性不高，如果 promise 內又有 promise 就更難讀了。所以隨手寫了個 wrapper ，把包在 callback 內的 resolve 跟 reject 釋放出來，可以直接呼叫

```typescript
/**
 * Wrapper that expose the `resolve` and `reject` functions to enable scala style promise API.
 *
 * <code>
 *     const p = promise<number>()
 *     p.resolve(3)
 *     await p.await()
 * </code>
 */
class PromiseWrapper<T> {
    private readonly _promise: Promise<T>
    private _resolve
    private _reject
    private _resolved = false

    constructor() {
        this._promise = new Promise<T>(((resolve, reject) => {
            this._resolve = resolve
            this._reject = reject
        }))
    }
    private resolved() {
        if (this._resolved) {
            throw new Error("Promise is resolved already.")
        }
        this._resolved = true;
    }
    resolve<T>(value: T) {
        this.resolved()
        this._resolve(value)
    }

    reject(error: Error) {
        this.resolved()
        this._reject(error)
    }

    await(): Promise<T> {
        return this._promise
    }
}

export function promise<T>(): PromiseWrapper<T> {
    return new PromiseWrapper()
}

```

這下子程式可以改成這樣寫

```typescript

const p = promise<number>()
try {
    // do something
    p.resolve(1)
} catch(error) {
    p.reject(error)
}

```

看起來效益不高，但是這寫法真正的好處是，可以把 promise 當成一個變數，傳到其他的函式內，在其它的地方呼叫 promise.resolve(...)

這種寫法，在寫非同步程式的測式時很好用，在底下的例子中， `Engine` 是個 PubSub ，會非同步執行 `workflow` 內容。

因為是在背景執行又沒有吐出資料，那麼我們要怎麼測試這個 `Engine` 是不是每一步都有執行到，又要怎麼樣把每一步的結果傳出來到測試當中呢？

這時 `promise(...)` 就很好用了，先把 `promise` 開好，再傳到非同步的函式內去呼叫 `resolve(...)` ，這樣子就可以把中間的狀態送到外層去，而且在測式內也不用寫很醜的 `sleep(5000)` 。

`sleep(5000)`一開始付出的成本還好，等到專案寫了一年後，有上百個測式都在 `sleep(5000)` ，然後偶爾因為 CI 的機器太慢，等的不夠久造成程式失敗，重跑又要十幾分鐘，你就會知道為什麼要用 promise API 了。

```
describe("workflow", () => {
    const engine = new Engine(pub)
    it("can step through steps", async () => {
        const name = randomString(10)
        const result = promise<any>()
        const workflow = WorkflowBuilder.newBuilder(name)
            .from({
                name: "step-1",
                run(context: Context, event: Event) {
                    return 2
                }
            })
            .to({
                name: "step-2",
                run(context: Context, event: Event) {
                    result.resolve(context)
                    return
                }
            })
            .build()
        engine.register(workflow)
        engine.trigger(workflow, {"msg": "hello"})

        const ret = await result.await()
        expect(ret).toStrictEqual({...})
    })
})


```
