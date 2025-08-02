## 说明
本项目改进于： https://github.com/xiaoziguys/pikpakHelpr

具体修改和新增了如下特性：
1. 修复了原仓库中由于官方接口改变导致不可用的情况；
2. 改良了获取当前目录下文件列表的实现方式，原仓库是通过js获取dom元素，本项目则是采用调用官方接口实现
3. 支持了文件夹的下载

安装地址：https://greasyfork.org/zh-CN/scripts/503530-pikpak%E5%8A%A9%E6%89%8Bplus

## 代理相关

pikpak的下载链接并没有被墙，所以并不需要走代理，这样可以大量节省机场的流量，分享一下我的clash配置如下（建议将aria2的的http代理设置为代理软件的端口，如`http://127.0.0.1:7890`，否则配置可能不会生效，具体请自行测试）：

```
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

经[@AKiSA07](https://github.com/AKiSA07)在[issue](https://github.com/jdysya/pikpakHelpr-plus/issues/4)中提醒：

建议将原规则的最后两行

```
    - 'DOMAIN,api-drive.mypikpak.com,DIRECT'
    - 'DOMAIN-KEYWORD,dl-a10b-,DIRECT'
```

更新为

```
    - 'DOMAIN,api-drive.mypikpak.com,[机场名]'
    - 'DOMAIN-KEYWORD,dl-a10b-,DIRECT'
    - 'DOMAIN-KEYWORD,dl-z01a-,DIRECT'
    - 'DOMAIN-KEYWORD,dl-b07-,DIRECT'
```

可解决文件转存失败的问题，以及新增了部分直链下载域名关键词

如果代理软件支持正则，则推荐

```
/dl-(?:a10b|z01a|b07)-/
```



### 配置
安装后刷新在新建按钮旁边会出现一个aria2配置按钮，点击配置
- 服务器 `http(s)://host:port/jsonrpc`
- 路径 aria2的下载路径 `/downloads`
- 密钥 aria2的密钥

