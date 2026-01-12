使用[golang-jwt/jwt/v5](https://github.com/golang-jwt/jwt)  
引用[Golang 一日一库之jwt-go - 始識](https://www.cnblogs.com/zichliang/p/17303759.html)
## settings.yaml
```yaml
jwt:  
  expires: 8  
  issuer: WxL  
  secret: uis&*jbb55dHRhf5
```

## 定义Jwt配置项
```go
// config_jwt.go
type Jwt struct {  
    Expires int    `yaml:"expires"` // 过期时间 单位小时  
    Issuer  string `yaml:"issuer"`  // 颁发人  
    Secret  string `yaml:"secret"`  // 秘钥  
}
```

## 编写颁发Token逻辑
### 注册声明结构体

```go

// jwts/enter.go
type JwtPayLoad struct {  
    NickName string `json:"nickname"`  
    RoleID   uint   `json:"role_id"`  
    UserID   uint   `json:"user_id"`  
}  

// 自定义 Claims
type CustomClaims struct {  
    JwtPayLoad  
    jwt.RegisteredClaims  
}
```

### 生成Token

```go
// jwts/generate_token.go

// GenToken 生成token  
func GenToken(user JwtPayLoad) (string, error) {  
    expires := time.Duration(global.Config.Jwt.Expires) * time.Hour  
    claims := CustomClaims{  
       JwtPayLoad: user,  
       RegisteredClaims: jwt.RegisteredClaims{  
          ExpiresAt: jwt.NewNumericDate(time.Now().Add(expires)), // 过期时间  
          Issuer:    global.Config.Jwt.Issuer,  
       },  
    }  
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)  
    // 小坑 这里的 SingedString 必须是byte字节的  
    return token.SignedString([]byte(global.Config.Jwt.Secret))  
}
```

### 解析Jwt

```go
// jwts/parse_token
// ParseToken token解析  
func ParseToken(token string) (*CustomClaims, error) {  
    Token, err := jwt.ParseWithClaims(  
       token, &CustomClaims{},  
       func(token *jwt.Token) (interface{}, error) {  
          return []byte(global.Config.Jwt.Secret), nil  
       },  
    )  
    if err != nil {  
       return nil, err  
    }  
    claims, ok := Token.Claims.(*CustomClaims)  
    if !ok {  
       // 数据不一致  
       return nil, err  
    }  
    if !Token.Valid {  
       // 令牌无效  
       return nil, err  
    }  
    return claims, nil  
}
```

## 编写Jwt鉴权中间件（解析token）
```go
// jwt_admin.go
func JwtAdmin(c *gin.Context) {  
    token := c.Request.Header.Get("token")  
    if token == "" {  
       res.FailWithMsg("未携带token", c)  
       c.Abort()  
       return  
    }  
    claims, err := jwts.ParseToken(token)  
    if err != nil {  
       res.FailWithMsg("token错误", c)  
       c.Abort()  
       return  
    }  
      
    ok := redis_service.CheckLogout(token)  
    if ok {  
       res.FailWithMsg("token已注销", c)  
       c.Abort()  
       return  
    }  
  
    if claims.RoleID != 1 {  
       res.FailWithMsg("权限不足", c)  
       c.Abort()  
       return  
    }  
    c.Set("claims", claims)  
}
```

## 用户登录时生成Token
```go
// user_login.go
token, err := jwts.GenToken(jwts.JwtPayLoad{  
    NickName: user.NickName,  
    RoleID:   user.RoleID,  
    UserID:   user.ID,  
    UserName: user.UserName,  
})
```