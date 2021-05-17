# React Hook Anti-patterns

React Hooks have been a transformational tool for how we build React components in two significant ways:

One is that they allow for composable, reusable, well-encapsulated logic and behaviors to be shared between components.

The other is that they simplified building components: if you've ever migrated a codebase from class components to functional components and hooks, you likely decomposed some large monolithic components with multiple or very complex render methods into a set of separate functional components. This results in better encapsulation, simpler code, and potentially better performance.

However, like any powerful tool, hooks can be misused.

## The anti-pattern

I'm going to take a rather controversial stance in this post. I believe that _data fetching and reading global state from inside a hook leads to inefficient, hard to understand, and hard to refactor code._

What makes this controversial is that it seems to be the defacto pattern for derived data in hooks: fetch some data using React-Query, SWR, etc, calculate some data, and return the result. Sometimes some global state (from the URL or a context) gets pulled in.

The first time I wrote a hook like this it felt awesome. I could have a hook, (`useSomeHardToDeriveValue()`) and inject it into any component and share the logic. So encapsulated. So clean. So reusable.

The problem is that the story doesn't stop there. Code evolves.

The next week, another engineer needs a second value based on the same data, so they change the hook to return two values. Meanwhile, a third engineer needs to use that value as an input to some other calculation, so they create some new hook (`useSomeHarderToDeriveValue()`) that calls the first hook. Repeat this process a few more times. Soon, you end up with an tree of hooks that are being called that are very difficult to reason about and understand. Making changes to the first hook will likely have unanticipated consequences.

Lets dive in and talk about the individual claims.

### First, what makes this pattern inefficient?

Let me qualify my inefficiency claim by saying that I'm specifically referring to codebases that are using [suspense for data fetching](https://reactjs.org/docs/concurrent-mode-suspense.html). While this is considered 'experimental', it is offered by a number of data fetching libraries ([React Query](https://react-query.tanstack.com/guides/suspense), [SWR](https://swr.vercel.app/docs/suspense), [JOTAI](https://github.com/pmndrs/jotai), some in-house ones at companies I've worked) and is the only way to fetch data with Facebook's own [Relay Hooks](https://relay.dev/blog/2021/03/09/introducing-relay-hooks/). I've been on teams using it in production for almost 3 years and it is really pretty awesome, as it eliminates having to write your code to deal with un-loaded data: code execution doesn't proceed past the `useGetDataSomehow()` line until the data is available.

One of the challenges of suspense is that it is prone to create data-fetching waterfalls -- the code suspends loading the first piece of data, and often doesn't request the next piece until the first request completes. There are various strategies of mitigating this in components: pre-fetching, parallelized fetch hooks in the same component, pushing data fetching to leaf components (React evaluates suspended siblings in parallel), etc. **But with hooks, there is no way to mitigate this issue**.

As hooks get reused and nested, data may be fetched at each layer of a nested hook call, and will waterfall. A simple addition of a network call in a widely-used hook could have unanticipated negative impacts on the performance of the entire application. What is worse, it might have a small negative impact -- which is then repeated a dozen times until the application has strange performance characteristics without an obvious cause.

### Second what makes this code hard to understand?

Opacity. If all you could do in a hook was `useState` and `useEffect`, it might not be that big of a deal -- you're just encapsulating some local component behavior. The problem comes when something calls `useContext` (either directly or indirectly). Context is a great tool at times, but also can be incredibly detrimental to code understandability. It _hides inputs_ to hooks and can _mask side effects_.

Network calls can't be reliably pre-fetched because they are hidden dependencies. The fact that a certain calculation is dependent on a URL parameter is opaque to the consumer of the hook.

This is ultimately the same argument as to why global variables are bad: it tightly couples code, gives unpredictable results, decreases modularity, etc.

### Third, how does this hamper refactor-ability?

Ultimately, because the code is tightly coupled and hard to understand makes it difficult to refactor. It becomes difficult to understand all the downstream effects of a simple change, and devs are left with two common reactions. One is to make a change, cross their fingers, and hope for the best and/or sufficient test coverage. The other is to be afraid of making wholesale changes, so they do the minimal possible change, often leaving vestiges in the code because, "nobody really knows what that does, but we are scared to remove it," eventually resulting in an unmaintainable codebase.

## What do we do instead?

### Extreme version: use pure functions for business logic

A pure function, for the uninitiated, is a function that can be replaced by its return value -- it always returns the same result for the same set of inputs. Instead of writing business logic in hooks, pulling in data and global state, make those all explicit inputs:

```ts
const someValue = useSomeComplexCalculation();
```

becomes

```ts
const [apiEndpoint1Data, apiEndpoint2Data] = useGetData(
  apiEndpoint1,
  apiEndpoint2
);
const { someParam } = useParams();
const someValue = someComplexCalculation({
  apiEndpoint1Data,
  apiEndpoint2Data,
  someParam,
});
```

Is this noisier? Yes. However, I think the benefits outweigh the tradeoffs.

First, `someComplexCalculation` is more reusable/composable, statically memoizable, easier to unit test, etc.

Secondly, I can understand at a glance what is used in the calculation, without having to look at its implementation.

### A slightly less extreme version, when using Relay

Relay is a GraphQL client library developed by Facebook, and it exposes a powerful API for using GraphQL fragments. A fragment is a _declaration of required data_ that is needed for a component or function. Relay includes a hook, called `useFragment`, that allows you to **read data from the global store without causing side effects** that also **exposes the data dependency and decouples the hook from its consumer**.

