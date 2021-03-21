# トークン認証

## Userモデル
<table>
<tr>
<td>
id
</td>
</tr>  
<tr> 
<td>
name: string
</td>
<tr>
<td>
email: string
</td>
</tr>  
<tr>
<td>
crypted_password: string
</td>
</tr>  
<tr>
<td>
salt: string
</td>
</tr>  
<tr>
<td>
created_at: datetime
</td>
</tr>  
<tr>
<td>
updated_at: datetime
</td>
</tr>  
</table>

## 使用するgem
```
gem 'bcrypt', '~> 3.1.7'　　#DBに保存するパスワードを暗号化

gem 'jwt'　　　　　　　　　　　#jwt認証に必要
```

## routes
```
Rails.application.routes.draw do
  namespace 'api' do
    namespace 'v1' do
     
      resource :users, only: [:create]
      post "/login", to: "auth#login"
      get "/auto_login", to: "auth#auto_login"
    end
  end
end
```

## UserController
### createメソッド
- ユーザーがサインアップする
- 新しいユーザーインスタンスが作成される
- クライアントサイドからPOSTリクエストが行われる
- ユーザーインスタンスが有効だった場合、ユーザーidを使用してトークンを生成
- 生成したトークンをクライアントサイドへ渡す

```
class Api::V1:: UsersController < ApplicationController
  skip_before_action :require_login, only: [:create]
  def create
    user = User.create(user_params)
    if user.valid?
      payload = {user_id: user.id}
      token = encode_token(payload)
      render json: {user: user.user_id, jwt: token}
    else
      render json: {errors: user.errors.full_messages},status: :not_acceptable
    end
   end
 private
 
 def user_params
   params.permit(:email,:password)
 end
end
```

## AuthController
### loginメソッド
- クライアントサイドから受け取ったメールアドレスと一致するものがあるかDBを検索（find_by）
- 一致するメールアドレスが存在する場合、authenticateメソッドを使ってパスワードが一致するか確認
- パスワードが有効だった場合、encode_tokenメソッドを使ってトークンを生成
- 生成したトークンをクライアントに渡す

```
class Api::V1::AuthController < ApplicationController
  skip_before_action :require_login, only: [:login, :auto_login]
  def login
    user = User.find_by(email: params[:email])
    if user && user.authenticate(params[:password])
      payload = {user_id: user.id,email: user.email}
      token = encode_token(payload)
      render json: {jwt: token,success: "Welcome back, #{user.email}"}
    else
      response_unauthorized
    end
  end
end
```

### auto_loginメソッド
- ログインしたユーザーが保持するトークンが適切かどうか検証
- `session_user`メソッドはApplicationControllerに定義

```
def auto_login
  if session_user 
    render json: session_user
  else
    render json: {errors: "No user Logged In"}
  end
end
```
 
## JWTトークン関連メソッド（ApplicationController）
### encode_tokenメソッド
- `user#create`や`auth#login`などで使用される

```
def encode_token(payload)
     JWT.encode(payload,'my_secret_key','HS256')
end
```
### session_userメソッド
- decoded_tokenの値が存在する場合、user_idを取り出してDBを検索
- 一致するuser_idが存在していたら、auto_loginメソッドに返して、クライアントサイドにデータを渡す


```
def session_user
  decoded_hash = decoded_token
  if !decoded_hash.empty?
    user_id = decoded_hash[0]['user_id']
    @user = User.find_by(id: user_id)
  else 
    nil
  end
end
```
### auth_headerメソッド
- トークンはheaderのAuthorizationにAuthorization：bearer<token>で含まれる

```
def auth_header
  request.headers['Authorization']
end 
```


### decoded_tokenメソッド
- auth_headerからトークンの部分だけ取り出して検証
- 暗号化の作業と同様、以下の値を引数にとる
> 第一引数にトークン
> 第二引数に秘密鍵
> 第三引数はトークンの検証を行うかの真偽値（行うのでtrue）
> 第四引数はアルゴリズムの種類を指定

```
def decoded_token
  if auth_header
     token = auth_header.split(' ')[1]
     begin
       WT.decode(token, 'my_secret', true,algorithm: 'HS256')
     rescue JWT::DecodeError
       []
     end
  end
end
```












