---
title: PlaySms 1.4代码执行漏洞分析
date: 2017-05-15 20:19:11
categories: exploit
tags:
	- web
	- php
---
PlaySms是一个灵活的基于Web的短信平台，可连接短信网关、个人信息系统以及企业的群组通讯工具等。本次爆出的漏洞是1.4版本的一个代码执行漏洞，该漏洞在exploit-db上有所收录，详情可参考https://www.exploit-db.com/exploits/42003/。

<!--more-->

# PlaySms 1.4代码执行漏洞分析
## 官方文案描述
- Exploit标题：PlaySMS 1.4 sendfromfile.php中使用`$filename`变量以及不严格的文件上传导致代码执行
- Date: 14-05-2017
- 软件链接：Software Link: https://playsms.org/download/
- 版本: 1.4
- Exploit作者：Touhid M.Shaikh
- 联系: http://twitter.com/touhidshaikh22
- 网站: http://touhidshaikh.com/
- 类别：webapps

1. 描述
- 不严格文件上传：
  - 任何注册的用户都可以上传任何类型的文件，因为在sendfromfile.php中没有任何的文件合法性检查。

- 使用`$filename`代码执行：
  - 我们知道了sendfromfile.php接受任何扩展名的文件，且仅仅只是读取内容而不会存储在server中。但当用户这样上传时依然有bug：服务器会欣然接受mybackdoor.php且不会存在任何一个目录中，因此我们的shell也就没什么卵用。但是如果用户将文件名从"mybackdoor.php"更改为"<?php system('uname -a'); dia();?>.php"，则服务器会检查该文件并将如是设置参数，`$filename="<?php system('uname -a'); dia();?>.php"`。你可以在页面上看到下面的代码并显示$filename。
  - 更多信息： www.touhidshaikh.com/blog/
2. POC
  作为一般用户登录（在index.php?app=main&inc=core_auth&route=register创建用户）：

去：http://127.0.0.1/playsms/index.php?app=main&inc=feature_sendfromfile&op=list

form如下：
```php
----------------------------Form for upload CSV file ----------------------
<form action=\"index.php?app=main&inc=feature_sendfromfile&op=upload_confirm\" enctype=\"multipart/form-data\" method=\"post\">
" . _CSRF_FORM_ . "
<p>" . _('Please select CSV file') . "</p>
<p><input type=\"file\" name=\"fncsv\"></p>
<p class=help-block>" . _('CSV file format') . " : " . $info_format . "</p>
<p><input type=checkbox name=fncsv_dup value=1 checked> " . _('Prevent duplicates') . "</p>
<p><input type=\"submit\" value=\"" . _('Upload file') . "\" class=\"button\"></p>
</form>
------------------------------Form ends ---------------------------

-------------PHP code for set parameter ---------------------------

	case 'upload_confirm':
		$filename = $_FILES['fncsv']['name'];

------------------------------php code ends ---------------------------
```

$filename应该在页面上可见：
```php
----------------------Vulnerable perameter show ----------------------

line 123 : $content .= _('Uploaded file') . ': ' . $filename . '<p />';

----------------------------------------------------------------------
```

## 搭建环境测试
exploit-db给的信息已是相当充足，提供了Exploit的下载，以及包含该漏洞的App。由于这一漏洞十分的简单，所以我们搭建环境复现一下，再进行进一步的分析与说明。

PlaySms 1.4的安装包解压后，参考里面的INSTALL.md的说明进行安装。安装方法有两种，第一种是集成的shell脚本一键安装，第二种则是拆开每个step进行安装。所需要的系统环境是Linux，我这里使用了Ubuntu Kylin 16.04以及phpStudy集成的lnmp（php5.5 + nginx + mysql）进行适配。安装中发现第一种方法安装后，有一些小问题需要微调，所以最终也就相当于使用了第二种方法。playSms不像一些很出名的诸如wordpress，dedecms等整站程序安装那样友好，如果你在调试时遇到问题，就多google吧。

安装之后，可以用默认的用户名密码登陆：(admin:admin)

![](/images/exploit/playSms/20170515_1.jpg)

在"MyAccount"菜单下找到"Send from file"，这里允许导入csv格式的文件。

![](/images/exploit/playSms/20170515_2.jpg)

我们可以看一下页面源码，找到这个表单：

![](/images/exploit/playSms/20170515_3.jpg)

可以看到表单通过post方法提交给`index.php?app=main&inc=feature_sendfromfile&op=upload_confirm`，提交的同时通过GET方法指定了3个参数，app,inc和op。表单的具体内容主要有3个，一个隐藏起来的防CSRF攻击的token字段，这一value是服务器在响应页面请求时带过来的，显然这里采用了token的方法对CSRF攻击进行了防护。另外两个一个是上传的文件，另一个是是否阻止duplicate的checkbox。

我们这里仅仅关心`<input type="file" name="fncsv">`的字段，当我们点击上传文件后，服务器端最终会走到playsms/plugin/feature/sendfromfile/sendfromfile.php的52行：

![](/images/exploit/playSms/20170515_4.jpg)

这里通过超全局变量`$_FILES`按表单的name字段取出'fncsv'文件数组，`$filename`被简单的指定为`$_FILES['fncsv']['name']`。其后又取出了`$_FILES['fncsv']`的其他成员，主要是对duplicate以及大小的一些检查处理和校验。

跳过这里的检查处理后，是对待响应的页面内容进行填充，其中$content变量用于保存响应的页面。这里在123行可以看到，对`$content`进行追加赋值时，没有对$filename进行任何过滤：

![](/images/exploit/playSms/20170515_5.jpg)

这也就导致了命令执行漏洞的出现，显然这里可以通过精心构造上传文件的名字来指定`$filename`，123行会将$filename按照php语法进行解析，如果我们上传的文件名为`<?php system('uname -r');?>`，123行就会变成`$content .= _('Uploaded file') . ':' . <?php system(uname -r);?> . '<p />';`。

![](/images/exploit/playSms/20170515_6.jpg)

当然，这里在123行之后还有一些对处理结果的逻辑响应，对于本例来说，不会影响页面的正常返回，如果不想继续解析下去而避免可能出现的各种问题，可以直接执行`die();`。名字改为：`<?php system('uname -a');die();?>`。

![](/images/exploit/playSms/20170515_7.jpg)

## 一点引申的思考
这个漏洞实际上是一目了然的，但是对于程序员来说又是极容易忽略的。我们很多时候习惯了Windows系统，认为文件名不能够包含‘<’等特殊字符，但实际上，且不说Linux允许`<?php system('uname -a');die();?>.php`这样的名字，就算都不允许，也可以通过抓包改包的方式（比如fiddler，burpsuite等一干好用的http proxy工具）改掉。从设计的角度来看，对于服务器的程序，要尽量少依赖其他模块的安全，本身做好检查与过滤。

另一方面，由于这个文件不会保存到服务器上，而只是简单的在内存中解析出内容进行后续处理，所以开发者很容易认为这并不危险。但实际上，如果权限足够，比如依然用`system()`命令，此时我不加载人畜无害的`uname -a`而是一个保存一句话木马内容到文件的shell语句，整个服务器也就快沦陷了。

为了防止查水表，后续不可描述的内容就不进行展示了，聪明的人懂得自扩展。