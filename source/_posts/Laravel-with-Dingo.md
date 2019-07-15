---
title: Laravel with Dingo
date: 2019-07-13 17:14:41
categories:
  - laravel
  - dingo
tags:
  - Tech
---
 Laravel使用Dingo扩展包
 参考：
 - [安装Dingo](https://learnku.com/courses/laravel-advance-training/5.8/install-dingoapi/3990)
 - [include机制](https://learnku.com/courses/laravel-advance-training/5.5/list-of-posts/923#Include-%E6%9C%BA%E5%88%B6)
 - [回复列表](https://learnku.com/courses/laravel-advance-training/5.5/reply-list/944)
 
## 安装 Dingo
```bash
$ composer require dingo/api
```
<!--more-->
### 配置

```bash
$ php artisan vendor:publish --provider="Dingo\Api\Provider\LaravelServiceProvider"
```
*.env*

```
.
.
.
# prs 未对外发布的，提供给公司 app，单页应用，桌面应用等
API_STANDARDS_TREE=prs
# 项目的简称
API_SUBTYPE=book
API_PREFIX=api
API_VERSION=v1
API_DEBUG=true
```
## 编写调试接口
*routes/api.php*

```php
<?php

use Illuminate\Http\Request;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/

$api = app('Dingo\Api\Routing\Router');

$api->version('v1', function($api) {
    $api->get('version1', function() {
        return response('this is version v1');
    });
});

$api->version('v2', function($api) {
    $api->get('version2', function() {
        return response('this is version v2');
    });
});
```
v1: GET http://book.test/api/version1
v2: GET Headers 中`Accept:application/prs.book.v2+json,
`,`Accept: application/<API_STANDARDS_TREE>.<API_SUBTYPE>.v2+json`
http://book.test/api/version2

## Transformers
*app/Transformers/BookTransformer.php* 

```php
<?php
namespace App\Transformers;
use League\Fractal\TransformerAbstract;
use App\History;

class HistoryTransformer extends  TransformerAbstract
{
    protected $availableIncludes = ['book'];
    public function transform(History $history)
    {

        return [
            'id' => $history->id,
            'user_id' => $history->user_id,
            'book_id' => $history->book_id,
            'borrow_state' => $history->borrow_state,
        ];
    }

    // $history 依赖注入,$history->book，history与book的关联，
    // 关联定义在model中
    public function includeBook(History $history)
    {
        return $this->item($history->book,new BookTransformer());
    }
}```
继承`League\Fractal\TransformerAbstract`类，实现transform方法。
include机制，返回history信息时，返回额外的book信息
### include机制的调试
某个用户借阅的所有图书
`{{book}}/users/:id/borrowed/histories?include=book`
响应：

```json
{
    "data": [
        {
            "id": 16,
            "user_id": 1,
            "book_id": 11,
            "borrow_state": 1,
            "book": {
                "id": 11,
                "isbn": "98788199293",
                "title": "testtitle",
                "author": "testauthor",
                "owner": "testowner",
                "total_borrow": 0,
                "missing_num": 0,
                "total_num": 1
            }
        }
    ]
}
```

## Dingo路由隐式绑定
*routes/api.php*
```php
$api->version(‘v1’,[
    ‘namespace’ => ‘App\Http\Controllers\Api’,
    ‘middleware’ => [‘serializer:array’, ‘bindings’]
],/function/($api){
   $api->get(‘/books/{book}’,’BooksController@show’)
    ->name(‘api.books.show’); 
});
```
增加bindings中间件。

*app/Http/Controllers/Api/BooksController.php*

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Requests\Api\BookRequest;
use App\Book;
use App\Transformers\BookTransformer;

class BooksController extends Controller
{
    .
    .
    .
    public function show(Book $book)
    {
        return $this->response->item($book,new BookTransformer());
    }

}

```
postman中
http://book.test/api/book/:id

![](http://ww2.sinaimg.cn/large/006tNc79gy1g50rxhedslj31kk0suaaz.jpg)

## 分页
*app/Http/Controllers/Api/BooksController.php*

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Requests\Api\BookRequest;
use App\Book;
use App\Transformers\BookTransformer;

class BooksController extends Controller
{
    public function index()
    {
        $books =Book::paginate(10);

        return $this->response->paginator($books,new BookTransformer());
    }
        .
        .
        .
}

```