What does the call signature look like in this case?

```ts
// Somewhere in the component
const someData = useQuery(
  graphql`
    query SomeComponentQuery {
      viewer {
        ...SomeComplexCalculationFragment
      }
    }
  `
);

const someValue = useSomeComplexCalculation({ someData });
```

I'm going to point out several things here:

1. The hook requires a reference to the data to be passed in: this maintains transparency.
2. The caller doesn't really know what the hook requires, only that it needs to include the hooks' fragment in its query.

### Wait, do I still have any hooks?

Yes, although a lot fewer of them. They end up being reserved for encapsulating reusable local component behavior.

### A note on enforcement

It is worth noting that unless this is enforced, you end up with the worst of both worlds: some noisy code, but it remains unpredictable. So, if you decide this makes sense, I suggest writing some ES Lint rules that prohibit calling data fetching or global contexts from inside hooks (e.g. no `useSwr`, `useQuery`, `useLazyLoadQuery`, `useParams` inside a hook).

## A code example

All of this has been pretty abstract -- lets look at an actual example in code. I'll use a financial application because it is an example that is easy to understand.

### The antipattern version

Imagine there is a list of accounts that I can fetch:

```ts
const accounts = useGetAccounts();
```

Lets say our application supports international accounts that might be denominated in different currencies, but I need to tell the user a notional balance in their currently selected currency. Hooks to the rescue!

First, I build a reusable currency converter that fetches exchange rates and returns a function bound to them:

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

Ok, lets stop a minute and look at what we built. First off, this is a pretty reasonable example that I've come across in several applications. It feels very powerful -- I can now pull the balance into any component that needs it. It is pretty understandable. It might even qualify as "Good Code".

However, there are already the seeds of problems that will plague our application as it grows and is refactored.

First, there is a network waterfall -- this hook will fetch the current exchange rates, and only once that is finished will fetch accounts.

Second, the caller of `useNotionalBalance` has no idea of all the side-effects that will occur by using this hook: it will trigger two network fetches, and depends on the state of the user's query string. While with a small example, reading the source for the hook is not a huge cognitive lift, it becomes drastically more difficult in a real application that might have further composed `useNotionalBalance` to calculate what is "available" (based on holds, etc).

While it _seems_ nice to encapsulate all of the dependencies into the hook, it ultimately hides concerns that the caller _should_ care about - it encapsulates the wrong things.

This makes the application hard to debug, test, and safely evolve.

Third, it is not very reusable -- how do I calculate the notional balance of only _some_ accounts? Etc.

## How does this look with pure functions?

First, the currency converter is just a function:

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

## How does this look with Relay + fragments?

As a bonus, lets look at what this becomes with Relay and fragments.

`useCurrencyConverter` will expose the fact that it requires some data, and require the caller to fetch the data _somehow_. (The caller doesn't even really know what the data is...)

```ts
const useCurrencyConverter = ({
  queryRef,
}: {
  queryRef: useCurrencyConverterFragment$Key;
}) => {
  const exchangeRates = useFragment(
    graphql`
      fragment useCurrencyConverterFragment on Query {
        exchangeRates {
          currencyPair
          exchangeRate
        }
      }
    `,
    queryRef
  );

  const exchangeRatesMap = exchangeRates.reduce(
    (acc, { currencyPair, exchangeRate }) => (
      (acc[currencyPair] = exchangeRate), acc
    ),
    {}
  );

  return (amount, fromCurrency, toCurrency) => {
    return amount * exchangeRateMap[`${fromCurrency}-${toCurrency}`];
  };
};
```

Same thing for calculating the notional balance:

```ts
const useNotionalBalance = ({
  accountsRef,
  queryRef,
  notionalCurrency,
}: {
  accountsRef: useNotionalBalanceFragment$Key;
  queryRef: useCurrencyConverterFragment$Key;
  notionalCurrency: string;
}) => {
  const accounts = useFragment(graphql`
    fragment useNotionalBalanceFragment on AccountList {
      account {
        balance
        currency
      }
    }
  `);

  const currencyConverter = useCurrencyConverter(queryRef);

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
  const query = useQuery(graphql`
    query ComponentQuery {
        ...useCurrencyConverterFragment,
        viewer {
            checkingAccounts: accounts(type: 'CHECKING') {
                ...useNotionalBalanceFragment
            }
            savingsAccounts: accounts(type: 'SAVINGS') {
                ...useNotionalBalanceFragment
            }
        }
    }
  `);
  const notionalCurrency = useParams("notional_currency");

  const checkingBalance = useNotionalBalance({
    query,
    accountsRef: query.viewer.checkingAccounts,
    notionalCurrency,
  });
  const savingsBalance = useNotionalBalance({
    query,
    accountsRef: query.viewer.savingsAccounts,
    notionalCurrency,
  });

  return (
    <div>
      Balance: {checkingBalance} (checking), {savingsBalance} (savings)
    </div>
  );
};
```

A few observations:

1. I can see all of the _inputs_ to calculating a notional balance -- it needs a query that can power the currency converter, and some fields on the account.
2. The calculation can be reused for different sets of accounts.

## Conclusion

Hooks, especially data-fetching-hooks-with-suspense have been around for a few years now, and I feel like the best practices are still being worked out. After working on several large code bases, I've become convinced that there are some huge pitfalls that are easy to fall into. Avoiding data fetching and global state access from hooks I think avoids a lot of them.
