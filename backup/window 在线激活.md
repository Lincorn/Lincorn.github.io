## 激活方法 PowerShell
1. **打开 PowerShell**  
   单击“开始”菜单，键入 ，然后将其打开。

2. **复制并粘贴下面的代码，然后按 Enter。**  
   - 对于 **Windows 8, 10, 11**：📌如果上述设置被ISP/DNS阻挡，可以试试这个（需要更新Windows 10或11）
     ```
     irm https://get.activated.win | iex
     ```
     ```
     iex (curl.exe -s --doh-url https://1.1.1.1/dns-query https://get.activated.win | Out-String)
     ```
   - 对于**Windows 7** 及更高版本：
     ```
     iex ((New-Object Net.WebClient).DownloadString('https://get.activated.win'))
     ```
