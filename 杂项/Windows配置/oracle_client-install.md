1. 下载oracle instant client  x86和x64版本都要下载
    [x86下载页面](https://www.oracle.com/database/technologies/instant-client/microsoft-windows-32-downloads.html)
    [x64下载页面](https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html)
    在页面中找到 Version 12.2.0.1.0，下载一下软件包：Basic Package，SQL*Plus Package，Tools Package，SDK Package，JDBC Supplement Package，ODBC Package

2. 将x86版本的压缩包解压到当前文件夹，得到instantclient_12_2
3. 修改instantclient_12_2文件夹名称为client_x86,将client_x86移动到D:\oracle中
4. 按照2，3两步将x64版本的zip包解药到D:\oracle\client_x64
5. 创建F:\oracle,并在其中创建tnsnames.ora文件
7. 添加系统环境变量ORACLE_HOME=D:\oracle\client_x64
8. 添加系统环境变量TNS_ADMIN=F:\oracle
