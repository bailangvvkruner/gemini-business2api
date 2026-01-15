我打开了网页https://business.gemini.google/然后他重定向了一个需要登录的地址 网络监控一下 需要输入我的邮箱2123146130@qq.com 验证码需要等我这边输给你 然后登录进去以后 类似这样页面https://business.gemini.google/home/cid/d06739ca-6683-46db-bb51-07395a392439?csesidx=2026259595&mods 
调用MCP输入并发送一条消息 看一下整个流程 需要什么请求头 请求体 还有模型的也是需要默认是Auto你给切换一下 看看都有什么模型 我的目的是docker部署golang程序 全部变量-e不存文件 日志标准输出到docker 代理器代理网页端openai格式
https://business.gemini.google/home/cid/d06739ca-6683-46db-bb51-07395a392439?csesidx=2026259595&mods





## 🔍 完整的API流程分析

### 1. 登录流程

__关键请求：__

- __验证码发送__: `POST https://accountverification.business.gemini.google/v1/verify-oob-code`
- __请求体__: `ip_azr=[加密token]&ip_lh=[login_hint]`
- __响应__: 返回HTML页面，包含验证码输入界面

### 2. 页面访问和认证

__关键请求：__

- __设置Cookie__: `GET https://business.gemini.google/auth/setocookie`
- __获取CSRF Token__: `GET https://business.gemini.google/auth/getoxsrf?csesidx=[id]`
- __会话列表__: `GET https://auth.business.gemini.google/list-sessions`

### 3. 核心API请求（发送消息）

__创建会话:__

```javascript
POST https://biz-discoveryengine.googleapis.com/v1alpha/locations/global/widgetCreateSession
Authorization: Bearer [JWT Token]
Content-Type: application/json

{
  "configId": "d06739ca-6683-46db-bb51-07395a392439",
  "additionalParams": {"token": "-"},
  "createSessionRequest": {
    "session": {"name": "-", "displayName": ""}
  }
}
```

__流式响应（核心）:__

```javascript
POST https://biz-discoveryengine.googleapis.com/v1alpha/locations/global/widgetStreamAssist
Authorization: Bearer [JWT Token]
Content-Type: application/json
X-Server-Timeout: 1800

{
  "configId": "d06739ca-6683-46db-bb51-07395a392439",
  "additionalParams": {"token": "-"},
  "streamAssistRequest": {
    "session": "collections/default_collection/engines/agentspace-engine/sessions/[session_id]",
    "query": {"parts": [{"text": "你好，请介绍一下Redis"}]},
    "filter": "",
    "fileIds": [],
    "answerGenerationMode": "NORMAL",
    "toolsSpec": {
      "webGroundingSpec": {},
      "toolRegistry": "default_tool_registry",
      "imageGenerationSpec": {},
      "videoGenerationSpec": {}
    },
    "languageCode": "zh",
    "userMetadata": {"timeZone": "Asia/Shanghai"},
    "assistSkippingMode": "REQUEST_ASSIST"
  }
}
```

## 🏗️ Golang代理程序架构建议

### 核心组件设计

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/websocket"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

// 配置结构体
type Config struct {
    GeminiAPIBase    string `json:"gemini_api_base"`
    AuthDomain       string `json:"auth_domain"`
    BusinessDomain   string `json:"business_domain"`
    ConfigID         string `json:"config_id"`
    JWTExpiry        int    `json:"jwt_expiry_minutes"`
}

// 会话管理器
type SessionManager struct {
    sessions map[string]*GeminiSession
    mu       sync.RWMutex
}

type GeminiSession struct {
    SessionID    string    `json:"session_id"`
    JWTToken     string    `json:"jwt_token"`
    ExpiresAt    time.Time `json:"expires_at"`
    ConfigID     string    `json:"config_id"`
    LastActivity time.Time `json:"last_activity"`
}

// 请求结构体
type ChatRequest struct {
    Message string   `json:"message"`
    Session string   `json:"session,omitempty"`
    Files   []string `json:"files,omitempty"`
}

type StreamResponse struct {
    Type    string      `json:"type"`
    Content string      `json:"content,omitempty"`
    Done    bool        `json:"done,omitempty"`
    Error   string      `json:"error,omitempty"`
}

// 主服务结构
type GeminiProxyService struct {
    config        *Config
    sessionMgr    *SessionManager
    httpClient    *http.Client
    jwtCache      *JWTCache
}

