# Don't useReducer

## What is useReducer?
React 16.8 shipped with a hook called `useReducer` -- it was a nod to Redux’s API and is encouraged as 
a pattern for managing state when multiple objects change at the same time.

> useReducer is usually preferable to useState when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one. useReducer also lets you optimize performance for components that trigger deep updates because you can pass dispatch down instead of callbacks.

There is even a whole video explaining why `useReducer` provides cleaner code for complex operations.

I disagree.

## Typical useReducer (or Redux) implementation

Lets say you have a state model:
```ts
type HistoryState = {
  past: string[];
  present: string;
  future: string[];
};
```

Updating this state model requires a bunch of logic -- with a reducer you dispatch a description of what you want changed:
```ts
type ActionType =
  | {
      type: "BACK";
    }
  | {
      type: "FORWARD";
    }
  | {
      type: "PUSH";
      newPresent: string;
    };
```

Then, you implement a reducer function along with some functions to apply the changes:

```ts
const reducer = (
  previousState: HistoryState,
  action: ActionType
): HistoryState => {
  switch (action.type) {
    case "BACK":
      return applyBack(previousState);
    case "FORWARD":
      return applyForward(previousState);
    case "PUSH":
      return applyPush(previousState, action.newPresent);
  }
  return previousState;
};

const applyBack = (state: HistoryState): HistoryState => {
  if (state.past.length === 0) {
    return state;
  } else {
    return {
      past: state.past.slice(0, state.past.length - 2),
      present: state.past[state.past.length - 1],
      future: [...state.future, state.present]
    };
  }
};

const applyForward = (state: HistoryState): HistoryState => {
  if (state.future.length === 0) {
    return state;
  } else {
    return {
      past: [...state.past, state.present],
      present: state.future[state.future.length - 1],
      future: state.future.slice(0, state.future.length - 2)
    };
  }
};

const applyPush = (state: HistoryState, newPresent: string): HistoryState => {
  return {
    past: [...state.past, state.present],
    present: newPresent,
    future: []
  };
};

```

Then, because you want to avoid having to actually create the change objects inline (which can be error prone), you create some action creators:
```ts
const ActionCreators = {
  createBack: () => {
    return { type: "BACK" } as const;
  },
  createForward: () => {
    return { type: "FORWARD" } as const;
  },
  createPush: (newPresent: string) => {
    return { type: "PUSH", newPresent } as const;
  }
};
```

And finally, create your hook:
```ts
export const useHistory = (initialPresent: string) => {
  return useReducer(reducer, {
    past: [],
    present: initialPresent,
    future: []
  });
};
```

Voile! Clients can call your code:
```tsx
const App = () => {
  const [inputValue, setInputValue] = React.useState("");
  const [history, dispatch] = useHistory("present");
  
  const handleBack = () => {
    dispatch(ActionCreators.createBack());
  };

  const handleForward = () => {
    dispatch(ActionCreators.createForward());
  };

  const handleAdd = () => {
    dispatch(ActionCreators.createPush(inputValue));
    setInputValue("");
  };

  <div>{render some stuff...}<div>
```

## What isn't that cool...
The issue with the above code is onerous boilerplate. There is an entirely useless switch statement, it needs to handle all of the potential action types.
If you want to see how the back action is implemented, good luck! You navigate to "createBack" only to find a method that creates an object. To find the implementation you have to go find (the right) reducer, then see how it is implemented. In a large codebase, this is cumbersome at best.

## What other options are there?
Well, we can just use state.

Start with the same model:
```ts
type HistoryState = {
  past: string[];
  present: string;
  future: string[];
};
```

then, copy the functions from above:
```ts
const Actions = {
  applyBack: (state: HistoryState): HistoryState => {
    if (state.past.length === 0) {
      return state;
    } else {
      return {
        past: state.past.slice(0, state.past.length - 2),
        present: state.past[state.past.length - 1],
        future: [...state.future, state.present]
      };
    }
  },
  applyForward: (state: HistoryState): HistoryState => {
    if (state.future.length === 0) {
      return state;
    } else {
      return {
        past: [...state.past, state.present],
        present: state.future[state.future.length - 1],
        future: state.future.slice(0, state.future.length - 2)
      };
    }
  },

  applyPush: (state: HistoryState, newPresent: string): HistoryState => {
    return {
      past: [...state.past, state.present],
      present: newPresent,
      future: []
    };
  }
};
```

Wrap it into a hook:
```ts
const useHistory = (initialPresent: string) => {
  return useState<HistoryState>({
    past: [],
    present: initialPresent,
    future: []
  });
};
```

And voile! call it from your component:
```tsx
const App = () => {
  const [inputValue, setInputValue] = React.useState("");
  const [history, setHistory] = useHistory("present");
  
  const handleBack = () => {
    setHistory(Actions.applyBack);
  };

  const handleForward = () => {
    setHistory(Actions.applyForward);
  };

  const handleAdd = () => {
    setHistory(s => Actions.applyPush(s, inputValue));
    setInputValue("");
  };

  <div>{render some stuff...}<div>
```

It uses half the code, removes the boilerplate, and best yet, is completely navigable in your IDE.

[code](https://codesandbox.io/s/dont-use-reducer-cqbz4)
