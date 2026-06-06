# PPT Skill

帮助用户基于 Marp + Awesome-Marp 制作 PPT。`##` 二级标题表示一页幻灯片。

## 固定不修改的页面

以下三类页面结构固定，**只改文字内容，不改 class/header**：

**封面页**（`cover_b` / `cover_c` 等）
**目录页**（`toc_b` 等）
**最后一页**：
```markdown
---
<!-- _class: lastpage  -->
<!-- _header: ![#l h:40](../images/logo.png)-->
###### Thank you! Q & A 
<div class = "icons">
</div>
```

## 导航栏规则（@Now 工作汇报）

每张内容页必须有 navbar header，当前节**加粗**，其余斜体：

```markdown
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **当前节** *其他节* *其他节* -->
```

### 无图片页（纯文字）

```markdown
## N. 节名 & 页面标题 <!-- page X -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **当前节** *节2* *节3* *节4* *节5* -->
<!-- _class: navbar -->

内容...
```

### 有图片页（图文分栏）

```markdown
## N. 节名 & 页面标题 <!-- page X -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *节1* **当前节** *节3* *节4* *节5* -->
<!-- _class: navbar cols-2-64-->
<div class="ldiv">

文字内容...

</div>
<div class="rdiv">

![#c](assets/folder/image.png)
> 💡 关键洞察：...

</div>
```

分栏比例选择：`cols-2`（1:1）、`cols-2-64`（6:4）、`cols-2-73`（7:3）、`cols-2-46`（4:6）、`cols-2-37`（3:7）

## @Seminar 组会模板

结构通常为：Introduction → Related Work → Motivation → Design → Evaluation → Conclusion

同样使用 navbar，节名对应组会各章节。

## 页码注释

每页标题行末尾写 `<!-- page X -->`，页码从封面后的第一张内容页开始（通常 page 3）。

## 文件命名与存放

- 工作汇报 / 论文分享 → `Now/YYYY-MM-DD-topic.md`
- 组会 → `Seminar/YYYY-MM-DD-topic.md`

## 使用方式

用户说 `/ppt` 后，询问：
1. 工作汇报还是组会分享？
2. 主题 / 论文名？
3. 节的划分（用于 navbar）？

然后生成完整 `.md` 文件。
