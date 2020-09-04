---
title: express parameter router skill
date: 2020-09-02 12:22:49
tags: 
- 技巧
---

### 1. Problem: router path is used as parameter

If we have both parameter router and normal router with same path structure, the normal router path will be used as a url parameter.
```
r.route('PUT', '/bind')
r.route('PUT', '/unbind')
r.route('PUT', '/:deviceId')
```

The above `bind` and `unbind` will be fetch and check in our middware, our middware check the parameter value type and throw error if the type is not number.

We get the url parameters by express standard interface, so we can not fix the bug in our middware. We only can tell express, please do not act noraml path as parameter in url. How to do?

### 2. Solution: use exact path rule
we can deinfe the parameter route with more exact path:
```
r.route('PUT', '/:deviceId(\\d+)')
```

### 3. Reference
- [Routing](http://expressjs.com/en/guide/routing.html)