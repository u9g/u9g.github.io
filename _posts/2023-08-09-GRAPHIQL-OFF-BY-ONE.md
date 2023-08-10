layout: post
title: "Fixing an off-by-one in GraphiQL"
date: 2023-08-09 22:22:00 -0000

[The Pull Request](https://github.com/graphql/graphiql/pull/3397)

This pr causes console spam in our case for when we hover a comment on the first line. See [https://github.com/obi1kenobi/trustfall/issues/285](https://github.com/obi1kenobi/trustfall/issues/285) for more context.

What does this pr do?

Fixes an off-by-one bug that caused console spam when hovering things in the first line of a graphql file. This occurs because monaco's `toGraphQLPosition()` from `monaco-graphql` returns 0-based line/col numbers and `getRange()` from `graphql-language-service` expects 1-based line/col numbers.

Why is this the right fix?
We have a 0-indexed row/col. To see this for myself, I put a `console.log()` [after this line](https://github.com/graphql/graphiql/pull/3397/files#diff-864a2605362ad715bfa1b09cd9cbf18396a38b11c4a09b28a05dd1e831aed2aaR87). I then hovered the first character on the first line. I see 0,0 printed, and this is after we subtract 1 from both line/col in `toGraphQLPosition()`, so we definitely have a 0-indexed number.

`getRange()` takes in a `SourceLocation`, however that's an interface, so it's not really obvious whether the line/col is 0-based or 1-based. However, there is a function `getLocation()` that returns a `SourceLocation` declared in the same `location.d.ts` from within the `graphql/language` repo. So taking a look at their `getLocation()` function, it looks like this:
```js
function getLocation(source, position) {
  let lastLineStart = 0;
  let line = 1;

  for (const match of source.body.matchAll(LineRegExp)) {
    typeof match.index === 'number' || (0, _invariant.invariant)(false);

    if (match.index >= position) {
      break;
    }

    lastLineStart = match.index + match[0].length;
    line += 1;
  }

  return {
    line,
    column: position + 1 - lastLineStart,
  };
}
```

When I saw that, it became clear that this line is 1-indexed, and the `+ 1` in the column leads me to believe this number is 1-based too.
