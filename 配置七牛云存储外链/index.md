# 配置七牛云存储外链


今天重新整理了下博客，发现之前配置好的七牛云图片显示错误。重新配置的同时也记录下配置步骤。

# 添加域名
登录七牛云的官网[融合CDN](https://portal.qiniu.com/cdn/domain)，在“域名管理”中“添加域名”，如图：

![](http://image.xingyys.club/blog/添加域名.png)

填写“加速域名”字段为`image.xingyys.club` 其他默认。

# 解析DNS
在添加完域名后，直接会在后台审核。

![](http://image.xingyys.club/blog/七牛云域名.png)

鼠标停在表格上获取CNAME，`image.xingyys.club.qiniudns.com`。之后就是在域名后台上添加解析。我的是腾讯云的域名，添加一条CNAME解析：

![](http://image.xingyys.club/blog/腾讯域名.png)

1.主机记录为二级域名。
2.记录类型为CNAME。 
3.线路类型默认。
4.记录值为刚才获取的七牛云CNAME。

然后保存即可。

# 绑定存储空间

在七牛云的控制台，“对象存储”里“新建存储空间”

![](http://image.xingyys.club/blog/存储空间.png)

最后在新建的存储空间里面在绑定刚添加的二级域名：

![](http://image.xingyys.club/blog/内存管理.png)

配置完成。
