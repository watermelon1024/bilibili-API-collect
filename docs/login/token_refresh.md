# APP Token 刷新（含新 SESSDATA）

> https://passport.bilibili.com/x/passport-login/oauth2/refresh_token  
> https://passport.bilibili.com/api/v2/oauth2/refresh_token  

*请求方式：POST*  
*Content-Type：`application/x-www-form-urlencoded`*

用登录 / 换票得到的 **`access_token` + `refresh_token`** 换取**新**的 `token_info` 与 **`cookie_info`（含新 SESSDATA）**。

---

## 端点

| URL | 备注 |
|-----|------|
| `POST https://passport.bilibili.com/x/passport-login/oauth2/refresh_token` | IPA 字符串 / 社区常见路径；**推荐** |
| `POST https://passport.bilibili.com/api/v2/oauth2/refresh_token` | 与国际版 `api/v2` oauth 同族；实测同样 **code=0** 且带回 `cookie_info` |

---

## 正文参数

| 参数名 | 类型 | 内容 | 必要性 | 备注 |
|--------|------|------|--------|------|
| access_token | str | 当前 access | 必要* | 与下一项二选一或可同时带 |
| access_key | str | 同上别名 | 必要* | 实测单独/同时带均可 |
| refresh_token | str | 当前 refresh | **必要** | 登录/oauth/上次 refresh 下发 |
| appkey | str | 见 [appkey](../misc/sign/APPKey.md) | 必要 | `0ac1706090f12cfc` |
| ts | num | UNIX 秒 | 必要 | |
| sign | str | APP 签名 | 必要 | 见上游 [APP 签名](../misc/sign/APP.md) |
| mobi_app | str | 例 `iphone_i` | 建议 | 与登录一致 |
| platform | str | `ios` | 建议 | |
| build | str/num | 例 `77500100` | 建议 | |

\* 至少提供 `access_token` 或 `access_key` 之一（值为同一串 token）。

### appkey 注意

| appkey | 对国际版游客 token |
|--------|---------------------|
| `0ac1706090f12cfc`（本链路 passport） | **可用** |
| 主站 android 常用 `1d8b6e7d…` 等 | 常见 **`86033` appID不匹配** |

**应用「发 token 时」的那组 appkey/appsec**，不要混站。

---

## 请求头（建议）

| 头 | 内容 |
|----|------|
| User-Agent | 国际版 UA（或任意合理 App UA） |
| Content-Type | `application/x-www-form-urlencoded` |
| buvid / x-bili-ticket | 可选；刷新**不依赖**旧 SESSDATA |

---

## JSON 回复（成功）

根对象：`code=0`，`data` 与重登同形：

| 字段 | 类型 | 内容 |
|------|------|------|
| status | num | 0 |
| token_info | obj | **新** mid / access_token / refresh_token / expires_in / region… |
| cookie_info | obj | **新** Cookie 列表（见下） |
| sso | array | SSO URL |
| is_new | bool | 常 false |
| is_tourist | bool | 游客为 true |

`token_info`：

| 字段 | 说明 |
|------|------|
| access_token | **新** App `access_key`；**旧 access 应视为作废** |
| refresh_token | **新** 刷新口令；**务必覆盖持久化** |
| expires_in | 实测仍约 `15552000`（≈180 天） |
| mid | 不变 |

`cookie_info.cookies[]` 常见：

| name | 说明 |
|------|------|
| **SESSDATA** | **新** Web 会话（本接口核心附加产物） |
| bili_jct | 新 CSRF |
| DedeUserID | mid |
| DedeUserID__ckMd5 | |
| sid | |

### 成功示例（脱敏）

<details>
<summary>展开</summary>
```json
{
  "code": 0,
  "message": "OK",
  "ttl": 1,
  "data": {
    "status": 0,
    "token_info": {
      "mid": 3707041629603926,
      "access_token": "9aa569c5…IIEC",
      "refresh_token": "b5fcda858da39d937468f69f…",
      "expires_in": 15552000,
      "region": "TW",
      "store_region": "TW"
    },
    "cookie_info": {
      "cookies": [
        { "name": "SESSDATA", "value": "61b7a817%2C…", "http_only": 1 },
        { "name": "bili_jct", "value": "…", "http_only": 0 },
        { "name": "DedeUserID", "value": "3707041629603926", "http_only": 0 },
        { "name": "DedeUserID__ckMd5", "value": "…", "http_only": 0 },
        { "name": "sid", "value": "…", "http_only": 0 }
      ],
      "domains": [".bilibili.com", ".biligame.com", ".bilibili.cn"]
    },
    "is_new": false,
    "is_tourist": true
  }
}
```
</details>

---

## 伪代码

```python
params = {
    "access_token": old_access,      # 或 access_key=
    "refresh_token": old_refresh,
    "appkey": "0ac1706090f12cfc",
    "ts": str(int(time.time())),
    "mobi_app": "iphone_i",
    "platform": "ios",
    "build": "77500100",
}
params["sign"] = appsign(params, appsec)  # cdb18f9752ab7a1d09d5941c62893b0c

POST passport.bilibili.com/x/passport-login/oauth2/refresh_token

# 成功后立刻覆盖存储：
#   token_info.access_token / refresh_token
#   cookie_info 全罐（至少 SESSDATA + bili_jct）
# 旧 access / 旧 refresh / 旧 SESSDATA 不要再混用
```

---

## 验证

| 栈 | 方法 |
|----|------|
| Web | `GET https://api.bilibili.com/x/web-interface/nav` + **新** `SESSDATA` → `isLogin=true`、`mid` 一致 |
| App | `GET https://app.bilibili.com/x/v2/account/myinfo` + **新** `access_key` → `is_tourist=1` |

---

## 错误码（实验）

| code | 含义 | 备注 |
|------|------|------|
| 0 | 成功 | 含新 token + cookie |
| -3 | 签名错误 | appsec / 参与签名字段 |
| **-400** | 请求错误 | 缺参、错 path 变体等 |
| **86033** | appID不匹配 | appkey 与发 token 时不一致 |
| -101 | 账号未登录 | 多见于 **Web** cookie 刷新误用；本接口材料对时不应出现 |
| 其它 passport 类 | token 失效 / 已撤销 | 请重新登入 |