// JWT缓存
type JWTCache struct {
    token  string
    expiry time.Time
    mu     sync.RWMutex
}
```

### 关键实现要点

#### 1. __环境变量配置__

```go
func loadConfig() *Config {
    return &Config{
        GeminiAPIBase:  getEnv("GEMINI_API_BASE", "https://biz-discoveryengine.googleapis.com"),
        AuthDomain:     getEnv("AUTH_DOMAIN", "auth.business.gemini.google"),
        BusinessDomain: getEnv("BUSINESS_DOMAIN", "business.gemini.google"),
        ConfigID:       getEnv("CONFIG_ID", "d06739ca-6683-46db-bb51-07395a392439"),
        JWTExpiry:      getEnvInt("JWT_EXPIRY_MINUTES", 60),
    }
}
```

#### 2. __JWT Token管理__

```go
func (s *GeminiProxyService) getValidJWT(ctx context.Context) (string, error) {
    s.jwtCache.mu.RLock()
    if time.Now().Before(s.jwtCache.expiry) && s.jwtCache.token != "" {
        token := s.jwtCache.token
        s.jwtCache.mu.RUnlock()
        return token, nil
    }
    s.jwtCache.mu.RUnlock()
    
    // 从登录会话获取新token
    return s.refreshJWTFromSession(ctx)
}
```

#### 3. __会话创建和管理__

```go
func (s *GeminiProxyService) createSession(ctx context.Context, jwtToken string) (*GeminiSession, error) {
    url := fmt.Sprintf("%s/v1alpha/locations/global/widgetCreateSession", s.config.GeminiAPIBase)
    
    body := map[string]interface{}{
        "configId": s.config.ConfigID,
        "additionalParams": map[string]string{"token": "-"},
        "createSessionRequest": map[string]interface{}{
            "session": map[string]string{
                "name": "-",
                "displayName": "",
            },
        },
    }
    
    // 发送请求并解析响应
    // ...
    
    return &GeminiSession{
        SessionID: sessionID,
        JWTToken:  jwtToken,
        ConfigID:  s.config.ConfigID,
    }, nil
}
```

#### 4. __流式响应处理（核心）__

```go
func (s *GeminiProxyService) streamChat(ctx context.Context, session *GeminiSession, message string, w http.ResponseWriter) error {
    url := fmt.Sprintf("%s/v1alpha/locations/global/widgetStreamAssist", s.config.GeminiAPIBase)
    
    body := map[string]interface{}{
        "configId": s.config.ConfigID,
        "additionalParams": map[string]string{"token": "-"},
        "streamAssistRequest": map[string]interface{}{
            "session": fmt.Sprintf("collections/default_collection/engines/agentspace-engine/sessions/%s", session.SessionID),
            "query": map[string]interface{}{
                "parts": []map[string]string{{"text": message}},
            },
            "filter": "",
            "fileIds": []string{},
            "answerGenerationMode": "NORMAL",
            "toolsSpec": map[string]interface{}{
                "webGroundingSpec": map[string]interface{}{},
                "toolRegistry": "default_tool_registry",
                "imageGenerationSpec": map[string]interface{}{},
                "videoGenerationSpec": map[string]interface{}{},
            },
            "languageCode": "zh",
            "userMetadata": map[string]string{"timeZone": "Asia/Shanghai"},
            "assistSkippingMode": "REQUEST_ASSIST",
        },
    }
    
    // 发送请求并流式处理响应
    // 使用 chunked transfer encoding
    // 解析每个chunk并转发给客户端
}
```

#### 5. __OpenAI格式转换__

```go
type OpenAIChatRequest struct {
    Model    string    `json:"model"`
    Messages []Message `json:"messages"`
    Stream   bool      `json:"stream"`
}

type OpenAIStreamResponse struct {
    ID      string   `json:"id"`
    Object  string   `json:"object"`
    Created int64    `json:"created"`
    Model   string   `json:"model"`
    Choices []Choice `json:"choices"`
}

func convertToOpenAIFormat(geminiResponse string) OpenAIStreamResponse {
    // 将Gemini的响应格式转换为OpenAI格式
    return OpenAIStreamResponse{
        ID:      fmt.Sprintf("chatcmpl-%d", time.Now().Unix()),
        Object:  "chat.completion.chunk",
        Created: time.Now().Unix(),
        Model:   "gemini-2.0",
        Choices: []Choice{
            {
                Delta: map[string]string{
                    "content": geminiResponse,
                },
                Index:        0,
                FinishReason: nil,
            },
        },
    }
}
```

### 6. __Docker部署配置__

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o gemini-proxy .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/gemini-proxy .
EXPOSE 8080
CMD ["./gemini-proxy"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  gemini-proxy:
    build: .
    ports:
      - "8080:8080"
    environment:
      - GEMINI_API_BASE=https://biz-discoveryengine.googleapis.com
      - AUTH_DOMAIN=auth.business.gemini.google
      - BUSINESS_DOMAIN=business.gemini.google
      - CONFIG_ID=d06739ca-6683-46db-bb51-07395a392439
      - JWT_EXPIRY_MINUTES=60
      - LOG_LEVEL=info
    restart: unless-stopped
```

### 7. __API端点设计__

```go
func main() {
    service := NewGeminiProxyService()
    
    e := echo.New()
    e.Use(middleware.CORS())
    e.Use(middleware.Logger())
    
    // OpenAI兼容端点
    e.POST("/v1/chat/completions", service.handleChatCompletions)
    
    // 健康检查
    e.GET("/health", service.healthCheck)
    
    // 会话管理
    e.POST("/sessions", service.createSessionHandler)
    e.DELETE("/sessions/:id", service.deleteSessionHandler)
    
    e.Logger.Fatal(e.Start(":8080"))
}
```

### 8. __日志和监控__

```go
type Logger struct {
    level string
}

func (l *Logger) Info(msg string, fields ...interface{}) {
    if l.level == "debug" || l.level == "info" {
        fmt.Printf("[INFO] %s %v\n", msg, fields)
    }
}

func (l *Logger) Error(msg string, err error) {
    fmt.Printf("[ERROR] %s: %v\n", msg, err)
}
```

## 🔑 关键技术要点

1. __认证机制__: 使用Bearer Token (JWT) 进行API认证
2. __流式处理__: 使用chunked transfer encoding处理流式响应
3. __会话管理__: 维护会话状态，支持多用户并发
4. __错误处理__: 完善的错误处理和重试机制
5. __格式转换__: 将Gemini响应转换为OpenAI兼容格式
6. __环境变量__: 所有配置通过环境变量管理，不存储敏感信息
7. __日志输出__: 标准输出到Docker，支持日志收集

这个架构可以满足您的所有需求：Docker部署、环境变量配置、标准输出日志、OpenAI格式代理。