1. 进入下载页面: [vscode](https://code.visualstudio.com/Download#)
2. 选择windows平台下的zip x64 
3. 下载完成后，将zip包中的内容解压到D:\vscode中
4. 在d:\bin中创建文件code.cmd, 其中内容如下
```cmd
@echo off
setlocal
set code_path=d:\vscode
set code_data=f:\vscode
set VSCODE_DEV=
set ELECTRON_RUN_AS_NODE=1
"%code_path%\Code.exe" "%code_path%\resources\app\out\cli.js"  --user-data-dir="F:\vscode\user-data" --extensions-dir="F:\vscode\extensions" %*
endlocal
```
5. 在d:\bin中创建文件vscode.vbe，其中内容如下

```vbe
set ws=wscript.createobject("wscript.shell")
ws.run "d:\bin\code.cmd" , 0
```
6. 创建快捷方式
7. 右键点击快捷方式，选择属性
8. 选择【快捷方式】选项卡，将目标修改为d:\bin\vscode.vbe
9. 在快捷方式选项卡，点击【更改图标】按钮，点击【浏览】，选择Code.exe，点击【打开】，点击【确定】，点击【确定】
10. 将快捷方式移动到桌面
11. 双击快捷方式，打开vscode。在任务栏上右键点击vscode图标，点击【固定到任务栏】
12. 右键点击任务栏上的vscode图标，再右键点击弹出菜单中的【Visula Studio Code】，选择【属性】打开窗口
13. 选择快捷方式选项卡，按照前面8，9步骤中设置。
14. 更新是下载新的zip包，解压到D:\vscode即可
