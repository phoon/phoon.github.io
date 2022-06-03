# 使用Certbot签发免费泛域名SSL证书


服务器环境：



> Archlinux
> Nginx1.15.6（自行编译）

（1）安装Certbot
```bash
sudo pacman -S certbot
```
（2）开始签发
因为我的Nginx是自行编译的，所以无法使用certbot的nginx plugin,我选择手动验证dns来进行签发
```bash
sudo certbot certonly  --preferred-challenges dns -d "*.example.com" -d example.com --manual
```
在这里，我指定了两个-d参数，因为certbot的泛域名其实只支持诸如*.example.com，如要直接使用顶域，就得为其单独也签发一条。
回车之后，certbot会让你去给你的域名添加2条TXT记录，名称为_acme-challenge.example.com（TXT记录可重复）

（3）检查解析记录
在添加完上述TXT记录后，在另一个终端输入：

```bash
nslookup -type=txt _acme-challenge.example.com 8.8.8.8
```
如果返回结果中有你添加的两条TXT记录值，那么说明解析已经生效了。这个时候再返回签发状态的certbot所在终端，回车，不出意外的话，certbot会在输出结果里打印出证书签发好之后存放的位置，找到它们，就OK了。
