---
title: markdown文件转pdf
date: 2021-08-26 18:09:17
tags:
 - 工具
 - 技巧
---

### 1. 方案: marked + wkhtmltopdf

两个工具简化成一个方法如下:
```bash
function mk2pdf() {
  if [[ ! -f $1 ]]; then
    echo "Please give a markdown file."
    echo "eg: mk2pdf ./source.md"
    return
  fi

  marked $1 | wkhtmltopdf --footer-spacing 2 --footer-right '[page]/[topage]' --footer-line --footer-font-size 10 --footer-center 'Generate by Jimmy' --quiet --encoding utf-8  - "$1.pdf"

  echo "PDF: $1.pdf"
}
```

### 2. 参考文档
- [关于 wkhtmltopdf：html 转 pdf 的命令行工具](https://segmentfault.com/a/1190000018988358)
- [wkhtmltopdf专成pdf文件之后的页码显示问题](https://www.codeleading.com/article/2798190256/)
- [How to convert Markdown to PDF without writing an intermediate HTML file?](https://stackoverflow.com/questions/60458709/how-to-convert-markdown-to-pdf-without-writing-an-intermediate-html-file)