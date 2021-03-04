## 动态生成活动图片

##### 输入模版json
  [json-model.md](json-model.md) 
##### 产生图片

[模版demo](../image/poster_demo.png)

---

#### Json解析

| 字段key      | 类型    | 是否非空                       | 默认值 | 说明     | 示例 |
| ------------ | ------- | ------------------------------ | ------ | -------- | ---- |
| images       | Arrays  | 否(默认) / 是(imageBlocks存在) | -      | 图片池   | -    |
| imageBlocks  | Arrays  | 否                             | -      | 图片组成 | -    |
| areaBlocks   | Arrays  | 否                             | -      | 底板组成 | -    |
| fontBlocks   | Arrays  | 否                             | -      | 字体组成 | -    |
| posterBlocks | Arrays  | 否                             | -      | 海报组成 | -    |
| width        | Integer | 是                             | -      | 宽       | 700  |
| height       | Integer | 是                             | -      | 高       | 900  |
| radius       | Object  | 否                             | -      | 圆角属性 | -    |

##### images

```json
[
    {
        "code":"jiuyang",
        "targetUrl":"/Users/xiaospace/Downloads/Joyoung-new.png",
        "targetType":"local"
    },
    {
        "code":"jiuyang",
        "targetUrl":"https://ai-cdn.joyoung.com/ia/poster/Joyoung-new.png",
        "targetType":"web"
    }
]
```



| 字段key    | 类型   | 是否非空 | 默认值 | 说明                             | 示例                                                 |
| ---------- | ------ | -------- | ------ | -------------------------------- | ---------------------------------------------------- |
| code       | String | 是       | -      | 图片code                         | jiuyang                                              |
| targetUrl  | String | 是       | -      | 图片url                          | https://ai-cdn.joyoung.com/ia/poster/Joyoung-new.png |
| targetType | Enum   | 是       | -      | 图片类型(local：本地、web：网址) | web/local                                            |

