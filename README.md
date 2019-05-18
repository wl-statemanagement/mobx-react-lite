# mobx-react-lite <!-- omit in toc -->

[![Build Status](https://travis-ci.org/mobxjs/mobx-react-lite.svg?branch=master)](https://travis-ci.org/mobxjs/mobx-react-lite)[![Coverage Status](https://coveralls.io/repos/github/mobxjs/mobx-react-lite/badge.svg)](https://coveralls.io/github/mobxjs/mobx-react-lite)

[![Join the chat at https://gitter.im/mobxjs/mobx](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mobxjs/mobx?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This is a next iteration of [mobx-react](https://github.com/mobxjs/mobx-react) coming from introducing React hooks which simplifies a lot of internal workings of this package.

**You need React version 16.8.0 and above**

Class based components **are not supported** except using `<Observer>` directly in its `render` method. If you want to transition existing projects from classes to hooks (as most of us do), you can use this package alongside the [mobx-react](https://github.com/mobxjs/mobx-react) just fine. The only conflict point is about the `observer` HOC. Subscribe [to this issue](https://github.com/mobxjs/mobx-react/issues/640) for a proper migration guide.

[![NPM](https://nodei.co/npm/mobx-react-lite.png)](https://www.npmjs.com/package/mobx-react-lite)

Project is written in TypeScript and provides type safety out of the box. No Flow Type support is planned at this moment, but feel free to contribute.

## 💥Documentation moved! 💥

There is a new site dedicated to using [MobX in React](https://mobx-react.js.org/) world. You will find there detailed information about this package along with various recipes. More will be added over time. Feel free to contribute as well.

---

---

**Following utilities are still available in the package, but they are deprecated and will be removed in the next major version. As such, they are not mentioned on the documentation site.**

### `useObservable<T>(initialValue: T): T`

React hook that allows creating observable object within a component body and keeps track of it over renders. Gets all the benefits from [observable objects](https://mobx.js.org/refguide/object.html) including computed properties and methods. You can also use arrays, Map and Set.

Warning: With current implementation you also need to wrap your component to `observer`. It's also possible to have `useObserver` only in case you are not expecting rerender of the whole component.

```tsx
import { observer, useObservable, useObserver } from "mobx-react-lite"

const TodoList = () => {
    const todos = useObservable(new Map<string, boolean>())
    const todoRef = React.useRef()
    const addTodo = React.useCallback(() => {
        todos.set(todoRef.current.value, false)
        todoRef.current.value = ""
    }, [])
    const toggleTodo = React.useCallback((todo: string) => {
        todos.set(todo, !todos.get(todo))
    }, [])

    return useObserver(() => (
        <div>
            {Array.from(todos).map(([todo, done]) => (
                <div onClick={() => toggleTodo(todo)} key={todo}>
                    {todo}
                    {done ? " ✔" : " ⏲"}
                </div>
            ))}
            <input ref={todoRef} />
            <button onClick={addTodo}>Add todo</button>
        </div>
    ))
}
```

#### Lazy initialization

Lazy initialization (similar to `React.useState`) is not available. In most cases your observable state should be a plain object which is cheap to create. With `useObserver` the component won't even rerender and state won't be recreated. In case you really want a more complex state or you need to use `observer`, it's very simple to use MobX directly.

```tsx
import { observer } from "mobx-react-lite"
import { observable } from "mobx"
import { useState } from "react"

const WithComplexState = observer(() => {
    const [complexState] = useState(() => observable(new HeavyState()))
    if (complexState.loading) {
        return <Loading />
    }
    return <div>{complexState.heavyName}</div>
})
```

[![Edit TodoList](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/jzj48v2xry?module=%2Fsrc%2FTodoList.tsx)

Note that if you want to track a single scalar value (string, number, boolean), you would need [a boxed value](https://mobx.js.org/refguide/boxed.html) which is not recognized by `useObservable`. However, we recommend to just `useState` instead which gives you almost same result (with slightly different API).

### `useComputed(func: () => T, inputs: ReadonlyArray<any> = []): T`

Another React hook that simplifies computational logic. It's just a tiny wrapper around [MobX computed](https://mobx.js.org/refguide/computed-decorator.html#-computed-expression-as-function) function that runs computation whenever observable values change. In conjuction with `observer` the component will rerender based on such a change.

```tsx
const Calculator = observer(({ hasExploded }: { hasExploded: boolean }) => {
    const inputRef = React.useRef()
    const inputs = useObservable([1, 3, 5])
    const result = useComputed(
        () => (hasExploded ? "💣" : inputs.reduce(multiply, 1) * Number(!hasExploded)),
        [hasExploded]
    )

    return (
        <div>
            <input ref={inputRef} />
            <button onClick={() => inputs.push(parseInt(inputRef.current.value) | 1)}>
                Multiply
            </button>
            <div>
                {inputs.join(" * ")} = {result}
            </div>
        </div>
    )
})
```

Notice that since the computation depends on non-observable value, it has to be passed as a second argument to `useComputed`. There is [React `useMemo`](https://reactjs.org/docs/hooks-reference.html#usememo) behind the scenes and all rules applies here as well except you don't need to specify dependency on observable values.

[![Edit Calculator](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/jzj48v2xry?module=%2Fsrc%2FCalculator.tsx)

### `useDisposable<D extends TDisposable>(disposerGenerator: () => D, inputs: ReadonlyArray<any> = []): D`

The disposable is any kind of function that returns another function to be called on a component unmount to clean up used resources. Use MobX related functions like [`reaction`](https://mobx.js.org/refguide/reaction.html), [`autorun`](https://mobx.js.org/refguide/autorun.html), [`when`](https://mobx.js.org/refguide/when.html), [`observe`](https://mobx.js.org/refguide/observe.html), or anything else that returns a disposer.
Returns the generated disposer for early disposal.

Example (TypeScript):

```typescript
import { reaction } from "mobx"
import { observer, useComputed, useDisposable } from "mobx-react-lite"

const Name = observer((props: { firstName: string; lastName: string }) => {
    const fullName = useComputed(() => `${props.firstName} ${props.lastName}`, [
        props.firstName,
        props.lastName
    ])

    // when the name changes then send this info to the server
    useDisposable(() =>
        reaction(
            () => fullName,
            () => {
                // send this to some server
            }
        )
    )

    // render phase
    return `Your full name is ${props.firstName} ${props.lastName}`
})
```
