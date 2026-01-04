好的，相信你已经根据上一节的教材正确安装了 `Python` 和 `VSCode`

**本节教程将帮助你完成 `VSCode` 软件的配置操作，方便你进行开发和学习**

-----------------

## 安装中文语言包和Python插件
如果你是按照 `初识Python` 教程安装的VSCode，你应该可以在桌面上找到VSCode快捷方式，双击打开
[caption id="attachment_1703" align="aligncenter" width="921"]<img src="https://esaps.net/wp-content/uploads/2024/10/START-VSC-1024x641.webp" alt="" width="921" height="577" class="size-large wp-image-1703" /> VSCode启动后初始画面[/caption]

我们看向左边栏目，点击扩展按钮（就是有4个正方形那个，右上角的正方形没连在一起那个）
或者按 `{ctrl}` + `{shift}` + `{x}` 直接打开扩展界面

在扩展搜索框内搜索Chinese，然后安装此扩展
[caption id="attachment_1704" align="aligncenter" width="921"]<img src="https://esaps.net/wp-content/uploads/2024/10/Chinese-VSC-1024x641.webp" alt="" width="921" height="577" class="size-large wp-image-1704" /> 中文语言包，请按照扩展指示更改语言[/caption]

在扩展搜索框内搜索Python，然后安装此扩展
[caption id="attachment_1705" align="aligncenter" width="921"]<img src="https://esaps.net/wp-content/uploads/2024/10/Python-VSC-1024x641.webp" alt="" width="921" height="577" class="size-large wp-image-1705" /> VSCode Python语言支持扩展[/caption]

至此，扩展安装完成

-----------------

## 添加项目文件夹
你可以先在你喜欢的地方创建一个文件夹，比如 `Project_exercises`

[caption id="attachment_1701" align="aligncenter" width="227"]<img src="https://esaps.net/wp-content/uploads/2024/10/OpenFolder-VSC-227x300.webp" alt="" width="227" height="300" class="size-medium wp-image-1701" /> VSCode添加工作区[/caption]
按照上方图片指示添加工作区，选择你刚刚创建好的文件夹这时，你应该就能看到你的VSCode项目目录中（左边栏上方文件图标，`{ctrl}` + `{shift}` + `{e}`）出现你刚刚选择的文件夹了

此时，对准刚刚出现的文件夹按一下右键，选择 `新建文件`，名称可以自己选择，在这里我输入的名称为 `HelloPython.py`。
右边会出现一个文件编辑窗口，在之后我们都会在这个窗口里写代码。
输入以下代码
```python
print("Hello Python~!")
```
输入完成后，按 `{ctrl}` + `{s}` 保存文件。VSCode右上方有个小三角形，点击这个三角形运行你刚刚写好的Python代码
[caption id="attachment_1702" align="aligncenter" width="921"]<img src="https://esaps.net/wp-content/uploads/2024/10/RUN-VSC-1024x640.webp" alt="" width="921" height="577" class="size-large wp-image-1702" /> 正常运行的画面[/caption]
如果输出结果和上图一致，恭喜！VSCode已经正确配置！继续投身进你的Python世界吧！

### 可能的错误：
 - 如果无法运行，你首先需要确保你的系统已经正确安装Python
 - 如果VSCode右下方有一个 `选择解释器` 的黄色图标，点击一下他，在弹出的窗口内选择你安装的Python即可重新运行
