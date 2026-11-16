# 语言栈 101

汇编、编译器、运行时与主流语言实现的中文笔记站。版式对齐 [Linux 101](https://101.lug.ustc.edu.cn/) 的 MkDocs Material（indigo）。

## 本地预览

```bash
cd langstack-101
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

浏览器打开提示的本地地址即可。

## 目录结构

- `docs/` — 正文（按章 `ch01`…`ch06`）
- `docs/Spec/` — 大纲与口试量规
- `docs/appendix/writing-spec.md` — 写作规范
- `docs/assets/svg/` — Simple Icons 等图标

## 能力标准

见 `docs/Spec/oral-rubric.md`：能与导师把 ABI、UB、链接、内存序、派发与基准读法谈清楚。
