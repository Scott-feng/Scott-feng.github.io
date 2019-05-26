---
title: laravel factories
date: 2019-05-26 08:56:14
categories:
  - Tech
tags:
  - laravel
  - seeder
---
## Laravel 数据填充

参考：
- [假数据填充](https://learnku.com/courses/laravel-intermediate-training/5.8/seeding-data/4163)
- [数据填充](https://learnku.com/docs/laravel/5.8/seeding/3929)
- [模型工厂](https://learnku.com/docs/laravel/5.8/database-testing/3940#writing-factories)
- [make-your-laravel-seeder-using-model-factories](https://scotch.io/@wisdomanthoni/make-your-laravel-seeder-using-model-factories)

```php
//文件名单数
php artisan make:factory ThreadFactory --model=Thread
php artisan make:factory ReplyFactory --model=Reply
```
<!--more-->
### 创建模型工厂文件
database/factories/UserFactory.php
```php
<?php

/** @var \Illuminate\Database\Eloquent\Factory $factory */
use App\User;
use Illuminate\Support\Str;
use Faker\Generator as Faker;

/*
|--------------------------------------------------------------------------
| Model Factories
|--------------------------------------------------------------------------
|
| This directory should contain each of the model factory definitions for
| your application. Factories provide a convenient way to generate new
| model instances for testing / seeding your application's database.
|
*/

$factory->define(User::class, function (Faker $faker) {
//bcrtpy消耗cpu的函数，静态变量不用每次重新计算
    static $password;
    return [
        'name' => $faker->name,
        'email' => $faker->unique()->safeEmail,
        'email_verified_at' => now(),
        'password' => $password ?: $password = bcrypt('123456'), // password
        'remember_token' => Str::random(10),
    ];
});

```

database/factories/ThreadFactory.php
```php
<?php

/* @var $factory \Illuminate\Database\Eloquent\Factory */

use App\Thread;
use Faker\Generator as Faker;

$factory->define(Thread::class, function (Faker $faker) {
    return [
        //user_id字段在seeder文件中处理
        'title'=>$faker->sentence,
        'body'=>$faker->paragraph,
    ];
});

```
database/factories/ReplyFactory.php
```php
<?php

/* @var $factory \Illuminate\Database\Eloquent\Factory */

use App\Reply;
use Faker\Generator as Faker;

$factory->define(Reply::class, function (Faker $faker) {
    return [
    //user_id,thread_id在seeder中处理
        'body'=>$faker->paragraph,
    ];
});

```


### 创建数据填充文件

```php
//文件名复数
php artisan make:seed UsersTableSeeder
php artisan make:seed ThreadsTableSeeder
```
database/seeds/UsrsTableSeeder
```php
<?php

use Illuminate\Database\Seeder;
use App\User;

class UsersTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
       factory(User::class,50)->create();
    }
}

```

database/seeds/TreadsTableSeeder

```php
<?php

use Illuminate\Database\Seeder;
use App\Thread;

class ThreadsTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        //取得所有的用户id，转换为数组
        $user_ids =\App\User::all()->pluck('id')->toArray();
        $faker = app(Faker\Generator::class);

        //make() method return collection,collection is in memory
        $threads = factory('App\Thread',90)->make()->each(function ($thread) use($user_ids,$faker){
            $thread->user_id = $faker->randomElement($user_ids);
        });

    //store to database
        Thread::insert($threads->toArray());
    }
}

```

database/seeds/RepliesTableSeeder

```php
<?php

use Illuminate\Database\Seeder;
use App\User;
use App\Thread;
use App\Reply;

class RepliesTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        $user_ids = User::all()->pluck('id')->toArray();
        $faker = app(Faker\Generator::class);
        $thread_ids = Thread::all()->pluck('id')->toArray();

        //collection
        $replies = factory('App\Reply',100)->make()->each(function ($reply) use($user_ids,$thread_ids,$faker){
           $reply->user_id = $faker->randomElement($user_ids);
           $reply->thread_id = $faker->randomElement($thread_ids);
        });

        Reply::insert($replies->toArray());
    }
}

```

### 运行 Seeders
重新生成composer的加载器
`composer dump-autoload`


```
php artisan db:seed
//运行特定的seeder
php artisan db:seed --class=UsersTableSeeder
```

`migrate:refresh` 这个命令来填充数据库，该命令会回滚并重新运行所有迁移。这个命令可以用来重建数据库：
```
php artisan migrate:refresh --seed
```
