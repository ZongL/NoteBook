
迁移vscode 缓存及用户配置
```
"D:\Program Files\Microsoft VS Code\Code.exe" 
--user-data-dir "D:\vscode_cache\User" 
--extensions-dir "D:\vscode_cache\Extensions"
```




set claudecode env
```
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "yourapikey",
    "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": 1,
	"ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-4.7"
	
  }
}
```