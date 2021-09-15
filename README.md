## 1. 使用submodule方式加载主题

```
# 添加适合的主题
git submodule add https://github.com/sanonz/hexo-theme-concise.git ./themes/concise

# 配置主题的分支(版本)
git config -f .gitmodules submodule.themes/concise.branch master

# 配置git pull自动拉取子模块
git config submodule.recurse true
```

除此之外, 需要把配置安装到主题中, 主题配置放在themes目录根目录, 然后copy到主题中, 所以修改package script如下:
```
"scripts": {
    "build": "npm run cp && hexo generate",
    "clean": "hexo clean",
    "deploy": "npm run cp && hexo deploy",
    "server": "npm run cp && hexo server",
    "cp": "cp themes/_config.yml themes/concise/"
  },
```
