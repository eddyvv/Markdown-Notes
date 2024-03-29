# vim操作

## vim的几种模式

**正常模式**：可以使用快捷键命令，或按`:`输入命令行。

**插入模式**：可以输入文本，在正常模式下，按`i`、`a`、`o`等都可以进入插入模式。

**可视模式**：正常模式下按`v`可以进入可视模式， 在可视模式下，移动光标可以选择文本。按`V`进入可视行模式， 总是整行整行的选中。`ctrl+v`进入可视块模式。

**替换模式**：正常模式下，按R进入。

## 光标移动

```bash
Ctrl+f					#向下翻一页
Ctrl+b					#向上翻一页
Ctrl+d					#向下半页
Ctrl+u					#向上半页
Ctrl+e					#屏幕向下滚动一行
Ctrl+y					#屏幕向上滚动一行
H,M,L					#移动至屏幕顶、中、底部
n						#敲数字后回车，光标向后移动n行
w,W 					#下一个词首
e,E 					#下一个词尾
0						#这一行的开头
^						#这一行的非空白开头
$						#这行的行尾
g_						#这行的非空白结尾
(						#句首
)						#句尾
{						#段首
}						#段尾
zz						#光标处字符不动，将当前行移动至屏幕居中
zt						#光标处字符不动，将当前行移动至屏幕顶部
zb						#光标处字符不动，将当前行移动至屏幕底部
ctrl+w n				#上下分屏，并在上屏开启一个新的文件
```

## 启动vim

```bash
vim -r file				#恢复上次异常退出的文件
vim -R file				#以只读方式打开文件，但可以强制保存
vim -M file				#以只读方式打开文件，不可以强制保存
vim + file				#从文件末尾开始
vim +num file			#从第num行开始
vim +/string file		#打开file文件，并将光标停留在第一个找到的string上
vim --remote file		#用已有的vim进程打开指定文件。
```

## 文档操作

```bash
:e file					#关闭当前编辑的文件，开启新的文件
:e! file				#放弃对当前文件的修改，编辑新的文件
:e + file				#开启新的文件。并从文件尾开始编辑
:e +n file				#开启新的文件，从第n行开始编辑
:enew					#编辑一个未命名的新文档
:e						#重新加载当前文档
:e!						#重新加载当前文档，并丢弃已做的改动
:e#或ctrl+^				#回到刚才编辑的文件
:f或ctrl+g				#显示文档名，是否修改和光标位置
gf						#打开以光标所在字符为文件名的文件
```

