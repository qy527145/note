# 安装openclaw

```bash
# 使用npm
npm i -g openclaw
# 使用pnpm
pnpm i -g openclaw
# 使用bun
bun i -g openclaw

# 如果下载慢需要设置镜像源或者使用代理（自行解决）
```
# 配置openclaw
写入配置文件: `~/.openclaw/openclaw.json`, 如果是以前安装过的，建议先备份配置文件，使用`openclaw reset`重置一下，然后再配置
```jsonc
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "10b9909aaafd9f58fbabc1356e287c4d44c0e8d13902304a"
    }
  },
  "models": {
    "providers": {
      "custom-1": {
        "baseUrl": "http://192.168.145.16/v1",
        "apiKey": "sk-xxxxxxxxxxxxxxx",
        "auth": "api-key",
        "api": "openai-responses",
        "authHeader": true,
        "models": [
          {
            "id": "gpt_oss_120b",
            "name": "oss",
            "reasoning": false,
            "input": [
              "text"
            ],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 200000,
            "maxTokens": 200000
          }
        ]
      }
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.4.9",
    "lastTouchedAt": "2026-04-10T02:43:34.798Z"
  }
}
```
> 其中的gateway.token对应后续的web鉴权<br />
  sk-xxxxxxxxxxxxxxx替换成自己个人申请的key
# 运行openclaw
```bash
openclaw gateway --allow-unconfigured
```
# 使用openclaw
1. 使用web

    浏览器打开：`http://127.0.0.1:18789/#token=10b9909aaafd9f58fbabc1356e287c4d44c0e8d13902304a`
    > 这里的token对应配置文件中的gateway.token
2. 使用tui（命令行客户端）
    
    `openclaw tui`

> 以上是保证成功的步骤，如果清楚具体每一步是干什么，可以灵活执行
