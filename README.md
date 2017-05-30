# Omniauth Try 

常見的登入或是認證方式
1.自行認證
2.把認證交給其他網站，如FB、Google

認證成功後再導回欲顯示的頁面

一開始先建立一個app，我們在terminal輸入以下的code
```
$ rails new app_name
```

其中

## rails g controller Pages


切換到該資料夾，執行
```
$ rails db:create
```
這樣可以依照目前的 RAILS_ENV 環境建立資料庫。

接著我們要安裝dvise這個gem。先到Gemfile加上devise
```ruby
# 略...
gem 'devise'
```
執行`bundle install`，然後在terminal執行以下的code，完成安裝。
```
$ rails generate devise:install
```

接著產生User這個model

```
$ rails g devise User
```

```
$ rails g devise views
```

```
$ bundle exec rake db:migrate
```

在完成了devise的安裝之後，下一步我們需要到[Facebook for developer](https://developers.facebook.com/)這個網站建立一個帳號，這樣我們才能取得ＸＸＸ

如果你有FB的帳號的話你可以直接建立devloper的帳號，接著可以建立新的應用程式編號

![](https://i.imgur.com/nFgpxgH.png)

接下來要在設定中新增平台，並在網址的部分輸入`http://localhost:3000/`，這樣才能找到正確的路徑。
![](https://i.imgur.com/lV5U70y.png)

否則到時候點擊登入按鈕時就會出現以下的錯誤。

![](https://i.imgur.com/9nYLqZV.png)



## 這樣看來似乎不同的帳號登入時就要安裝各自的gem

我們需要在Gemfile加入這個gem`omniauth-facebook`


## 我的電腦似乎不能用postgre!

為了要讓database能夠儲存登入的資料，所以我們要在User table上新增幾個欄位，讓我們能夠順利的存取並使用資料。[參考資料](https://github.com/rails-camp/facebook-omniauth-demo#update-the-user-table-with-the-params-needed)
在terminal裡執行以下的code
```
$ rails g migration AddOmniauthToUsers provider:string uid:string name:string image:text
```
這樣我們就新增了provider、uid、name還有image這幾個欄位

而既然我們更改了schema，我們要記得執行
```
$ rails db:migrate
```

接著我們需要update initializer（確認一下這是幹嘛的（api connecter?））

問：什麼是initializer?
>  An initializer is any Ruby file stored under config/initializers in your application. You can use initializers to hold configuration settings that should be made after all of the frameworks and gems are loaded, such as options to configure settings for these parts.


讓我們移到`config/initializers/devise.rb`這個檔案並加入以下的code
```ruby
config.omniauth :facebook, "App ID", "App Secret", callback_url: "http://localhost:3000/users/auth/facebook/callback"
```
在App ID、App Secret的部分可以在FB devloper的主控版找到，分別是應用程式編號和應用程式密鑰。

> http://localhost:3000/users/auth/facebook/callback

其中localhost:3000是，users是path，auth是extension，facebook是表明透過facebook做認證


下一步我們要做的是[Update Model](https://github.com/rails-camp/facebook-omniauth-demo#update-the-model)

給予user model permission給Omniauto model，所以我們要在`user.rb`這個model中加入以下的code`devise :omniauthable, :omniauth_providers => [:facebook]`並改寫。


## 問：Model這邊的devise是怎樣？
```ruby
# user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable,
         :omniauthable, :omniauth_providers => [:facebook]
end
```

其中omniauth_provider這一欄是表示提供登入的平台是誰，所以假設之後我想要加入google帳號登入的話，我可以在User這個model加入`:omniauth_providers => [:google_oauth2]`（為什麼是google_oauth2是因為我使用了[omniauth-google-oauth2](https://github.com/zquestz/omniauth-google-oauth2#devise)這個gem。

 接著為了讓使用者可以在進入頁面的時候登入，我們要在`app/views/pages/home.html.erb`這個檔案中加入以下的code
 
 ```html
 # app/views/pages/home.html.erb
 <% unless current_user %>
  <%= link_to "Sign in with Facebook", user_facebook_omniauth_authorize_path %>
<% else %>
  <%= link_to "Logout", destroy_user_session_path, method: :delete %>
<% end %>
 ```
 接著我們要更新我們的route，在`config/routes.rb`當中加入以下的code。
 
 ```ruby
 # config/routes.rb
devise_for :users, :controllers => { :omniauth_callbacks => "users/omniauth_callbacks" }
 ```
 
 ## 問：為什麼要建立一個User directory然後再建立一個omniauth_callbacks_controller.rb？
 首先建立一個users directory，然後在底下建立一個`omniauth_callbacks_controller.rb`檔案
 
 ```ruby
 class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def facebook
    @user = User.from_omniauth(request.env["omniauth.auth"])

    if @user.persisted?
      sign_in_and_redirect @user, :event => :authentication
      set_flash_message(:notice, :success, :kind => "Facebook") if is_navigational_format?
    else
      session["devise.facebook_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url
    end
  end

  def failure
    redirect_to root_path
  end
end
 ```
 
> persisted?() public
Returns true if the record is persisted, i.e. it’s not a new record and it was not destroyed, otherwise returns false.

 接著在model中的`user.rb`新增以下的method
 ```ruby
 # app/model/user.rb
 def self.new_with_session(params, session)
  super.tap do |user|
    if data = session["devise.facebook_data"] && session["devise.facebook_data"]["extra"]["raw_info"]
      user.email = data["email"] if user.email.blank?
    end
  end
end

def self.from_omniauth(auth)
  where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
    user.email = auth.info.email
    user.password = Devise.friendly_token[0,20]
    user.name = auth.info.name   # assuming the user model has a name
    user.image = auth.info.image # assuming the user model has an image
  end
end
 ```
 
 如果這個user存在的話就選取他，沒有則創造
 
 

 
 
 為了測試資料，我們可以在`show.html.erb`這個檔案中加入 
```
<h1>Homepage</h1>
<% if current_user %>
  <%= current_user.inspect %>
<% end %>

<% unless current_user %>
  <%= link_to "Sign in with Facebook", user_facebook_omniauth_authorize_path %>
<% else %>
  <%= link_to "Logout", destroy_user_session_path, method: :delete %>
<% end %>
```

> 另一個顯示物件數值的有用方法是 inspect，對陣列或 Hash 尤其有用。會把物件的數值以字串形式印出，譬如：
><%= [1, 2, 3, 4, 5].inspect %>
><p>
><b>Title:</b>
><%= @article.title %>
></p>
>算繪出來為：
>[1, 2, 3, 4, 5]
>Title: Rails debugging guide

 
接著讓我們回到首頁，我們應該會看到以下的畫面
![](https://i.imgur.com/vPdjMcY.png)

點擊之後則會出現確認按鈕。
![](https://i.imgur.com/i0VdX5a.png)

被導回頁面後應該會看到以下的畫面

![](https://i.imgur.com/x4Cx1Xa.png)


稍微簡單修改一下html後，我們應該可以看到頁面取用了我的名字還有我使用的FB大頭貼檔案

如果讓我們用rails c查看的話，我們可以看到在database裡面已經建立了一筆我的個人資料

![](https://i.imgur.com/dXu3g0g.png)


登入一次之後就不需要重複認證


使用devise的各種網站，他們的code看起來差不多，應該是有效的被devise整合了

## Google認證登入

唯一要注意的是可能會出現ＸＸＸ，在這種情況下通常
要等上一段時間

參考資料：
[Google OAuth 2 authorization - Error: redirect_uri_mismatch](https://stackoverflow.com/questions/11485271/google-oauth-2-authorization-error-redirect-uri-mismatch)

https://somescooby.wordpress.com/2015/11/19/rails-authenticate-with-google-and-devise/

## GitHub認證登入

GitHub的登入也差不多，首先先讓我們到github的[application頁面](https://github.com/settings/applications/)設定。

我們要先建立一個application並產生金鑰，所以先點選左側邊欄下方的OAuth applications。

Homepage URL使用`http://localhost:3000/`，Authorization callback URL使用`http://localhost:3000/users/auth/github/callback/`

問：為什麼是`http://localhost:3000/users/auth/github/callback/`，而不是`http://localhost:3000/auth/github/callback/`？

![](https://i.imgur.com/CFw1Z6B.png)


在建立完金鑰後讓我們回到資料夾，在`app/model/user.rb`這個檔案中`:omniauth_provider`這個key所對應的array中加入`:github`後，就可以產生相對應的routes

```ruby
# app/model/user.rb
devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable,
         :omniauthable, :omniauth_providers => [:facebook, :google_oauth2, :github]       
# 略         
```

在`app/views/pages/home.html.erb`加入使用GitHub認證的連結
```
<%= link_to "Sign in with GitHub", user_github_omniauth_authorize_path %>
```

在`config/initializer/devise.rb`這個檔案中設定github的config

```ruby
Devise.setup do |config|
    config.omniauth :github, 'App ID', 'App Secret', :scope => 'user:email'
end
```

在`app/users/omniauth_callbacks_cotroller.rb`加上對應的method

```ruby
# app/users/omniauth_callbacks_cotroller.rb

def github
    @user = User.from_omniauth(request.env["omniauth.auth"])
    if @user.persisted?
      sign_in_and_redirect @user, :event => :authentication
      set_flash_message(:notice, :success, :kind => "Github") if is_navigational_format?
    else
      session["devise.github_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url, alert: @user.errors.full_messages.join("\n")
    end
  end
```

這樣就可以使用GitHub認證登入了

How it works?

There are basic steps for user handling by Auth with Omniauth:

1.User click on 'auth with github'
2.Rails app redirect him to github page
3.User sign in on github and grant access to your app
4.Github redirect user back to your rails app
5.Rails app got at least two things: provider and uid
6.Now we can use this info to sign in user in app



參考資料：
1.[Guide for Integrating Omniauth in Rails 5 for Facebook Login Feature](https://www.crondose.com/2016/12/guide-integrating-omniauth-rails-5-facebook-login-feature/)
2.[Rails. Omniauth with devise (github example)](https://www.codementor.io/anaumov/rails-omniauth-with-devise--github-example-du107rmn7)
