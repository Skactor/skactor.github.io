---
layout: "post"
title: "从IANA提取最新根域名列表的Python脚本"
date: "2018-03-26T17:35:06.684Z"
tags:
 - Python
categories:
 - 原创
 - 工具
---

仅作记录

```Python
# coding:utf-8
import re

import requests

if __name__ == '__main__':
    ret = requests.get('https://www.iana.org/domains/root/db').text
    i = ret.find('<tbody>')
    j = ret.find('</tbody>', i)
    ret = ret[i:j]
    ret = re.findall('<tr>\s*<td>(.+?)</td>\s*<td>(.+?)</td>\s*<td>(.+?)</td>\s*</tr>', ret, re.S)
    result = {
        'generic':[],
        'generic-restricted':[],
        'country-code':[],
        'sponsored':[],
        'infrastructure':[],
        'test':[]
    }
    for i in ret:
        j, k = re.findall('db/(\S+).html">(\S+)</a>', i[0].strip())[0]
        result[i[1]].append({'domain': j,'':k,'tld_manager':i[2]})
    print(result)

```
