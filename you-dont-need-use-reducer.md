# You don't need useReducer

`useReducer` is alternative to the `setState` hook that is frequently suggested for handling updates to complex state objects.

From the React docs:

> useReducer is usually preferable to useState when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one. useReducer also lets you optimize performance for components that trigger deep updates because you can pass dispatch down instead of callbacks.

[This video](https://egghead.io/lessons/react-when-to-usereducer-instead-of-usestate) explains some of the dangers of separating dependent/related state into multiple `useState` hooks, and explains how this can be solved by grouping them into a single state hook. This is an important point that should not be minimized.

It goes on to say that it is an ideal case for `useReducer`.

I disagree.

## Everything you can do with useReducer can also be done with useState

> useReducer is usually preferable to useState when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one.

Luckily, `setState` can also be called with an updater function, giving the same capabilities.

> useReducer also lets you optimize performance for components that trigger deep updates because you can pass dispatch down instead of callbacks.

Again, the same can be said for the `setState` function, and/or wrapping the callbacks in `useCallback`.

## An example

Using the example from the video, lets see how this code might look using `setState` and updater functions.

First, lets remind ourselves of the example -- a state object with past, present, and future, which should be atomically updated:

```ts
type History = {
  past: string[];
  present: string;
  future: string[];
};
```

Now, we can define our actions as functions. (Note: I am using higher-ordered functions here, but that is not requried):

```ts
type Updater = (state: HistoryState) => HistoryState;

const HistoryUpdaters = {
    back: (): Updater =>
        (state) => {
            if (state.past.length === 0) {
                return state;
            } else {
                return {
                    past: state.past.slice(0, state.past.length - 2),
                    present: state.past[state.past.length - 1],
                    future: [...state.future, state.present],
                };
            }
        },
    forward: (): Updater =>
        (state) => {
            if (state.future.length === 0) {
                return state;
            } else {
                return {
                    past: [...state.past, state.present],
                    present: state.future[state.future.length - 1],
                    future: state.future.slice(0, state.future.length - 2),
                };
            }
        },
    push: (newPresent: string): Updater =>
        (state) => ({
                past: [...state.past, state.present],
                present: newPresent,
                future: [],
            });
}
```

Then, used in a component:

```tsx
const Component: React.FC = () => {
  const [{ past, present, future }, setHistory] = React.useState<History>({
    past: [],
    present: initialPresent,
    future: [],
  });

  const [textInput, setTextInput] = React.useState("");

  return (
    <div>
      <button onClick={() => setHistory(HistoryActions.back())}>Back</button>
      <div>
        <input
          type="text"
          value={textInput}
          onChange={({ target }) => setTextInput(target.value)}
        />
        <button onClick={() => setHistory(HistoryActions.push(textInput))}>
          Add
        </button>
      </div>
      <button onClick={() => setHistory(HistoryActions.forward())}>Back</button>
    </div>
  );
};
```

As you can see, `setHistory` is called (`dispatched`) with the function grabbed from `HistoryActions`, offering the same benefits as `useDispatcher`. However, it has some distinct advantages (like, go to definition, lack of boilerplate code) that are outlined below.

## Comparison to `useReducer`

|                                                    | useReducer | useState with updater |
| -------------------------------------------------- | ---------- | --------------------- |
| Provides functions for complex updates             | Yes        | Yes                   |
| Updates state based on previous value              | Yes        | Yes                   |
| Passes dispatch instead of callbacks (performance) | Yes        | Yes                   |
| **Binds updater function to state**                | Yes        | No                    |
| **Code understandability and navigability**        | No         | Yes                   |

Lets talk about the differences:

### `useReducer` binds the updater function to state

`useReducer` accepts a single updater function when the hook is called. This means that callers (in theory) can only dispatch allowed actions, where as `setState` doesn't prevent callers from making invalid state updates. While this may offer some benefits (it still doesn't prevent invalid actions which result in errors or noops), I argue they do not outweigh the big downside: understanding and code navigability.

### Code understandability and navigability

My main argument against `useReducer` is how difficult it makes it to navigate, debug, and understand the code.

Consider the equivelent dispatch call:

```ts
dispatch({ action: "push", newValue: textInput });
```

How do you navigate to the definition? You're forced to go to the definition of `dispatch`, which will take you to the `React.useDispatch` call. From there, you navigate to the definition of the reducer (which is typically extracted for unit testing). From there, you need to trace through the dispatch method (remembering the action that was dispatched) to see what actually happens.

And this is a trivial case. Most real-world implementations involve a lot of additional ceremony.

### Ceremony and real world applications

In most real-world application, the actions will be typed to prevent dispatching an incorrect action:

```ts
type ActionType =
  | {
      type: "back";
    }
  | {
      type: "forward";
    }
  | {
      type: "push";
      newPresent: string;
    };
```

Then, because we want to be able to unit test each action type (and maybe compose them), the actual implementations get pulled out into separate functions (that are the same thing as our updater functions in the `setState` example):

```ts
const applyBack = (previous: HistoryState): HistoryState => {
  if (previous.past.length === 0) {
    return previous;
  } else {
    return {
      past: previous.past.slice(0, previous.past.length - 2),
      present: previous.past[previous.past.length - 1],
      future: [...previous.future, previous.present],
    };
  }
};

const applyForward = (previous: HistoryState): HistoryState => {
  if (previous.future.length === 0) {
    return previous;
  } else {
    return {
      past: [...previous.past, previous.present],
      present: previous.future[previous.future.length - 1],
      future: previous.future.slice(0, previous.future.length - 2),
    };
  }
};

const applyPush = (
  previous: HistoryState,
  newPresent: string
): HistoryState => {
  return {
    past: [...previous.past, previous.present],
    present: newPresent,
    future: [],
  };
};
```

And then wired together with a reducer:

```ts
const reducer = (previous: HistoryState, action: ActionType): HistoryState => {
  switch (action.type) {
    case "back":
      return applyBack(previous);
    case "forward":
      return applyForward(previous);
    case "push":
      return applyPush(previous, action.newPresent);
    default:
      console.error(`Unknown action type: ${action}`);
  }
  return previous;
};
```

So, in addition to the multi-step navigation already inherent with `useDispatcher`, we now have an additional reducer function to wire all of our action implementations together -- a function with no value beyond action routing, but plenty of opportunity to introduce bugs and obfuscate code. Some of this can be addressed by using really good code organization conventions -- but all of this is unnecessary boilerplate code that can introduce bugs and inhibits understandability.

## Conclusion

TLDR? There is ultimately nothing that can be done with `useReducer` that can't be done with `useState`. The minor benefit in binding the updater function to the state is nothing compared to the drawbacks of obfuscation and typical boilerplate.
