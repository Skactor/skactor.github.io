---
layout: "post"
title: "Cobalt Strike 3.12 界面中文乱码解决方案"
date: "2018-12-21T11:20:11.357Z"
tags:
  - Cobalt Strike
categories:
  - 原创
  - 工具
---

众所周知的是，Cobalt Strike 对中文的支持并不好，这一点在 3.10 版本得到了缓解，在回显部分提供了中文支持，但是界面的中文支持还是存在问题，如下图所示：

![1](./Cobalt%20Strike%203.12%20界面中文乱码解决方案/20181221113323921.png)

这里我选择了 3.12 版本，也是目前为止最新版进行处理，下图是处理之后的结果：

![2](./Cobalt%20Strike%203.12%20界面中文乱码解决方案/2018122111344492.png)

这里大概有如下两个方案

## 方案一

一个比较简易的方案是这样

修改`%USERPROFILE%\.aggressor.prop`，在最后一行加入`client.font.font=Microsoft-YaHei-UI 10`

或者任意你想要的字体

这里有一个小 tip，这里的字体是通过 Java 的`java.awt.Font.decode`方法进行解码的。

我这里选用的`Microsoft-YaHei-UI 10`是我个人觉得比较合适的一个字体。

## 方案二

修改 UseSynthetica.class 文件

把下面的代码保存为`UseSynthetica.java`

```Java
package aggressor.ui;

import aggressor.Prefs;
import de.javasoft.plaf.synthetica.SyntheticaLookAndFeel;
import java.awt.Font;
import javax.swing.UIManager;

public class UseSynthetica extends UseLookAndFeel {
    public UseSynthetica() {
    }

    public void setup() {
        try {
            SyntheticaLookAndFeel.setWindowsDecorated(false);
            set("Synthetica.extendedFileChooser.rememberPreferences", false);
            set("Synthetica.font.enabled", true);
            set("Synthetica.text.antialias", false);
            set("Synthetica.textArea.border.opaqueBackground", false);
            set("Synthetica.font.respectSystemDPI", false);
            UIManager.setLookAndFeel("de.javasoft.plaf.synthetica.SyntheticaBlueIceLookAndFeel");
            SyntheticaLookAndFeel.setFont("Microsoft YaHei UI", 12);
        } catch (Exception var2) {
            var2.printStackTrace();
        }

    }

    static {
        String[] li = new String[]{"Licensee=Strategic Cyber LLC", "LicenseRegistrationNumber=404478475", "Product=Synthetica", "LicenseType=Small Business License", "ExpireDate=--.--.----", "MaxVersion=2.30.999"};
        UIManager.put("Synthetica.license.info", li);
        UIManager.put("Synthetica.license.key", "D6363B2A-F83CD00A-C4EB6105-31B2770B");
    }
}

```

然后执行 `javac -classpath cobaltstrike.jar UseSynthetica.java`

就可以看到已经在当前目录出现了 `UseSynthetica.class`

用这个文件替换掉`cobaltstrike.jar`中的`aggressor\ui\UseSynthetica.class`文件

然后再启动就可以看到已经支持中文了

---

[直接下载](https://github.com/Skactor/CS_Chinese_Support)
