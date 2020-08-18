# Don't useReducer

## A little history

Prior to React 16.8, components stored their local state in a single object, and typically had functions to update portions of this state:

```tsx
class Component extends React.Component {
  constructor(props) {
    this.state = {
      firstName: 'Chris',
      lastName: 'Erickson',
    }
  }

  setFirstName: (value) => this.setState({firstName: value})
  setLastName: (value) => this.setState({lastName: value})

  render() {
    const {firstName, lastName} = this.state;

    return ...
  }
}
```

With the release of React 16.8, hooks were added and people started moving from class to functional components. The `useState` hook allowed us to simplify some of this boilerplate and separate each piece of state into a separate variable:

```tsx
const Component = () => {
  const [firstName, setFirstName] = React.useState('Chris');
  const [lastName, setLastName] = React.useState('Erickson');

  return ...
}
```

This offers several key benefits:

- State becomes local variables located in the render function, and can be declared immediately before it is used.
- Smaller, more granular state objects are simpler and can be easier to think about.
- Less boilerplate
- ... (think more about this)

## The standard argument for useReducer

However, there can also be downsides, as [this video](https://egghead.io/lessons/react-when-to-usereducer-instead-of-usestate) does a great job explaining. The suggestion is that when there are pieces of interrelated state that change together, they should be grouped together into a single, multi-valued state object. This is especially true when a state update should be atomic.

Using their example, this:

```tsx
const [past, setPast] = useState([]);
const [present, setPresent] = useState(initialPresent);
const [future, setFuture] = useState([]);
```

becomes this:

```ts
const [{ past, present, future }, setHistory] = useState({
  past: [],
  present: initialPresent,
  future: [],
});
```

The video goes on to argue that this is a good case for `useReducer`.

From the React docs:

> useReducer is usually preferable to useState when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one. useReducer also lets you optimize performance for components that trigger deep updates because you can pass dispatch down instead of callbacks.

In short, `useReducer` gives you a Redux-like API for making state changes using a single reducer function and dispatching actions to it.
(Note, the statement about performance is misleading: the same is true if you pass down a state setter or wrap your callback in `useCallback`).

Lets look at the history example -- with `useState`, each operation is defined as a function:

```js
const [{ past, present, future }, setHistory] = useState({
  past: [],
  present: initialPresent,
  future: [],
});

const goBack = useCallback(() => {
  setHistory((previous) => {
    if (previous.past.length === 0) {
      return previous;
    } else {
      return {
        past: previous.past.slice(0, previous.past.length - 2),
        present: previous.past[previous.past.length - 1],
        future: [...previous.future, previous.present],
      };
    }
  });
}, setHistory);

const goForward = useCallback(() => {
  setHistory((previous) => {
    if (previous.future.length === 0) {
      return previous;
    } else {
      return {
        past: [...previous.past, previous.present],
        present: previous.future[previous.future.length - 1],
        future: previous.future.slice(0, previous.future.length - 2),
      };
    }
  });
}, setHistory);

const push = useCallback((newValue) => {
  setHistory((previous) => ({
    past: [...previous.past, previous.present],
    present: newPresent,
    future: [],
  }));
});
```

Restructuring this with `useReducer`:

```js
const historyReducer = (previous, action) => {
  switch (action.name) {
    case "back":
      if (previous.past.length === 0) {
        return previous;
      } else {
        return {
          past: previous.past.slice(0, previous.past.length - 2),
          present: previous.past[previous.past.length - 1],
          future: [...previous.future, previous.present],
        };
      }
      break;

    case "forward":
      if (previous.future.length === 0) {
        return previous;
      } else {
        return {
          past: [...previous.past, previous.present],
          present: previous.future[previous.future.length - 1],
          future: previous.future.slice(0, previous.future.length - 2),
        };
      }
      break;

    case "push":
      setHistory((previous) => ({
        past: [...previous.past, previous.present],
        present: action.newPresent,
        future: [],
      }));
  }
};
```

```js
const [{ past, present, future }, dispatchHistory] = useReducer(
  historyReducer,
  {
    past: [],
    present: initialPresent,
    future: [],
  }
);

const goBack = useCallback(() => dispatchHistory({ action: "back" }), []);
const goForward = useCallback(() => dispatchHistory({ action: "forward" }), []);
const push = useCallback(
  (value) => dispatchHistory({ action: "push", newPresent: value }),
  []
);
```

The touted benefits of `useReducer` in this case are that it co-locates all of the logic that can modify history into a single function thus creating cleaner code for complex operations.

(There may be an argument that this creates these beautiful declarative operations for updating state, but I think this argument lacks practical teeth -- I'm dubious that the majority of people keep the event stream of dispatched actions to read it and understand what's happened.)
(There is also an argument that it gives you a single place to observe state changes, e.g. one breakpoint at the top of the reducer. This may have some merit but I think it is probably overstated, as you often only want to watch for a certain type of change.)

I disagree.

First off, I don't think the above switch statement is inherently cleaner or easier to understand. But moreover, I think it is a trivial example that doesn't represent what happens in a typical application.

## Typical useReducer implementation

Frequently, when `useReducer` is reached for, it is to manage a large state model in a Redux-like way, and the toy example above

First, with a large team, it is likely that Typescript will be used to improve developer productivity, document the code, and make refactoring safer. So we'll define a state model:

```ts
type HistoryState = {
  past: string[];
  present: string;
  future: string[];
};
```

Next, to avoid problems where invalid actions are being dispatched, we'll define our action types and their correct payloads:

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

Then, because we want to be able to unit test each action type (and compose them), the actual implementations will be pulled out into separate functions:

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

Then wired together with a reducer:

```ts
const reducer = (previous: HistoryState, action: ActionType): HistoryState => {
  switch (action.type) {
    case "back":
      return applyBack(previous);
    case "forward":
      return applyForward(previous);
    case "push":
      return applyPush(previous, action.newPresent);
  }
  return previous;
};
```

Then the actual `useDispatcher` call:

```ts
const [{ past, present, future }, dispatch] = useDispatcher(reducer, {
  past: [],
  present: initialPresent,
  future: [],
});
```

## What's wrong with this picture

There are several things about this pattern that make it painful:

- Lack of code navigibility, indirection. In a real application, these large reducers will often get split into multiple files with the actual implementation of the state modification potentially in a different file than the reducer itself. When trying to debug a problem, you find yourself first making 3-4 code hops and completely losing context when trying to understand what a dispatcher is doing. The indirection actively hinders code understandability.

- Boilerplate (or bugs). Adding new actions requires one to add to the ActionType, the reducer, as well as creating the function to make the state update. This example leaves off additional common patterns, such as `actionCreators` that are used to create the actions when you dispatch them. I can hear someone saying, "But that's just typescript!" -- but without typescript we have a different problem -- there are so many places in this chain of logic that can break -- action names might be different, payloads not be as expected, etc.

## What are the other options?

To be honest, I don't see what is wrong with the state example above, except perhaps as written the functions are not reusable or unit testable, but this is a simple refactor. Lets look at how we could build a reusable version of this just using state and sharing functions:

Lets start with the same model:

```ts
type HistoryState = {
  past: string[];
  present: string;
  future: string[];
};
```

And use the functions that the reducer called:

```ts
const HistoryStateActions = {
  applyBack: (state: HistoryState): HistoryState => {
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
  applyForward: (state: HistoryState): HistoryState => {
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

  applyPush: (state: HistoryState, newPresent: string): HistoryState => {
    return {
      past: [...state.past, state.present],
      present: newPresent,
      future: [],
    };
  },
};
```

This can now be used as follows:

```tsx
const App = () => {
  const [inputValue, setInputValue] = React.useState("");
  const [history, setHistory] = useState<HistoryState>({past: [], present: "present", future: []);

  const handleBack = () => {
    setHistory(HistoryStateActions.applyBack);
  };

  const handleForward = () => {
    setHistory(HistoryStateActions.applyForward);
  };

  const handleAdd = () => {
    setHistory(s => HistoryStateActions.applyPush(s, inputValue));
    setInputValue("");
  };

  <div>{render some stuff...}<div>
```

What are some of the benefits?

- Code navigation: When I go to definition on `HistoryStateActions.applyBack`, it takes me straight to the implementation. There is no indirection stepping through a reducer to find the implementation.
- Reduction in boilerplate: This code removes the boilerplate (and bug prone) switch statement in the dispatcher, as well as ActionTypes type declaration. It also removes the chance that there is a missing case in the dispatcher, or a mismatch in an action name somewhere. Refactoring is easier (I can just change the arguments to the function -- I don't need to update the action's payload type anywhere).
