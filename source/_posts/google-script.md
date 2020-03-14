---
title: Google Script 学习
date: 2019-08-15 15:04:49
tags: google
categories: 学习
---
本文记录用[Google Script](https://developers.google.com/apps-script/guides/sheets) 处理数据，发送每日邮件的过程。
<!--more-->
# 创建 Sheet 和 Script
首先用自己的谷歌账号创建一个新的 Google Sheet, 然后点击菜单栏的 Tools > Script editor 就可以创建脚本。其语法和 JavaScript 相似的。
```js
/**
 * Sends emails with data from the current spreadsheet.
 */

// 首先添加一个发送按钮名称为 Action，绑定到函数 SendEmail
function onOpen() {
  SpreadsheetApp.getUi()
      .createMenu('Action')
      .addItem('Send Daily Report', 'SendEmail')
      .addToUi();
}
```
# 学会 debug
可以调用函数 Logger.log(); 打印结果到后台，然后在 View > Logs 查看结果。举一个简单的例子：  
markdown 语法注意，使用时发现，表格的语句上一行必须为空行，不然表格不生效。比如你的表是这个样子的  

|       | A      |  B     |
| :----:| :---: | :---: |
| 1      | A1     | B1     |
| 2      | A2     | B2     |
| 3      | A3     | B3     |
| 4      | A4     | B4     |

```js
function logProductInfo() {
  var sheet = SpreadsheetApp.getActiveSheet();
  var data = sheet.getDataRange().getValues();
  for (var i = 0; i < data.length; i++) {
    Logger.log('Product name: ' + data[i][0]);
    Logger.log('Product number: ' + data[i][1]);
  }
}
```
我们在 script editor 界面点击 Run > Run function > logProductInfo, 运行结束后，可以点击 View > Logs 查看结果。
```
[19-08-15 01:16:37:475 PDT] Product name: A1
[19-08-15 01:16:37:475 PDT] Product number: B1
[19-08-15 01:16:37:476 PDT] Product name: A2
[19-08-15 01:16:37:476 PDT] Product number: B2
[19-08-15 01:16:37:477 PDT] Product name: A3
[19-08-15 01:16:37:477 PDT] Product number: B3
[19-08-15 01:16:37:477 PDT] Product name: A4
[19-08-15 01:16:37:478 PDT] Product number: B4
```
# Code.gs
直接放上代码
```js

/**
 * Sends emails with data from the current spreadsheet.
 */

function onOpen() {
  SpreadsheetApp.getUi()
      .createMenu('Action')
      .addItem('Send Daily Report', 'SendEmail')
      .addToUi();
}

function test() {
  var scriptProperties = PropertiesService.getScriptProperties();
  var nickname = scriptProperties.getProperty('Project');
  Logger.log(nickname)
}


function SendEmail() {
  var client = PropertiesService.getScriptProperties().getProperty('Project');
  // Property 可以在 File > Project Properties > Script properties 里面设置
  var monitor_vn = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("VN").getDataRange().getValues();
  // 获取表单名称为 VN 的表内容
  var monitor_th = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("TH").getDataRange().getValues();
  // 获取表单名称为 TH 的表内容
  var sendlist = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("email").getDataRange().getValues();
  // 获取表单名称为 email 的表内容
  var sendto = []
  var sendcc = []
  
  // 将发送和 cc 分开存储
  for (var i=1;i<sendlist.length;i++){
    if (sendlist[i][0]!='') sendto.push(sendlist[i][0])
    if (sendlist[i][1]!='') sendcc.push(sendlist[i][1])
  }
  
  sendto = sendto.join(',')
  sendcc = sendcc.join(',')
  
 // Email Template 连接绑定文件 index.html
  var template = HtmlService.createTemplateFromFile('index');
  // 传参到 template
  template.monitor_vn = monitor_vn;
  template.monitor_th = monitor_th;
  template.client = client;
  
  // 自定义一个数组，将数字月份映射为英文
  const monthNames = ["January", "February", "March", "April", "May", "June",
  "July", "August", "September", "October", "November", "December"
  ];
  // 获取当前的日期
  var d = new Date();
  
  // 发送邮件
  MailApp.sendEmail({
    to: sendto,
    cc: sendcc,
    subject: client+' Monitoring Daily Report ('+d.getDate()+'-'+monthNames[d.getMonth()]+'-'+d.getYear()+')',
    htmlBody: template.evaluate().getContent()});
}
```

# index.html
这个是邮件的主体模板，可以看一下[官网介绍](https://developers.google.com/apps-script/guides/html/templates)
条件语句嵌套<? ... ?>
```html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
  </head>
  <body>
    <? if (true) { ?>
      <p>This will always be served!</p>
    <? } else  { ?>
      <p>This will never be served.</p>
    <? } ?>
  </body>
</html>

```
赋值语句<?= ... ?> 是

```html 
<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
  </head>
  <body>
    <?= 'My favorite Google products:' ?>
    <? var data = ['Gmail', 'Docs', 'Android'];
      for (var i = 0; i < data.length; i++) { ?>
        <b><?= data[i] ?></b>
    <? } ?>
  </body>
</html>
```

```html
<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
  </head>
  <body>
     <p>Hi,</p>
     <p>Below is daily ops report for <?=client?>:</p>

     
     <b style="font-size: 15px;">Monitoring_VN:</b><br>
     <table style="border-collapse: collapse; margin-left:20px;border: 1px solid black">
     <tr>
     <? for (var j=1;j<7;j++) { ?>  <!-- 因为只读取1-6列的数据 -->
     <th style="border: 1px solid black;background-color: grey; color: white; width:15%"><?= monitor_vn[0][j] ?></th>  
     <? } ?>
     </tr>
                    <!--monitor_vn.length 是行数-->
     <? for(var i=1;i<monitor_vn.length;i++) { ?>
     <tr>
       <? for(var j=1;j<7;j++) { ?>
         <? if ( j>2 && j< 6  ) { ?>
          <? if (monitor_vn[i][j] > 0.7) {?> <!-- 值大于0.7 底色红色-->
            <td style="border: 1px solid black; background-color: red;"><?= (monitor_vn[i][j]*100).toFixed(2)?>%</td> 
           <? } else  { ?>   <!--toFixed(2) 保留两位小数 -->
            <td style="border: 1px solid black;"><?= (monitor_vn[i][j]*100).toFixed(2)?>%</td> 
            <? } ?>
       <? } else  { ?>
        <? if ( j==6 && monitor_vn[i][j] > 100) {?>
            <td style="border: 1px solid black;background-color: red;"><?= (monitor_vn[i][j]) ?></td>
           <? } else  { ?>
            <td style="border: 1px solid black;"><?= (monitor_vn[i][j]) ?></td>
                   <? } ?>
     
         <? } ?>
       <? } ?>
     </tr>
     <? } ?>
     </table><br>
     
     <b style="font-size: 15px;">Monitoring_TH:</b><br>
     <table style="border-collapse: collapse; margin-left:20px;border: 1px solid black">
     <tr>
     <? for (var j=1;j<7;j++){ ?>
     <th style="border: 1px solid black;background-color: grey; color: white; width:15%"><?= monitor_th[0][j] ?></th>  
     <? } ?>
     </tr>
     
     <? for(var i=1;i<monitor_th.length;i++){?>
     <tr>
       <? for(var j=1;j<7;j++){?>
         <? if ( j>2 && j< 6  ) { ?>
           <? if (monitor_th[i][j] > 0.7) {?>
            <td style="border: 1px solid black; background-color: red;"><?= (monitor_th[i][j]*100).toFixed(2)?>%</td> 
           <? } else  { ?>
            <td style="border: 1px solid black;"><?= (monitor_th[i][j]*100).toFixed(2)?>%</td> 
                   <? } ?>
         <? } else  { ?>
           <? if ( j==6 && monitor_th[i][j] > 100) {?>
            <td style="border: 1px solid black;background-color: red;"><?= (monitor_th[i][j]) ?></td>
           <? } else  { ?>
            <td style="border: 1px solid black;"><?= (monitor_th[i][j]) ?></td>
                   <? } ?>
         <? } ?>
       <? } ?>
     </tr>
     <? } ?>
     </table><br>
  </body>
</html>
```
# 未完