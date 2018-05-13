# jwt(JSON Web Token)

- [写経](https://qiita.com/nirasan/items/1d5a2527a5384c863aa3)
- [メリット](https://qiita.com/kaiinui/items/21ec7cc8a1130a1a103a)

## 流れ

### 1. / ログインフォーム 行き先は/login

フォームでユーザーがログインする

### 2. /login 認証トークン発行

ユーザーの認証ロジックを定義している。例えばRDBMSとかに確認しにいくとか。

```
curl -F 'username=admin' -F 'password=admin' http://localhost:8080/login
Your token is eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MjYyNTYyMDksInN1YiI6ImFkbWluIn0.5ToBUNMG4s50-7yLcvm_dKfC4wCitfnDeW6JdatGFEg'
```

### 3. /hello 認証必須コンテンツ

非ユーザーには見せないようにしたいとこ

```
curl -H 'Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MjYyNTYyMDksInN1YiI6ImFkbWluIn0.5ToBUNMG4s50-7yLcvm_dKfC4wCitfnDeW6JdatGFEg' http://localhost:8080/hello
Your name is admin
```


### 4. /refresh トークンの有効期限を延ばす

切れそうなトークン、切れたトークンを延ばす

```
curl -H 'Authorization:Bearer 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MjYyNTYyMDksInN1YiI6ImFkbWluIn0.5ToBUNMG4s50-7yLcvm_dKfC4wCitfnDeW6JdatGFEg' http://localhost:8080/refresh
Your new token is eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0NzM3MzE0MDgsInN1YiI6ImFkbWluIn0.nPpJka3zzUdhVrK-hOV5tRYizmc82cmbfWRvmZNgWGo
```

## OAuth2との違い

やり取りしている内容が、認可して取得した情報を使って認証っぽいことするのではなく、単純な認証ロジックでユーザーを一意に紐付けるだけに特化しているので、ロジックが直感的でわかりやすい。また、外部サービスを使っている場合、認証のロジックを増やすとかも簡単に出来るので、認証周りの保守が楽そう。(例えば、OAuthに加えて、二段階認証とかワンパス追加するの辛いよねっていう)

# 内部実装

## /login 認証

```go
http.Handle("/login", jwth.AuthenticationHandler(http.HandlerFunc(login)))
```

```go
// AuthenticationHandler can be used by clients to authentication and get token.
// Clients must define the username and password getter and the authenticator.
// On success, token is stored in http.Request.Context.
func (h *JwtHandler) AuthenticationHandler(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

    // 1.usernameとpasswordをただ取得するだけ
		username, password := h.LoginDataGetter(r) // JwtHandlerに定義されたLoginDataGetterを使う

    // 2.正しく認証できるか
		if !h.Authenticator(username, password) {
			h.ErrorHandler(w, r, ErrAuthentication)
			return
		}

    // 3.usernameからトークンを作成する
		tokenString, err := h.createSignedToken(h.createToken(username))
		if err != nil {
			h.ErrorHandler(w, r, err)
			return
		}

    // 4.作成されたtokenをContextに保存する
    // 今回の場合、デフォルトのcontextSetterが使用される
		h.ContextSetter(r, signedTokenKey, tokenString)

    // 5.ユーザーが実際に行いたいハンドラへ
		next.ServeHTTP(w, r)
	})
}
```

1.認証に使う情報の取得

```go
type LoginDataGetter func(r *http.Request) (string, string) // username, passwordを最終返せばいいっぽい
```

2.認証ロジック

```go
type Authenticator func(string, string) bool // 認証結果をboolで返すだけ(RDBMSに確認取りに行くとかそういうの)
```

3.トークンの作成

```go
// jwt.Token構造体をユーザー名と有効期限で作成する
func (h *JwtHandler) createToken(username string) *jwt.Token {

	return jwt.NewWithClaims(h.SigningMethod, jwt.MapClaims{
		"sub": username,
		"exp": time.Now().Add(h.Timeout).Unix(),
	})
}
```

3.トークンの文字列作成

```go
func (h *JwtHandler) createSignedToken(token *jwt.Token) (string, error) {

  // どのkeyを使うか？
	var key interface{}
	switch {
	case h.isHmac():
		key = h.HmacKey
	case h.isRsa():
		key = h.RsaPrivateKey
	case h.isEcdsa():
		key = h.EcdsaPrivateKey
	}

  // 作成されたtokenを使ってtokenの文字列を作成する
  // .区切りに連結しているだけ
	tokenString, err := token.SignedString(key)
	if err != nil {
		return "", err
	}

	return tokenString, err
}
```

4.トークンをContextに保存する

```go
func contextSetter(r *http.Request, key interface{}, value interface{}) {
	ctx := r.Context()
	*r = *(r.WithContext(context.WithValue(ctx, key, value)))
}
```

5.保存されたトークンを使って何かする

```go
func login(w http.ResponseWriter, r *http.Request) {
	// 発行されたトークンをコンテキストから取得
	token, ok := jwthandler.SignedTokenFromContext(r.Context())
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
	}
	fmt.Fprint(w, "Your token is "+token)
}
```

```go
// SignedTokenFromContext is signed token string getter from http.Request.Context.
func SignedTokenFromContext(ctx context.Context) (string, bool) {
	val, ok := ctx.Value(signedTokenKey).(string)
	return val, ok
}
```

## /hello 認証必須コンテンツ

```go
http.Handle("/hello", jwth.AuthorizationHandler(http.HandlerFunc(hello)))
```

```
// AuthorizationHandler can be used by clients to authorization token.
// Clients must set the token to Authorization header. Example: "Authorization:Bearer {SIGNED_TOKEN_STRING}"
// On succss, token is stored in http.Request.Context.
func (h *JwtHandler) AuthorizationHandler(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

    // 1.トークンを解析する
		token, err := h.parseToken(r)
		if err != nil {
			h.ErrorHandler(w, r, ErrAuthorization)
			return
		}

    // 2トークンが正しいものかをバリデーション
		if _, ok := token.Claims.(jwt.MapClaims); !ok || !token.Valid {
			h.ErrorHandler(w, r, ErrAuthorization)
			return
		}

    // 3.正しかったトークンをContextへ保存
		h.ContextSetter(r, tokenKey, token)

    // 4.ユーザーの定義したハンドラへ
		next.ServeHTTP(w, r)
	})
}
```

1.トークン情報を取得する

```go
func (h *JwtHandler) parseToken(r *http.Request) (*jwt.Token, error) {
	authHeader := r.Header.Get("Authorization")

  // そもそもHeaderある？
	if authHeader == "" {
		return nil, errors.New("Auth header empty")
	}

  // Bearer句ある？
	parts := strings.SplitN(authHeader, " ", 2)
	if !(len(parts) == 2 && parts[0] == "Bearer") {
		return nil, errors.New("Invalid auth header")
	}

  // 取得したHeaderの値を使って
	return jwt.Parse(parts[1], func(token *jwt.Token) (interface{}, error) {
    // 想定しているメソッドか
		if h.SigningMethod != token.Method {
			return nil, errors.New("Invalid signing algorithm")
		}
		switch {
		case h.isHmac():
			return h.HmacKey, nil
		case h.isRsa():
			return h.RsaPublicKey, nil
		case h.isEcdsa():
			return h.EcdsaPublicKey, nil
		default:
			return nil, errors.New("Invalid signing algorithm")
		}
	})
}
```

1.すっごい頑張ってパースしてた

```
// Parse, validate, and return a token.
// keyFunc will receive the parsed token and should return the key for validating.
// If everything is kosher, err will be nil
func Parse(tokenString string, keyFunc Keyfunc) (*Token, error) {
	return new(Parser).Parse(tokenString, keyFunc)
}
```

4.ユーザー定義のハンドラ

```go
func hello(w http.ResponseWriter, r *http.Request) {
	// 認証に使われたトークンをコンテキストから取得
	token, ok := jwthandler.TokenFromContext(r.Context())
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
	}
	// トークンからユーザーIDを取得
	username, ok := jwthandler.SubjectFromToken(token)
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
	}
	fmt.Fprint(w, "Your name is "+username)
}
```

4.ユーザー名取得。もし、権限情報とかもほしかったらここらへんでどうにかすれば良さそう？？

```
// SubjectFromToken returns claims subject
func SubjectFromToken(token *jwt.Token) (string, bool) {
	if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
		return claims["sub"].(string), true
	}
	return "", false
}
```
