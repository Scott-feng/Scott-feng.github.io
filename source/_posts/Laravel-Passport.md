---
title: Laravel Passport
date: 2019-06-01 10:41:17
categories:
  - Tech
tags:
  - laravel
  - passport
  - api
---
## 使用Laravel Passport 实现api用户认证
参考：
- [使用Laravel Passport](https://learnku.com/laravel/t/22586#db06c7)


### Passport配置
```php
composer require laravel/passport

//表迁移，生成oauth相关的表
php artisan migrate

php artisan passport:install
```
<!--more-->
*app/User.php*
```php
<?php
...
use Laravel\Passport\HasApiTokens;
class User extends Authenticatable 
{
    use Notifiable,HasApiTokens;
}
```
引入 **HasApiTokens**

*app/Providers/AuthServiceProvider.php*
```php
<?php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Laravel\Passport\Passport;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
         'App\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        //引入Ppassport routes,
        //注册必要的路由去颁发访问令牌，撤销访问令牌，客户端和个人令牌
        Passport::routes();

        
    }
}


```

*config/auth.php*
```php
'api' => [
    //driver 改为passport
    'driver' => 'passport',
    'provider' => 'users',
    'hash' => false,
],
```

### 添加api 路由
*routes/api.php*
```php
<?php
Route::group([
    'prefix' => 'auth'
], function () {
    Route::post('login', 'AuthController@login');
    Route::post('signup', 'AuthController@signup');

    Route::group([
      'middleware' => 'auth:api'
    ], function() {
        Route::get('logout', 'AuthController@logout');
        Route::get('user', 'AuthController@user');
    });
});
```

### 创建控制器
*app/Http/Controllers/AuthController.php*


```php
<?php

namespace App\Http\Controllers;

use App\User;
use Carbon\Carbon;
use Illuminate\Http\Request;
use Auth;
use Illuminate\Support\Facades\Log;
class AuthController extends Controller
{
    public function signup(Request $request)
    {
        $request->validate([
            'name' => 'required|string',
            'email' => 'required|string|email',
            'password' => 'required|string',
        ]);

        $user = new User([
            'name'=>$request->name,
            'email'=>$request->email,
            'password'=>bcrypt($request->password)
        ]);

        $user->save();

        return response()->json([
            'message'=>'create user success'
        ],201);

    }

    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|string|email',
            'password' => 'required|string',
            'remember' => 'boolean'
        ]);

        $credentials = request(['email','password']);

        if (!Auth::attempt($credentials)) {
            return response()->json([
                'message' => 'unauthorized'
            ],401);
        }

        $user = $request->user();

        $tokenResult = $user->createToken('Personal Access Token');
        Log::info($tokenResult);
        $token = $tokenResult->token;

        if ($request->remember_me) {
            $token->expires_at = Carbon::now()->addWeeks(1);
        }

        $token->save();
        return response()->json([
            'access_token' => $tokenResult->accessToken,
            'token_type' => 'Bearer',
            'expires_at' => Carbon::parse(
                $tokenResult->token->expires_at
            )->toDateTimeString()
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->token()->revoke();

        return response()->json([
            'message' => 'logged out'
        ]);
    }

    public function user(Request $request)
    {
        return response()->json($request->user());
    }
}

```
`public function login(Request $request)`中的

`$tokenResult = $user->createToken('Personal Access Token');`

$tokenResult的数据结构

```
{"accessToken":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp0aSI6IjcxNDQyZmVjMzBhNGI3ZWVjMmY2YTMyOTQwYjkyNjkzNzRjNTE1MTNjOWFhMzVlYTY1ODRl......",
"token":{"id":"71442fec30a4b7eec2f6a32940b9269374c51513c9aa35ea6584e051e6e21075f2255c765028a44a",
"user_id":52,"client_id":1,
"name":"Personal Access Token","scopes":[],"revoked":false,"created_at":"2019-06-01 02:30:45","updated_at":"2019-06-01 02:30:45",
"expires_at":"2020-06-01 02:30:45"}
} 
```
`public function logout(Request $request)`调用
`$request->user()->token()->revoke();`
注销用户

### Postman测试

[Run in Postman](https://app.getpostman.com/run-collection/f22c66cf3d144b138982)
其中的form.test改为自己的项目域名。

signup
**Headers** 设置
`Content-Type:application/json`
`X-Requested-With:XMLHttpRequest`
![sigup](http://ww2.sinaimg.cn/large/006tNc79gy1g3m03alns3j31hs0u0ad7.jpg)

login
![Login](http://ww4.sinaimg.cn/large/006tNc79gy1g3mg8932yfj30s60f7gml.jpg)

logout
![logout](http://ww4.sinaimg.cn/large/006tNc79gy1g3mgadyp8oj30s80dpglw.jpg)

user
![user](http://ww1.sinaimg.cn/large/006tNc79gy1g3mgbgfwbqj30rq0e8t92.jpg)







