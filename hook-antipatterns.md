# React Hook Antipatterns

React Hooks have been a transformational tool for how we build React components in two significant ways:

One is that they allow for composable, reusable, well-encapsulated behaviors to be shared between components.

The other is that they simplified building components: if you've ever migrated a codebase from class components to functional components and hooks, you likely decomposed some large monolithic components with multiple or very complex render methods into a set of separate functional components. This results in better encapsulation, simpler code, and potentially better performance.

However, like any powerful tool, hooks can be misused. Specifically, they can be used in ways that hinder understandability and refactorability, making code brittle and unapproachable. Again, if you've worked on a sufficiently large codebase you've probably come across some hooks that were extremely complex and/or brittle and made you question whether the entire paradigm should be discarded. Usually this is the product of starting with great code that then evolves in unexpected ways (which is preciesly what any codebase does with a team of > 2 engineers working on it).

Here we will explore a few of examples to help illustrate the types of problems that might be encountered, and then propose several principles to help minimize or mitigate these classes of problems.

## An example

Lets assume we are building an application using a modern hooks-based fetching library, such as React-Query, SWR, Apollo hooks, or Relay. (Note: These libraries are great examples of extracting reusable logic into hooks, effectively separating the concerns of _how_ you get data from _what_ data a component needs.)

I'll use a financial application because it is an example that is easy to understand.

Imagine this application has a series of accounts that I can fetch using one of these data fetching libraries:

```ts
const accounts = useGetAccounts();
```

> For the uninitiated, `use` as a prefix signals that this is a React hook: it is a way to signal that it may in turn, call other hooks, "hooking" into React state and lifecycle methods. It means you can't use this function outside of React.

Lets say our application supports international accounts that might be denominated in different currencies, but I need to tell the user a notional balance in their currently selected currency. Hooks to the rescue!

First, I build a reusable currency converter that makes a function bound to the current exchange rates:

```ts
const useCurrencyConverter = () => {
  const exchangeRates = useGetExchangeRates();
  return (amount, fromCurrency, toCurrency) => {
    return amount * exchangeRates[`${fromCurrency}-${toCurrency}`];
  };
};
```

And then can use it as part of another hook that calculates the balance:

```ts
const useNotionalBalance = () => {
  const currencyConverter = useCurrencyConverter();
  const accounts = useGetAccounts();
  const notionalCurrency = useParams("notional_currency");

  return accounts.reduce((previousTotal, account) => {
    const accountBalance = currencyConverter(
      account.balance,
      account.currency,
      notionalCurrency
    );
    return previousTotal + accountBalance;
  }, 0);
};
```

And then in a component:

```tsx
const Component = () => {
  const notionalBalance = useNotionalBalance();

  return <div>Balance: {notionalBalance}</div>;
};
```

Lets stop a minute and talk about what we have here. FIrst off, this is a pretty reasonable example that I've come across in several applications. It feels very powerful -- I can now pull the balance into any component that needs it. It is pretty understandable. It might even qualify as "Good Code".

However, there are already the seeds of problems that will plague our application as it grows and is refactored.

### Problem 1 data fetching waterfalls (if using suspense)

Now, a lot of the new hooks-based data-fetching libraries make use of React's experimental suspense "API", which pauses rendering until the data is available. This is really nice because it prevents having to deal with null/undefined states. Unfortunately, it is also prone to creating waterfalls. In the case above, account balances won't be fetched until exchange rates have successfully been fetched.

Technically, this is more of a pitfall of using suspense -- but whereas it can be mitigated in components (by pushing data fetching to the leaf components and parallelizing multiple data fetching calls in components), it is exacerbated by hooks largely because of the second problem, which that hooks are opaque.

### Problem 2 hooks are opaque

In our example above, the caller of `useNotionalBalance` has no idea of all the side-effects that will occur by using this hook: it will trigger two network fetches, and depends on the state of the user's query string. While with a small example, reading the source for the hook is not a huge cognitive lift, it becomes drastically more difficult in a real application that might have further composed `useNotionalBalance` to calculate what is "available" (based on holds, etc). There is also likely dozens of these hooks in a single component, making the cognative load quite significant.

While it _seems_ nice to encapsulate all of the dependencies into the hook, it ultimately hides concerns that the caller _should_ care about.

This makes the application hard to debug, test, and safely evolve.

## What could be different?

Lets consider another implementation of what we have above that might address these problems.

Instead of having the currency converter fetch exchange rates, make it a pure function and pass exchange rates as an argument:

```ts
const makeCurrencyConverter = (exchangeRates) => {
  return (amount, fromCurrency, toCurrency) => {
    return amount * exchangeRates[`${fromCurrency}-${toCurrency}`];
  };
};
```

Same thing for calculating the notional balance, it is now a pure function:

```ts
const calculateNotionalBalance = ({
  accounts,
  exchangeRates,
  notionalCurrency,
}) => {
  const currencyConverter = makeCurrencyConverter(exchangeRates);

  return accounts.reduce((previousTotal, account) => {
    const accountBalance = currencyConverter(
      account.balance,
      account.currency,
      notionalCurrency
    );
    return previousTotal + accountBalance;
  }, 0);
};
```

And the component:

```tsx
const Component = () => {
  const [exchangeRates, accounts] = useParallel(
    useGetExchangeRates,
    useGetAccounts
  );
  const notionalCurrency = useParams("notional_currency");

  const notionalBalance = calculateNotionalBalance({
    exchangeRates,
    accounts,
    notionalCurrency,
  });

  return <div>Balance: {notionalBalance}</div>;
};
```

How does this compare to the first example?

The biggest difference is that it doesn't encapsulate the logic into hooks anymore - instead they are pure functions. This documents their dependencies and makes them easier to test at the expense of more work at the callsite. However, I would argue this also makes them more reusable. For instance, this new version of `calculateNotionalBalance` could be reused to calculate the balance for a subset of accounts, or we could calculate the notional balance in multiple different currencies. This was not possible before.

This also eliminates the waterfall concern because the data requirements of the function are exposed as arguments.

## Takeway

Ok, so this is neat. How do we encourage engineers to build #2 and not #1?

### Principle 1: Avoid data fetching in hooks

**exception** Relay's `useFragment`

### Principle 1b: Avoid accessing context in hooks

-- no `useNavigation`
-- no `useParams`
-- no `use`

### Wait, what is left in my hooks at this point?

If we are no longer fetching data or accessing global state in hooks, what is left? A lot of times, nothing, and you can turn themm into **pure functions** and create better, more testable, and more understandable code.

This relegates hooks to being used for things that need to participate in the React component lifecycle and/or need to have state.
