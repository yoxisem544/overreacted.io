---
title: Testing File
date: '2019-08-30'
spoiler: 🚕🚕🚕🚕🚕🚕🚕🚕
---

How to solve: https://blog.oisulab.com/maybe-absolute-links/

```
[ { GraphQLError: Cannot query field "maybeAbsoluteLinks" on type "fields_4".
      at Object.Field (/Users/David/Desktop/overreacted.io/node_modules/graphql/validation/rules/FieldsOnCorrectType.js:65:31)
      at Object.enter (/Users/David/Desktop/overreacted.io/node_modules/graphql/language/visitor.js:324:29)
      at Object.enter (/Users/David/Desktop/overreacted.io/node_modules/graphql/language/visitor.js:366:25)
      at visit (/Users/David/Desktop/overreacted.io/node_modules/graphql/language/visitor.js:254:26)
      at visitUsingRules (/Users/David/Desktop/overreacted.io/node_modules/graphql/validation/validate.js:74:22)
      at validate (/Users/David/Desktop/overreacted.io/node_modules/graphql/validation/validate.js:59:10)
      at graphqlImpl (/Users/David/Desktop/overreacted.io/node_modules/graphql/graphql.js:106:50)
      at /Users/David/Desktop/overreacted.io/node_modules/graphql/graphql.js:66:223
      at Promise._execute (/Users/David/Desktop/overreacted.io/node_modules/bluebird/js/release/debuggability.js:313:9)
      at Promise._resolveFromExecutor (/Users/David/Desktop/overreacted.io/node_modules/bluebird/js/release/promise.js:483:18)
      at new Promise (/Users/David/Desktop/overreacted.io/node_modules/bluebird/js/release/promise.js:79:10)
      at graphql (/Users/David/Desktop/overreacted.io/node_modules/graphql/graphql.js:63:10)
      at graphqlRunner (/Users/David/Desktop/overreacted.io/node_modules/gatsby/dist/bootstrap/index.js:372:14)
      at Promise (/Users/David/Desktop/overreacted.io/gatsby-node.js:47:7)
      at Promise._execute (/Users/David/Desktop/overreacted.io/node_modules/bluebird/js/release/debuggability.js:313:9)
      at Promise._resolveFromExecutor (/Users/David/Desktop/overreacted.io/node_modules/bluebird/js/release/promise.js:483:18)
    message:
     'Cannot query field "maybeAbsoluteLinks" on type "fields_4".',
    locations: [ [Object] ],
```

Leave this page in your blog, then everything will be fine. Guess that you should need to have a at less one link like `maybeAbsoluteLinks` like below. Otherwise, keep this file.
```markdown
[this](/testing-file/)
```
