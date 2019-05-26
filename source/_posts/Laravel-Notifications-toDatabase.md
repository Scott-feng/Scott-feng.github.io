---
title: Laravel Notifications toDatabase
date: 2019-05-18 07:51:22
categories:
  - Tech
tags:
  - laravel
  - notifications
---
参考:
[消息通知](https://learnku.com/docs/laravel/5.8/notifications/3921)
### Prepare notification database
```php
	php artisan notifications:table
	php artisan migrate

	#add users table notification_count column
	php artisan make:migration add_notification_count_to_users_table --table=users
```
### Generate notification class
```php
php artisan make:notification 
```
<!--more-->
app/Notifications/TopicReplied

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use App\Models\Message;
use Auth;

class TopicReplied extends Notification
{
    use Queueable;

    /**
     * Create a new notification instance.
     *
     * @return void
     */
    public $message;
    public $topic;

    public function __construct(Message $message)
    {
        $this->message = $message;
    }

    /**
     * Get the notification's delivery channels.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function via($notifiable)
    {
        return ['database'];
    }



    /**
     * Get the array representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function toArray($notifiable)
    {

        return [
            'from'=>Auth::user()->name,
            'message'=>$this->message->content,
        ];
    }
}

```



### Get all notifications,use Notifiable trait
```php
	$notifications = $user->notifications()->paginate(20);
	# get the serialize data by toArray() method
	$notification->data toArray() data
	
	$user->notify(new TopicReplied($message))
```
### MarkAsRead
```php
	$user->unreadNotifications->markAsRead();
```