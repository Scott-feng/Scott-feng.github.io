---
title: Laravel websockets
date: 2019-05-16 07:00:55
categories:
  - Tech
tags:
  - laravel
  - websockets
---
Laravel 配合laravel-echo 使用 websockets

参考
[Laravel Websockets][1] 
[广播系统][2]
[Build a chat app with laravel][3]
<!--more-->
### 配置
```php
composer require pusher/pusher-php-server "~3.0"
composer require beyondcode/laravel-websockets
composer require predis/predis


// a migration to store statistic information
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"

php artisan migrate

// publish the WebSocket configuration file
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"

```

.env
>BROADCAST_DRIVER=pusher
QUEUE_CONNECTION=sync //设为sync方便调试,生产环境设为redis

config/broadcasting.php
```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'encrypted' => true,
        'host' => '127.0.0.1',
        'port' => 6001,
        'scheme' => 'http'
    ],
],
```
config/app.php
>App\Providers\BroadcastServiceProvider::class,

**取消注释,否则auth/broadcasting not found 错误**

### 后端部分
> php artisan make:event ExampleEvent

App\Events\ExampleEvent.php
```php
<?php

namespace App\Events;

use App\Message;
use App\User;
use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */

    public $user;
    public $message;
    public function __construct(User $user,Message $message)
    {
        $this->user = $user;
        $this->message = $message;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('chat');
    }


}


```

routes/channel.php
```php
Broadcast::channel('chat', function ($user) {
    return \Auth::check();
});
```

使用laravel login 脚手架
php artisan make:auth
`protected $redirectTo = '/';`
**Auth/RegisterController,Auth/LoginController Auth/ResetPasswordController**

routes/web.php
```php
Auth::routes();

Route::get('/', 'ChatsController@index');
Route::get('messages', 'ChatsController@fetchMessages');
Route::post('messages', 'ChatsController@sendMessage');
```

>php artisan make:controller ChatsController

Controllers/ChatsController
```php
<?php

namespace App\Http\Controllers;

use App\Events\MessageSent;
use App\Message;
use Illuminate\Http\Request;
use Auth;

class ChatsController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
    }


    public function index()
    {
        return view('chat');
    }


    /**
     * @return Message
     */
    public function fetchMessages()
    {
        return Message::with('user')->get();
    }

    /**
     * @param Request $request
     * @return array
     */
    public function sendMessage(Request $request)
    {
        $user = Auth::user();

        $message = $user->messages()->create([
           'message'=>$request->input('message')
        ]);

        //只广播给其他人
        broadcast(new MessageSent($user,$message))->toOthers();

        return ['status'=>'Message sent!'];

    }

}

```

### 安装前端依赖
>npm install --save laravel-echo pusher-js

bootstrap.js
```js
import Echo from "laravel-echo"

window.Pusher = require('pusher-js');

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'b540cc10ff9a76b8ee18', //your pusher key
    wsHost: window.location.hostname,
    wsPort: 6001,
    disableStats: true,
});
```
app.js
```js
window.Echo.private('chat')
            .listen('MessageSent',(e)=>{
               this.messages.push({
                    message: e.message.message,
                    user: e.user
                });
            });
```
增加wsHost,wsPort 指向Laravel WebSocket server。
> npm run watch

编译css,js

### 启动WebSocket server
>php artisan websockets:serve


### Debugging
http://tasks.test/laravel-websockets

总结：
1.后端生成事件，依赖注入要发送的对象。
2.使用broadcast()方法广播事件
3.前端Echo接收频道以及频道的特定事件。

Note:

 - 使用队列处理事件，启动php artisan queue:work,调试的时候QUEUE_DRIVER= sync
 - 前端更改代码，npm run watch，刷新后看到效果
 - App\Providers\BroadcastServiceProvider::class要取消注释。


  [1]: https://docs.beyondco.de/laravel-websockets/
  [2]: https://learnku.com/docs/laravel/5.8/broadcasting/3914
  [3]: https://pusher.com/tutorials/chat-laravel