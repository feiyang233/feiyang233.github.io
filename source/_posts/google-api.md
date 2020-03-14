---
title: Google Sheet API 学习
date: 2019-05-24 14:47:34
tags: google
categories: 学习
---
# Python 修改 Google sheet
[官方文档](https://developers.google.com/sheets/api/quickstart/python)  
记录一下自己调用 Google Api 的方法。
<!--more-->

# 几个重要的概念
1. spreadsheetId 整个总表的 ID 是很长的一串字符 
2. sheetId 单页的 ID 是纯数字


# Get 
获取数据[get 方法](https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/get)

```python
SAMPLE_SPREADSHEET_ID = spreadsheetId
SAMPLE_RANGE_NAME = 'feiyang!G1:G4'

sheet = service.spreadsheets()
result = sheet.values().get(spreadsheetId=SAMPLE_SPREADSHEET_ID,
                            range=SAMPLE_RANGE_NAME).execute()
values = result.get('values', [])
```


# Append Data

```python
range_ = 'capacity-raw!A:E'  # 表内的页名称 ! 范围
value_input_option = 'USER_ENTERED'  
insert_data_option = 'INSERT_ROWS'  

value_range_body = {
    "range": "capacity-raw!A:E",
"values": getdata.get_data(today,product),
"majorDimension": "ROWS"
}

request = service.spreadsheets().values().append(spreadsheetId=spreadsheet_id, range=range_, valueInputOption=value_input_option, insertDataOption=insert_data_option, body=value_range_body)
response = request.execute()

```

# Update Data
举个例子

```python
SAMPLE_SPREADSHEET_ID = 'xxxxxxxxx'
SAMPLE_RANGE_NAME = 'daily_report!A9:D9'
value_input_option = "RAW"  # 还有其他的方式
value_body = {
    "majorDimension": "ROWS",
    "range": "daily_report!A9:D9",
    "values": [["test","123","a","b"]],
}



sheet = service.spreadsheets()
result = sheet.values().update(spreadsheetId=SAMPLE_SPREADSHEET_ID,
                            range=SAMPLE_RANGE_NAME, valueInputOption=value_input_option,body=value_body)

response = result.execute()
pprint(response)
```

# Sheet Operation
删除行，插入行，复制一行，最重要的是 post body 格式。 官方文档写得不够详细。

```json
delete_body ={
   "requests": [
    {
      "deleteDimension": {
        "range": {
          "sheetId": 233333333,
          "dimension": "ROWS",
          "startIndex": 1,
          "endIndex": 2
        }
      }
    },
  ],
}

insert_body ={
   "requests": [
    {
      "insertDimension": {
        "range": {
          "sheetId": 233333333,
          "dimension": "ROWS",
          "startIndex": 8,
          "endIndex": 9
        }
      }
    },
  ],
}


copy_body ={
  "requests": [
    {
      "copyPaste": {
        "source": {
          "sheetId": 233333333,
          "startRowIndex": 6,
          "endRowIndex": 7,
          "startColumnIndex": 1,
          "endColumnIndex": 5
        },
        "destination": {
          "sheetId": 1172952310,
          "startRowIndex": 7,
          "endRowIndex": 8,
          "startColumnIndex": 1,
          "endColumnIndex": 5
        },
        "pasteType": "PASTE_NORMAL",
        "pasteOrientation": "NORMAL"
      }
    }
  ]
}

```
然后是 Python post 部分
```python
request = sheet.batchUpdate(spreadsheetId=SAMPLE_SPREADSHEET_ID, body=body_item)
response = request.execute()
pprint(response)

```

