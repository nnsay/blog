---
title: AppImage Auto-Upgrade Failure Because of APPIMAGE NO Found
date: 2020-04-08 22:19:44
tags:
- 工作
- Linux
- 研究
excerpt: troubleshoot the electrol appImage application auto upgrade failure
---

### 1. What hanppened?

We build a cross-platform application with electron. But its auto upgrade feature does not works. In order to explain, we image we have build a desktop application: app.AppImage. When we start the application in terminal and check the latest the version, there are some output information look like below:

```
Error: Error: APPIMAGE env is not defined
    at t.newError (/tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:1:12502)
    at Object.task (/tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:142:178326)
    at h.executeDownload (/tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:1:254868)
    at async /tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:315:180714
    at async EventEmitter.Hr (/tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:315:180675)
09:30:53.222 › Cannot dispatch error event: TypeError: Error processing argument at index 1, conversion failure from Error: APPIMAGE env is not defined
    at Object.showErrorBox (/tmp/.mount_no-app2Kmp8v/resources/electron.asar/browser/api/dialog.js:163:24)
    at h.<anonymous> (/tmp/.mount_no-app2Kmp8v/resources/app.asar/dist/main.js:315:179401)
```
You will more detail inforamtion from output if you start with a debug flag like below 
```
chmod +x app.AppImage
DEBUG=true ./app.AppImage
```
You will find the `APPIMAGE` environment parameter is existing. Yes, it is here, but you program can not find it.

### 2. Code checking & Research

#### 2.1 from research

- [APPIMAGE env is not defined ](https://github.com/electron-userland/electron-builder/issues/3167)
- [Best way to pass variables through Webpack?](https://github.com/PatrickJS/starter/issues/386)

According some github issue, we know some pulugin for example: `DefinePlugin` will reset the environment parameters. Maybe this is the reason.

#### 2.2 from code
```
...
const stringified = {
    'process.env': Object.keys(raw).reduce(
      (env, key) => {
        env[key] = JSON.stringify(raw[key]);
        return env;
      }, {}
    ),
  };
new webpack.DefinePlugin(stringified)
...
```
From the code, we find the we use the `DefinePlugin` to pass some environment parameter for building. And the important thing is that the whole `process.env` is rewrited.

### 3. How to fix?

We have find the reason and there two way to fix the bug:
- append `APPIMAGE` to the existing code
- only rewrite some process.env parameters, look like below:
```
const envParames = {};
  Object.keys(raw).forEach(k => {
    envParames[`process.env.${k}`] = JSON.stringify(raw[k])
  });
new webpack.DefinePlugin(envParames)
```
Because the `APPIMAGE` is a runtime environment, we can not set the value in building time. So the second it is right way to fix. Build a new application with new code, we test the `app.AppImage` auto upgrade function, it works now.

### 4. What have learned?

#### 4.1 there are two kind of environment parameter

- build time
- runtime

#### 4.2 never to rewrite whole process.env, only set the what you need

#### 4.3 Summary

if your code running some website, you maybe never to care about runtime environment parameter. But if you code will build to some package application eg: command line tools / desktop application, you should care about the runtime environment.

### 5. Other useful article
- [Type 2 AppImage runtime](https://docs.appimage.org/packaging-guide/environment-variables.html#id2): the APPIMAGE is a absolute path with symlinks resolved