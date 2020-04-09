---
title: thinking about retry
date: 2020-04-09 16:04:30
tags:
- 代码
- 技巧
---

### 1. What is retry?

`Retry` is way for code to process the expection. If you develop with javascrpt, too many tools support retry, for example:
- [retry-axios](https://github.com/JustinBeckwith/retry-axios)
- [bluebird-retr](https://github.com/demmer/bluebird-retry)
- [async](http://caolan.github.io/async/v3/docs.html#retry)

Besides all above, we can implement retry by recursion or iteration method. `Retry` can help catch the expection and run same logic for multiple times.

### 2. Thinking about retry

`Retry` is so nice that many developer to use it everywhere. But something you should know.

#### 2.1 retry is not silver bullet

If a logic has error, we should fix the bug but not retry. Retry maybe make you lazzy to think the reason of the error.

#### 2.2 retry need some basic requirement
```
// the function can not do retry
function demov1(from, to) {
    fs.renameSync('./package.json', './tmp/package.json')
    ... other logic
}

// the fuction can do retry
function demov2(from, to) {
    fs.copyFile('./package.json', './tmp/package.json')
    ... other logic
}
```
Above functions just a demo, it is not the real logic, but it is enough to explain my thoughts.

`demov1` can not support retry because if the expection is thrown in `other logic`. At that time the source file moving is done, so the retry will always failure. But in the `demov2`, the source file exists.

Above all, the retry target logic must have some requirement for retry. `Retry` is not easy to use.

#### 2.3 retry nest

you write your module with retry, another developer finish his module also using retry. If the one module calls another one. The retry nest will appear. At end, you code maybe look like:
```
retry(()=>{
    retry(()=>{}, 5)
}, 5)
```
I do think the retry nest is good design.

#### 2.4 drawback
- hide real bug
- if you do some aggregation, retry will let the aggregation result not right


### 3. Suggestion
- use retry as less as possible
- think about the retry context, to avoid retry nest and make you code retryable
