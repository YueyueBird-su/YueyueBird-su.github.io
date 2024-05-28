---
layout:     post
title:      "Zotero read notes template"
subtitle:   "论文笔记阅读模板"
date:       2024-05-28 22:18:30
author:     "LpengSu"
header-img: https://raw.githubusercontent.com/YueyueBird-su/Pitcture_Git/main/images/image-20240528225048199.png
catalog: true
tags:
    - 经验总结
    - Zotero技巧
---

# Zotero read notes template

在本文档中，提交成功后会出现Liquid语法错误，需要用以下Liquid代码块escape

**{% raw %}
...
{% endraw %}**

## 一、论文笔记模板

进行了一些修改，摘要同时保留英文和翻译的内容。

```xml
{% raw %}
<!-- 标题 -->
<h1 style="color:#193c47; background-color:#eef9fd; padding:8px;">
  ${(() => {
    const titleTranslation = topItem.getField("titleTranslation");
    if (titleTranslation) {
      return `(${topItem.getField("date")}) ${topItem.getField("title")} (${titleTranslation})`;
    } else {
      return `(${topItem.getField("date")}) ${topItem.getField("title")}`;
    }
  })()}
</h1>

<!-- Meta Data -->
<span>
  <h2 style="color:#1b5e20; background-color:#f1f8e9;">💡 Meta Data</h2>
</span>

<table>
  <!-- 作者 -->
  <tr>
    <td style="color:#193c47; background-color:#dbeedd; padding:8px;">
      <b>作者:</b> ${topItem.getCreators().map((v) => v.firstName + " " + v.lastName).join("; ")}
    </td>
  </tr>

  <!-- 期刊 -->
  <tr>
    <td style="color:#193c47; background-color:#f3faf4; padding:8px;">
      <b style="color:#193c47;">期刊: <b style="color:#FF0000">${topItem.getField('publicationTitle')}</b></b><b style="color:#193c47;"> （发表日期: ${topItem.getField("date")}）</b>
    </td>
  </tr>

  <!-- 本地链接 -->
  <tr>
    <td style="color:#193c47; background-color:#f3faf4; padding:8px;">
      <b>本地链接: </b>
      <a href=zotero://open-pdf/0_${Zotero.Items.get(topItem.getAttachments()).filter((i) => i.isPDFAttachment())[0].key}>
        ${Zotero.Items.get(topItem.getAttachments()).filter((i) => i.isPDFAttachment())[0].getFilename()}
      </a>
    </td>
  </tr>
  
  <!-- DOI or URL -->
  <tr>
    <td style="color:#193c47; background-color:#dbeedd; padding:8px;">
      ${(() => {
        const doi = topItem.getField("DOI");
        if (doi) {
          return `<b>DOI: </b><a href="https://doi.org/${topItem.getField('DOI')}">${topItem.getField('DOI')}</a>`;
        } else {
          return `<b>URL: </b><a href="${topItem.getField('url')}">${topItem.getField('url')}</a>`;
        }
      })()}
    </td>
  </tr>
  
  <!-- 摘要 -->
  <tr>
    <td style="color:#193c47; background-color:#f3faf4; padding:8px;">
      <b>摘要: </b><i>${topItem.getField('abstractNote')}</i>;
    </td>
  </tr>
  <tr>
    <td style="color:#193c47; background-color:#f3faf4; padding:8px;">
      ${(() => {
        const abstractTranslation = topItem.getField('abstractTranslation');
        return `<b>摘要翻译: </b><i>${abstractTranslation}</i>`;
      })()}
    </td>
  </tr>

  <!-- 笔记日期 -->
  <tr>
    <td style="color:#193c47; background-color:#dbeedd; padding:8px;">
      <b>笔记日期: </b>${new Date().toLocaleString()}
    </td>
  </tr>

</table>

<!-- 正文 -->
<span>
  <h2 style="color:#e0ffff; background-color:#66cdaa;">📜 研究核心</h2>
  <hr />
</span>
<blockquote>Tips: 做了什么，解决了什么问题，创新点与不足？</blockquote>
<p></p>
<h3>⚙️ 内容</h3>
<p></p>
<h3>💡 创新点</h3>
<p></p>
<h3>🧩 不足</h3>
<p></p>

<span>
  <h2 style="color:#20b2aa; background-color:#afeeee;">🔁 研究内容</h2>
  <hr />
</span>
<p></p>
<h3>💧 数据</h3>
<p></p>
<h3>👩🏻‍💻 方法</h3>
<p></p>
<h3>🔬 实验</h3>
<p></p>
<h3>📜 结论</h3>
<p></p>

<span>
  <h2 style="color:#004d99; background-color:#87cefa;">🤔 个人总结</h2>
  <hr />
</span>
<blockquote>Tips: 你对哪些内容产生了疑问，你认为可以如何改进？</blockquote>
<p></p>
<h3>🙋‍♀️ 重点记录</h3>
<p></p>
<h3>📌 待解决</h3>
<p></p>
<h3>💭 思考启发</h3>
<p></p>
{% endraw %}
```

## 二、论文批注笔记模板

按照颜色对论文笔记中的批注进行整理，形成笔记

方便对生词、短句的学习

```xml
{% raw %}
${{
  async function getAnnotationsByColor(item, colorFilter) {
    const annots = item.getAnnotations().filter(colorFilter);
	return await Zotero.BetterNotes.api.convert.annotations2html(annots, { noteItem: targetNoteItem });
  }

  const attachments = Zotero.Items.get(topItem.getAttachments()).filter((i) =>
    i.isPDFAttachment() || i.isSnapshotAttachment() || i.isEPUBAttachment()
  );
  let res = "";
  const colors = ["#ffd400", "#ff6666", "#5fb236", "#2ea8e5", "#a28ae5", "#e56eee", "#f19837", "#aaaaaa"];
  const colorNames = ["Yellow", "Red", "Green", "Blue", "Purple", "Magenta", "Orange", "Gray"];
  for (let attachment of attachments) {
    res += `<h1>${attachment.getFilename()}</h1>`;
    for (let i in colors) {
      const renderedAnnotations = (
        await getAnnotationsByColor(
          attachment,
          (annot) => annot.annotationColor === colors[i]
        )
      );
      if (renderedAnnotations) {
        res += `<h2><p style="background-color:${colors[i]};">${colorNames[i]} Annotations</p></h2>\n${renderedAnnotations}`;
      }
    }
    const renderedAnnotations = (
      await getAnnotationsByColor(
        attachment,
        (annot) => !colors.includes(annot.annotationColor)
      )
    );
    if (renderedAnnotations) {
      res += `<h2><p>Other Annotations</p></h2>\n${renderedAnnotations}`;
    }
  }
  return res;
}}$
{% endraw %}
```
