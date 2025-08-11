# 激活方法 PowerShell
1. **打开 PowerShell**  
   单击“开始”菜单，键入 ，然后将其打开。

2. **复制并粘贴下面的代码，然后按 Enter。**  
   - 对于 **Windows 8, 10, 11**: 📌
     ```
     irm https://get.activated.win | iex
     ```
   - 对于**Windows 7** 及更高版本:
     ```
     iex ((New-Object Net.WebClient).DownloadString('https://get.activated.win'))
     ```
