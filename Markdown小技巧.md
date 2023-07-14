# Markdown 中插入空格
```markdown
&emsp;          全角空格
&ensp;          半角空格
&nbsp;          不换行空格
&thinsp;        细小空格
```
例1：

	我&emsp;他

显示效果为

---

**我&ensp;他**

---

例2：

	我&emsp;他

显示效果为

---

**我&ensp;他**

---

例3：

	我&nbsp;他

显示效果为

---

**我&nbsp;他**

---

例4：

	我&thinsp;他

显示效果为

---

**我&thinsp;他**

---

# Markdown 中插入空行
```markdown
<br />        方式1
&nbsp;        方式2
```

例1：

	我<br /> 
	他

显示效果为

---

**我<br /> 
他**

---

例2：

	我
	&nbsp;
	他

显示效果为

---

 **我
&nbsp;
他**

---

# Markdown 中文字左、中、右对齐
```html
<center>Markdown</center>

<p align = "left">Markdown</p>

<p align="right" >Markdown</p>

```

显示效果为

---

<center>Markdown</center>

<p align = "left">Markdown</p>

<p align="right">Markdown</p>

---

# 上下角标

```markdown
H<sub>2</sub>O
x<sup>2</sup>+y<sup>2</sup>
```

显示效果为

---

H<sub>2</sub>O

x<sup>2</sup>+y<sup>2</sup>

# <span id="jump">文章内跳转</span>

1. 定义一个锚点 ` <span id="jump">跳转到的地方</span>`
2. 跳转语法 ` [点击跳转`](#jump)

示例：

跳转至[文章内跳转](#jump)
