## 说明
本项目改进于： https://github.com/xiaoziguys/pikpakHelpr

具体修改和新增了如下特性：
1. 修复了原仓库中由于官方接口改变导致不可用的情况；
2. 改良了获取当前目录下文件列表的实现方式，原仓库是通过js获取dom元素，本项目则是采用调用官方接口实现
3. 支持了文件夹的下载

安装地址：https://greasyfork.org/zh-CN/scripts/503530-pikpak%E5%8A%A9%E6%89%8Bplus

## 代理相关

建议将aria2的的http代理设置为代理软件的端口，如`http://127.0.0.1:7890`，否则可能下载没有速度

下列clash配置请自行测试，若可正常下载则说明下载链接被没有被墙，此时下载不会消耗机场流量。若无法下载请不要使用下列配置。

```yml
rules:
    - 'DOMAIN,mypikpak.com,[机场名]'
    - 'DOMAIN,mypikpak.net,[机场名]'
    - 'DOMAIN,user.mypikpak.com,[机场名]'
    - 'DOMAIN,access.mypikpak.com,[机场名]'
    - 'DOMAIN,api-drive.mypikpak.com,[机场名]'
    - 'DOMAIN-KEYWORD,dl-a10b-,DIRECT'
    - 'DOMAIN-KEYWORD,dl-z01a-,DIRECT'
    - 'DOMAIN-KEYWORD,dl-b07-,DIRECT'
```

### 配置
安装后刷新在新建按钮旁边会出现一个aria2配置按钮，点击配置
- 服务器 `http(s)://host:port/jsonrpc`
- 路径 aria2的下载路径 `/downloads`
- 密钥 aria2的密钥

