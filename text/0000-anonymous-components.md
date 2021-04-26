- Start Date: 2021-02-09
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC introduces a new pattern/primitive, the `Anonymous` component (and the `anonymous` function as a layer of syntatic sugar). Final naming is TBD, userspace proof of concept [here](https://github.com/lewisl9029/react-anonymous), implementation is a [1 liner](https://github.com/lewisl9029/react-anonymous/blob/master/src/index.js).

Essentially, this RFC aims to relax the now infamous Rules of Hooks to provide significantly better developer ergonomics by making hook usage less restrictive when it comes to the extra levels of indirection it often necessitates, i.e. defining/naming trivial new components to comply with rules of hooks, and all the associated boilerplate with types and props drilling.

Here's what the new rules of hooks could look like once this RFC is fully adopted:

> ## Only Call Hooks at the Top Level of Component Functions
> 
> Don’t call Hooks inside loops, conditions, or nested functions. Instead, always use Hooks at the top level of your component function (including [anonymous](link-to-guide-on-anonymous-component-usage) ones). By following this rule, you ensure that Hooks are called in the same order each time a component renders. That’s what allows React to correctly preserve the state of Hooks between multiple useState and useEffect calls.

(TODO: Could use some help to wordsmith this for better clarity)

# Basic example

Using my proof of concept in userspace: https://github.com/lewisl9029/react-anonymous

Examples are slightly contrived to fit in a variety of use cases, but hopefully it showcases all the new possibilities adequately. I've left some inline comments sprinkled throughout to help focus attention and provide additional context.

```js
// This example uses the `anonymous` function for syntactic sugar to 
// minimize boilerplate/nesting, instead of the `Anonymous` component.
import * as React from "react";
import anonymous from "@lewisl9029/react-anonymous";

const ImagineIfYouCould = ({ 
  colors, 
  loading
}: { 
  colors: string[],
  loading: boolean 
}) => {
  if (loading) {
    // Call hooks within an early return
    return anonymous(() => React.useContext(LoadingMessageContext));
  }

  // Call hooks after an early return
  return anonymous(() => {
    const [isOpen, setIsOpen] = React.useState(false);

    return (
      <div>
        {isOpen
          ? // Call hooks within an inline branching path
            anonymous(() => (
              <Modal
                close={React.useCallback(() => setIsOpen(false), [])}
              >
                <ul>
                  {colors.map((color) =>
                    // Call hooks within a mapping function
                    anonymous(() => (
                      <li style={useMemo(() => ({ color }), [color])}>{color}</li>
                    ))
                  )}
                </ul>
              </Modal>
            ))
          : null}
      </div>
    );
  });
};
```

For contrast, here's an equivalent implementation without the anonymous component pattern (again using comments to offer commentary):

```js
import * as React from "react";

// Note how you can no longer follow the implementation 
// in natural reading order from top to bottom.
const Loading = () => React.useContext(LoadingMessageContext)

type Color = string;
type Colors = Color[];

// Note how many levels this color prop had to be drilled down to get here,
// when it could have been directly referenced within the same closure.
const Color = ({ color }: { color: Color }) => (
  <li style={useMemo(() => ({ color }), [color])}>{color}</li>
)

const ColorsModal = ({ 
  setIsOpen, 
  colors 
}: { 
  setIsOpen: (boolean) => void, 
  // Note how many times these types had to be repeatedly defined,
  // when they could be fully inferred if referred to inline.
  colors: Colors 
}) => (
  <Modal
    // Note that it's no longer possible for the linter to infer
    // setIsOpen will never change.
    close={React.useCallback(() => setIsOpen(false), [setIsOpen])}
  >
    <ul>
      {colors.map((color) =>
        <Color color={color} />
      )}
    </ul>
  </Modal>
)

const Loaded = ({ colors }: { colors: Colors }) => {
  const [isOpen, setIsOpen] = React.useState(false);

  return (
    <div>
      {isOpen
        ? <ColorsModal colors={colors} setIsOpen={setIsOpen} />
        : null}
    </div>
  );
}

const ImagineIfYouCould = ({ 
  colors, 
  loading
}: { 
  colors: Colors,
  loading: boolean 
}) => {
  if (loading) {
    // Note how little information these component names add
    // when the immediate context makes them completely redundant.
    return <Loading />;
  }

  return <Loaded colors={colors} />;
};
```

Hopefully this was enough to pique your interest and demonstrate how this pattern can drastically reduce the amount of friction involved with using hooks on a day to day basis.

# Motivation

Part of the [original motivation](https://reactjs.org/docs/hooks-intro.html#its-hard-to-reuse-stateful-logic-between-components) for React Hooks was to reduce indirection necessitated by earlier patterns like render props and higher-order components:

> If you’ve worked with React for a while, you may be familiar with patterns like render props and higher-order components that try to solve this. But these patterns require you to restructure your components when you use them, which can be cumbersome and make code harder to follow.

React Hooks in its current iteration doesn't fully accomplish this design goal, as noted in the [original list of drawbacks](https://github.com/reactjs/rfcs/blob/master/text/0068-react-hooks.md#drawbacks):

> - The “Rules of Hooks”: in order to make Hooks work, React requires that Hooks are called unconditionally. Component authors may find it unintuitive that Hook calls can't be moved inside if statements, loops, and helper functions.
> - The “Rules of Hooks” can make some types of refactoring more difficult. For example, adding an early return to a component is no longer possible without moving all Hook calls to before that conditional.

In practice, transitioning from render props and HOCs to React Hooks involved trading the various forms of indirection necessitated by those patterns for a different form of indirection necessitated by the rules of hooks, which often forces users to create extra layers of components for the sole purpose of ensuring hook calls remain at the top level of component functions.

As illustrated in the example above, this new form of indirection can add a significant amount of friction to working with hooks on a day to day basis, causing countless new named components to be created whose name might add little to no value, and in the process exacerbating prop drilling and types boilerplate.

The anonymous component pattern introduced in this RFC aims to directly address these shortcomings of React Hooks and offer users the option to use hooks with significantly less indirection.

More interestingly, it opens up the possibility for a new set of use cases for hooks where previously the excessive levels of indirection necessitated by the current rules of hooks made usage ergonomics prohibitively poor. My most prominent use case for this is an experimental [useStyles](https://github.com/lewisl9029/use-styles) CSS-in-JS hook meant to be used at arbitrary depth to provide styles to elements without necessitating indirection through named styled components at every step of the way (think of it as a more robust, runtime-only version of `@emotion/babel-plugin`'s [css prop](https://emotion.sh/docs/css-prop)). I'm hopeful that more use cases for React Hooks like this will be discovered once we're able to alleviate the forced indirection problem.

# Detailed design

The `Anonymous` component is a component that delegates the implementation of its entire render function to the user through its `children` render prop. 

```js
export const Anonymous = ({ children }) => children()
```

Users can render this component with arbitrary implementation to call hooks at any level of nesting/branching/mapping within a tree of React elements, while keeping consistent hook call ordering thanks to the `Anonymous` component acting as a distinct component for hooks call ordering at every level. 

Here's what usage of the `Anonymous` component looks like in practice in the most trivial case: 

```js
const Example = () => {
  return (
    <Anonymous>
      {() => { 
        return useMemo(() => someExpensiveFunction(), [])
      }}
    </Anonymous>
  )
}
```

At first sight, this might look like a violation of the rules of hooks on calling within a callback, but in practice, it's not in violation because the hooks end up getting called at the top level of the `Anonymous` component's render function thanks to the `children` render prop.

The [official linting rule for React hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks) is not able to recognize this fact, however, and thus still treats this as a violation. So we'll need to make an exception in the linting rule for the anonymous component pattern, as I've done in [this fork](https://github.com/facebook/react/compare/master...lewisl9029:support-render-hooks-in-rule-of-hooks).

This was actually my main motivation for writing up this RFC in hopes of getting official blessing for the pattern, since the lack of support in the linter rule is by far the biggest practical impediment for this pattern to gain widespread adoption in the real world.

We also currently offer an `anonymous` function as syntactic sugar on top of the `Anonymous` component.

```js
export const anonymous = (children, { key } = {}) => React.createElement(Anonymous, { children, key })
```

Here's what it looks like in usage:

```js
const Example = () => {
  return anonymous(() => { 
    return useMemo(() => someExpensiveFunction(), [])
  })
}
```

As you can see, it is quite a bit easier to type and more compact than the component version in its current state.

We accept the React `key` as an arg to offer a workaround for this edge case where the same `Anonymous` component ends up getting rendered in both branches:

```js
isOpen
  ? anonymous(() => useMemo(() => "yes", []), { key: "yes" })
  : anonymous(() => useMemo(() => "no", []), { key: "no" })
```

When `isOpen` changes, React will rerender the same `Anonymous` component instead of unmounting/remounting a separate component for the other branch if we don't have a `key` to distinguish between them. We take the `key` as part of an options object instead of accepting it directly to allow for future extension without introducing breaking changes.

# Drawbacks

Here are some drawbacks that I've thought about so far:

- This is yet another new pattern to teach, and makes the rule of hooks slightly more nuanced than it already is, and thus potentially harder to teach as well.
- This pattern necessitates more complexity in the linter rule implementation. I have implemented the necessary changes in [this fork](https://github.com/facebook/react/compare/master...lewisl9029:support-render-hooks-in-rule-of-hooks), but it could very well be missing edge cases. It may necessitate the same for the [react-refresh babel plugin](https://github.com/facebook/react/blob/master/packages/react-refresh/src/ReactFreshBabelPlugin.js) as well, though I haven't looked into it in detail yet. Same applies to any other form of static analysis the React team may be planning to introducing in the future.
- It is possible to abuse this pattern to create larger component functions than would otherwise be possible with the current rules of hooks, which can act as a forcing function that limits component function size in certain cases. Though in my opinion, this forcing function adds net negative value as it removes too many degrees of freedom over when to add/remove indirection from the hands of users.
- This pattern can add extra lines and extra levels of indentation in component functions when used, which can add up to a significant level of visual noise.
- This pattern can introduce a significant number of `Anonymous` component nodes, which can make debugging more cumbersome, and may have performance implications for reconciliation due to a larger component tree.

# Alternatives

This pattern went through several naming iterations:

- I started with "render hooks", reflecting a combination between render props and hooks.
- Then "boundary components", as it creates a component "boundary" inline at which users can call hooks.
- Finally "anonymous components", which I like the best so far, as I feel the semantics & mental model it generates aligns very well with the usage pattern and how things work under the hood.

Happy to take other suggestions as well, of course.

I'm still not entirely settled on whether to recommend the component API or the function one. Here are some of the tradeoffs I've been pondering about:

- The function version only takes up 2 extra lines, and 1 level of indent in a prettier formatted codebase, compared to 4 extra lines and 2 levels of indent for the component version. This is why the function API is the one I currently use, and what I recommend as the default in the userspace library, though once this pattern gains enough adoption, we could possibly petition prettier to format the `Anonymous` component differently.
- The function version may look more foreign inside a component function as it's not JSX and as a result may be more difficult to teach. 
- The component version doesn't need any special APIs for passing in props like the React `key`, as it's meant to be rendered like any other component.

In the current state, the function API is a _lot_ more ergonomic to use, but its advantages over the component API may be short lived if the pattern can gain adoption and influence projects like prettier to implement special case support for it. But at the same time, lack of support in prettier may hinder early adoption if we went with the component API, creating a chicken-and-egg situation.

Not implementing this simply means we continue with the status quo where users are forced to create new components every time they want to use a hook in a place that would violate rules of hooks.

# Adoption strategy

There are no breaking changes involved in this proposal as it's a completely new pattern. The pattern can be adopted in a grassroots fashion as people find compelling use cases for it.

# How we teach this

As mentioned in the alternatives section, I've gone through a few iterations of naming, and am currently happy with the latest terminology of "anonymous components" for the pattern, and "Anonymous" for the name of the component/function APIs. 

The name draws parallels to anonymous functions, which is a useful analogue for building the mental model, as with the introduction of this pattern, we now have both named and anonymous components, and anonymous ones don't have to be explicitly created before they are used, and can be used inline within other component functions, unlike named components. Though it's not a perfect analogy as under the hood, the anonymous component is really just the same named component that's being re-used over and over again with different implementations, rather than created on the spot at usage as is the case with anonymous functions.

I feel a good way to teach this could be to introduce it in a standalone guide describing its various use cases, and linking to the guide in places in the existing docs where we discuss rules of hooks by mentioning how anonymous components can help. My experience in this area is definitly lacking though, so would love to get ideas/thoughts from the more seasoned educators on the React team and in the community.

# Unresolved questions

1. Does this pattern pose any implications for concurrent mode and/or server components that I may have missed?

2. Are there ways to implement this as a special first-class primitive that can work around edge cases like the one below, and potentially have performance/debugging benefits over the userspace version?

    ```js
    isOpen
      ? anonymous(() => useMemo(() => "yes", []))
      : anonymous(() => useMemo(() => "no", []))
    ```

3. I'd love some help brainstorming use cases for this pattern outside of hooks.

    Here's one that I've discovered involving contexts:

    ```js
    import * as React from 'react'
    import anonymous from "@lewisl9029/react-anonymous";

    const themeContext = React.createContext()

    const Example = () => {
      return (
        <themeContext.Provider value={{ foreground: 'black', background: 'white' }}>
          {anonymous(() => {
            // Calling useContext(themeContext) outside of anonymous would result in undefined.
            // 
            // Since only components downstream from the component where the Provider 
            // is rendered can read from the Provider.
            const theme = React.useContext(themeContext)
            return (
              <span className={
                useStyles(
                  () => ({ 
                    color: theme.foreground, 
                    backgroundColor: theme.background 
                  }), 
                  [theme.background, theme.background]
                )}
              >
                themed text
              </span>
          })}
        </themeContext.Provider>
      )
    }
    ```

  