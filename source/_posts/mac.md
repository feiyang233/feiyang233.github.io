---
title: Mac 小技巧
date: 2019-09-11 10:51:35
tags: mac
categories: Linux
---
苹果电脑小技巧。
<!--more-->
# 命令行技巧
+ 移动到行首 **ctrl+A**
+ 移动当行尾 **ctrl+E**
+ 往左移一个单词 **esc+B**
+ 往右移一个单词 **esc+F**
+ 删除光标前一个单词 **ctrl+W**
+ 删除光标前所有 **ctrl+U**

# 上传rz 下载sz
+ 首先下载 [iTerm2](https://www.iterm2.com/) 因为自带的 terminal 不行啊
+ 安装 lrzsz **brew install lrzsz**
+ 下载两个脚本 send-zmodem.sh  recv-zmodem.sh
```bash
cd /usr/local/bin 
sudo wget https://gist.githubusercontent.com/sy-records/1b3010b566af42f57fa6fa38138dd22a/raw/2bfe590665d3b0e6c8223623922474361058920c/iterm2-send-zmodem.sh 
sudo wget https://gist.githubusercontent.com/sy-records/40f4ba22e3fbdeedf58463b067798962/raw/b32d2f7ac3fa54acca81be3664797cebb724690f/iterm2-recv-zmodem.sh
sudo chmod 755 /usr/local/bin/iterm2-* 
```
如果链接失败，可以复制以下文件
```bash
#vim /usr/local/bin/iterm2-send-zmodem.sh

#!/bin/bash
# Author: Matt Mastracci (matthew@mastracci.com)
# AppleScript from http://stackoverflow.com/questions/4309087/cancel-button-on-osascript-in-a-bash-script
# licensed under cc-wiki with attribution required 
# Remainder of script public domain

osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
    FILE=`osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
else
    FILE=`osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
fi
if [[ $FILE = "" ]]; then
    echo Cancelled.
    # Send ZModem cancel
    echo -e \\x18\\x18\\x18\\x18\\x18
    sleep 1
    echo
    echo \# Cancelled transfer
else
    /usr/local/bin/sz "$FILE" -e -b
    sleep 1
    echo
    echo \# Received $FILE
fi

#vim /usr/local/bin/iterm2-recv-zmodem.sh

#!/bin/bash
# Author: Matt Mastracci (matthew@mastracci.com)
# AppleScript from http://stackoverflow.com/questions/4309087/cancel-button-on-osascript-in-a-bash-script
# licensed under cc-wiki with attribution required 
# Remainder of script public domain

osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
    FILE=`osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
else
    FILE=`osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
fi

if [[ $FILE = "" ]]; then
    echo Cancelled.
    # Send ZModem cancel
    echo -e \\x18\\x18\\x18\\x18\\x18
    sleep 1
    echo
    echo \# Cancelled transfer
else
    cd "$FILE"
    /usr/local/bin/rz -E -e -b
    sleep 1
    echo
    echo
    echo \# Sent \-\> $FILE
fi
```
+  iTerm2 的配置
点击 iTerm2 的设置界面 Perference -> Profiles -> Default -> Advanced -> Triggers 的 Edit 按钮
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190911151439.png)
点击+号，添加如下的参数
```bash
Regular expression: \*\*B0100
            Action: Run Silent Coprocess
        Parameters: /usr/local/bin/iterm2-send-zmodem.sh
           Instant: checked

Regular expression: \*\*B00000000000000
            Action: Run Silent Coprocess
        Parameters: /usr/local/bin/iterm2-recv-zmodem.sh
           Instant: checked
```
添加完成如下图所示
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190911151446.png)
简单的用法，当 ssh 登录后
```
# 下载一个文件： 
sz filename 
# 下载多个文件： 
sz filename1 filename2
# 下载dir目录下的所有文件，不包含dir下的文件夹： 
sz dir/*

# 直接键入rz命令即可
rz
# 在弹出的文件窗口，选择需要上传的文件
```