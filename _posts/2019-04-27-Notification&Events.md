---
title: Why Jekyll with GitBook
author: Tao He
date: 2019-04-27
category: Jekyll
layout: post
---

07. 通知とイベント

新しいChirpが作成されたときにメール通知を送信することで、Chirperを次のレベルへ引き上げましょう。

Laravelでは、メール送信のサポートに加え、メール、SMS、Slackなど様々な配信チャンネルで通知を送信することができます。さらに、コミュニティで作られた様々な通知チャネルが作成されており、何十もの異なるチャネルで通知を送ることが可能です また、通知はデータベースに保存され、Web インターフェースに表示されることがあります。

#通知の作成
Artisanは、次のコマンドで、またしても、私たちのためにすべてのハードワークを行うことができます。
```
php artisan make:notification NewChirp
```

これで、app/Notifications/NewChirp.phpに新しいnotificationが作成され、カスタマイズする準備が整いました。

NewChirpクラスを開いて、先ほど作成したChirpを受け取り、メッセージをカスタマイズして、作者の名前とメッセージのスニペットを入れてみましょう。

```
app/Notifications/NewChirp.php
<?php
namespace App\Notifications;
 
use App\Models\Chirp;
use Illuminate\Bus\Queueable;
use Illuminate\Support\Str;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;
 
class NewChirp extends Notification
{
    use Queueable;
 
    /**
     * Create a new notification instance.
     *
     * @return void
     */
    public function __construct()
    public function __construct(public Chirp $chirp)
    {
        //
    }
 ...
    /**
     * Get the mail representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return \Illuminate\Notifications\Messages\MailMessage
     */
    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->line('The introduction to the notification.')
                    ->action('Notification Action', url('/'))
                    ->subject("New Chirp from {$this->chirp->user->name}")
                    ->greeting("New Chirp from {$this->chirp->user->name}")
                    ->line(Str::limit($this->chirp->message, 50))
                    ->action('Go to Chirper', url('/'))
                    ->line('Thank you for using our application!');
    }
 ...
}

```

ChirpControllerクラスのstoreメソッドから直接通知を送ることもできますが、これはコントローラの作業を増やすことになり、特にデータベースへの問い合わせとメール送信を行うことになるので、逆にリクエストを遅くしてしまう可能性があります。

その代わりに、アプリケーションの動作の軽快さを保つために、バックグラウンドのキューで待機し、処理できるようなイベントをディスパッチしましょう。

#イベントの作成

イベントは、アプリケーションのさまざまな側面を切り離すのに最適な方法です。なぜなら、ひとつのイベントには複数のリスナーが存在し、互いに依存しないからです。

次のコマンドで新しいイベントを作成してみましょう。
```
php artisan make:event ChirpCreated
```

これで、app/Events/ChirpCreated.phpに新しいイベントクラスが作成されます。

ChirpCreatedイベントを更新して、新しく作成されたChirpを受け取り、それをnotificationに渡します。


```
app/Events/ChirpCreated.php
<?php
 
namespace App\Events;
 
use App\Models\Chirp;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
 
class ChirpCreated
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
 
    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()
    public function __construct(public Chirp $chirp)
    {
        //
    }
 ...
}
```


イベントのディスパッチ

イベントクラスができたので、Chirpが作成されたときにいつでもそれをディスパッチできるようになりました。アプリケーションのライフサイクルのどこででもイベントをディスパッチできますが、今回のイベントはEloquentモデルの作成に関連しているので、Chirpモデルが私たちのためにイベントをディスパッチするように構成することができます。


```
app/Models/Chirp.php
<?php
 
namespace App\Models;
 
use App\Events\ChirpCreated;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
 
class Chirp extends Model
{
    use HasFactory;
 
    protected $fillable = [
        'message',
    ];
 
    protected $dispatchesEvents = [
        'created' => ChirpCreated::class,
    ];
 
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```


これで、新しい Chirp が作成されるたびに、ChirpCreated イベントがディスパッチされるようになりました。

#イベントリスナーの作成

さて、イベントをディスパッチしたところで、そのイベントをリスニングして通知を送信する準備が整いました。

ChirpCreated イベントを受信するリスナーを作成しましょう。

```
php artisan make:listener SendChirpCreatedNotifications --event=ChirpCreated
```

新しいリスナーは、app/Listeners/SendChirpCreatedNotifications.php に配置されます。それでは、通知を送信するためにリスナーを更新してみましょう。

```
app/Listeners/SendChirpCreatedNotifications.php
<?php
 
namespace App\Listeners;
 
use App\Events\ChirpCreated;
use App\Models\User;
use App\Notifications\NewChirp;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
 
class SendChirpCreatedNotifications
class SendChirpCreatedNotifications implements ShouldQueue
{
 ...
    /**
     * Handle the event.
     *
     * @param  \App\Events\ChirpCreated  $event
     * @return void
     */
    public function handle(ChirpCreated $event)
    {
        //
        foreach (User::whereNot('id', $event->chirp->user_id)->cursor() as $user) {
            $user->notify(new NewChirp($event->chirp));
        }
    }
}
```



リスナーをShouldQueueインターフェースでマークしました。これは、リスナーがキューで実行されるべきだとLaravelに伝えるものです。デフォルトでは、「sync」キューがジョブを同期的に処理するために使われます。しかし、バックグラウンドでジョブを処理するためにキューワーカーを設定することもできます。

また、Chirpの作者を除く、プラットフォーム内のすべてのユーザに通知を送るようにリスナーを設定しました。実際には、これはユーザを困らせる可能性があるので、「フォロー」機能を実装して、ユーザがフォローしているアカウントに対してのみ通知を受け取るようにするとよいでしょう。

ここでは、すべてのユーザを一度にメモリに読み込まないように、データベースカーソルを使用しました。


本番環境では、このようにユーザーが通知の受信を解除できるようにする必要があります。

#イベントリスナーの登録

最後に、イベントリスナーをイベントにバインドしましょう。これは、対応するイベントがディスパッチされたときに、イベントリスナーを呼び出すようにLaravelに指示します。これは、EventServiceProviderクラス内で行うことができます。


```
App\Providers\EventServiceProvider.php
<?php
 
namespace App\Providers;
 
use App\Events\ChirpCreated;
use App\Listeners\SendChirpCreatedNotifications;
use Illuminate\Auth\Events\Registered;
use Illuminate\Auth\Listeners\SendEmailVerificationNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Event;
 
class EventServiceProvider extends ServiceProvider
{
    /**
     * The event to listener mappings for the application.
     *
     * @var array<class-string, array<int, class-string>>
     */
    protected $listen = [
        ChirpCreated::class => [
            SendChirpCreatedNotifications::class,
        ],
 
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];
 ...
}
```

テストする

DockerとLaravel Sailで開発している場合、MailHogというメールテストツールを利用すると、アプリケーションからのメールをキャッチして、閲覧することができます。

今回は、Chirpの作者には通知しないように設定しましたので、最低でも2つのユーザーアカウントを登録しておいてください。次に、新しい Chirp を投稿して、通知を開始します。

Web ブラウザで MailHog を開き、http://localhost:8025/ に移動すると、今チャープしたメッセージの通知がある inbox が見つかります!

実運用でメールを送信する

本番で実際にメールを送信するには、SMTPサーバー、またはMailgun、Postmark、Amazon SESなどのトランザクション用メールプロバイダーが必要です。Laravelは、これらの全てをすぐにサポートします。詳しくは、Mailのドキュメントをご覧ください。

アプリケーションのデプロイについて学び続ける...
