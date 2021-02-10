- Start Date: 2021-02-09
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

This RFC introduces a new pattern/primitive, the `component` function (final naming TBD). This function creates an inline component boundary for users to call hooks within, at any arbitrary level of nesting within a tree of React elements.

Essentially, this RFC aims to make the following modifications to the now infamous Rules of Hooks to provide better developer ergonomics by making hook usage slightly less restrictive:

> ## ~~Only Call Hooks at the Top Level~~ Only Call Hooks at the Top Level of Component Boundaries
> 
> Don’t call Hooks inside loops, conditions, or nested functions. Instead, always use Hooks at the top level of ~~your React function~~ a component boundary created by your React function or a call to the `component` function. By following this rule, you ensure that Hooks are called in the same order each time a component renders. That’s what allows React to correctly preserve the state of Hooks between multiple useState and useEffect calls. (If you’re curious, we’ll explain this in depth below.)

(TODO: could probably figure out some better wording for this to make it easier to understand)

# Basic example

Taken from my proof of concept in userspace: https://github.com/lewisl9029/render-hooks

Examples are slightly contrived to fit everything into 1 component, but hopefully it showcases all the new possibilities adequately.

```js
import * as React from "react";
import component from "@lewisl9029/render-hooks";

const ImagineIfYouCould = ({ colors, loading }) => {
  if (loading) {
    // Call hooks within an early return
    return component(() => React.useContext(LoadingMessageContext));
  }

  return component(() => {
    // Call hooks after an early return
    const [isOpen, setIsOpen] = React.useState(false);

    return (
      <div>
        {isOpen
          ? // Call hooks within an inline branching path
            component(() => (
              <Modal
                close={React.useCallback(() => setIsOpen(false), [setIsOpen])}
              >
                <ul>
                  {colors.map((color) =>
                    // Call hooks within a mapping function
                    component(() => (
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

Each of the `component` calls in the above example would have had to be broken into a separate component without this pattern, due to the rules of hooks, requiring 4 additional levels of forced indirection. More concretely, this translates into 4 extra components to name, 4 extra layers of props to drill, 4 extra sets of prop types to define, etc. 

It also allows us to much closer colocate return values of hook calls to their usages, in many cases eliminating the need for an intermediate variable completely (i.e. the `close={React.useCallback(() => setIsOpen(false), [setIsOpen])}`).

The `component` function can drastically reduce the amount of friction involved with using hooks on a day to day basis.

# Motivation

Compared to HOCs, it strictly removes indirection, compared to render props it actually strictly increases indirection.

Hooks were originally 
Colocate definition and usage
A lambda component, if you will
Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?