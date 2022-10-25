---
title: Why Jekyll with GitBook
author: Tao He
date: 2019-04-27
category: Jekyll
layout: post
---

03. Chirpsの作成

これで、新しいアプリケーションを作り始める準備ができました。Chirpsと呼ばれる短いメッセージをユーザーが投稿できるようにしましょう。

#モデル、マイグレーション、コントローラ

Chirpsを投稿できるようにするためには、モデル、migrations、controllersを作成する必要があります。それぞれの概念について、もう少し掘り下げてみましょう。

モデルは、データベースのテーブルと対話するための強力で楽しいインターフェイスを提供します。
マイグレーションは、データベースのテーブルを簡単に作成したり変更したりすることを可能にします。また、アプリケーションを実行する場所によって、同じデータベース構造が存在することを保証します。
コントローラは、アプリケーションへのリクエストを処理し、 応答を返す役割を担います。
あなたが作るほとんどの機能は、これらすべてのピースが調和して動作することになるので、artisan make:modelコマンドはそれらを一度に作成することができます。

以下のコマンドで、Chirpsのモデル、マイグレーション、リソースコントローラを作成してみましょう。

php artisan make:model -mrc Chirp

php artisan make:model --helpコマンドを実行すると、利用可能なすべてのオプションを見ることができます。

このコマンドを実行すると、3つのファイルが作成されます。

app/Models/Chirp.php - Eloquentモデル。
database/migrations/<timestamp>_create_chirps_table.php - あなたのデータベーステーブルを作成するデータベースマイグレーションです。
app/Http/Controller/ChirpController.php - リクエストを受信してレスポンスを返すHTTPコントローラです。
#ルーティング
コントローラ用のURLも作成する必要があります。これは、プロジェクトの routes ディレクトリで管理している "routes" を追加することで実現できます。リソースコントローラを使用しているので、1つのRoute::resource()ステートメントを使用して、従来のURL構造に従ってすべてのルートを定義することができます。

まず始めに、2つのルートを有効にします。

indexルートは、フォームとChirpsの一覧を表示します。
storeルートは、新しいChirpsを保存するために使用されます。
また、これらのルートは2つのミドルウェアの後ろに配置します。

authミドルウェアは、ログインしたユーザのみがルートにアクセスできるようにするためのミドルウェアです。
認証用ミドルウェアは、メール認証を有効にする場合に使用します。

routes/web.php
```
<?php
 
use App\Http\Controllers\ChirpController;
use Illuminate\Support\Facades\Route;
 ...
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'verified'])->name('dashboard');
 
Route::resource('chirps', ChirpController::class)
    ->only(['index', 'store'])
    ->middleware(['auth', 'verified']);

require __DIR__.'/auth.php';
```
This will create the following routes:

Verb	URI	Action	Route Name
GET	/chirps	index	chirps.index
POST	/chirps	store	chirps.store

php artisan route:list コマンドを実行すると、アプリケーションのすべてのルートを表示することができます。

新しい ChirpController クラスの index メソッドからテストメッセージを返すことで、ルートとコントローラをテストしてみましょう。


app/Http/Controllers/ChirpController.php
```
<?php
 ...
namespace App\Http\Controllers;
 
use App\Models\Chirp;
use Illuminate\Http\Request;
 
class ChirpController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        //
        return 'Hello, World!';
    }
 ...
}
```

先ほどからログインしたままであれば、http://localhost:8000/chirps、Sailを使っている場合はhttp://localhost/chirps に移動すると、メッセージが表示されるはずです

#ブレード

まだ感動していない？ChirpControllerクラスのindexメソッドを更新して、Bladeビューをレンダリングしてみましょう。


```
app/Http/Controllers/ChirpController.php
<?php
 ...
class ChirpController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        return 'Hello, World!';
        return view('chirps.index');
    }
 ...
}
```


そして、新しいChirpsを作成するためのフォームを持つBladeビューテンプレートを作成することができます。


```
resources/views/chirps/index.blade.php
<x-app-layout>
    <div class="max-w-2xl mx-auto p-4 sm:p-6 lg:p-8">
        <form method="POST" action="{{ route('chirps.store') }}">
            @csrf
            <textarea
                name="message"
                placeholder="{{ __('What\'s on your mind?') }}"
                class="block w-full border-gray-300 focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50 rounded-md shadow-sm"
            >{{ old('message') }}</textarea>
            <x-input-error :messages="$errors->get('message')" class="mt-2" />
            <x-primary-button class="mt-4">{{ __('Chirp') }}</x-primary-button>
        </form>
    </div>
</x-app-layout>
```

これで完了です。ブラウザでページを更新すると、Breezeが提供するデフォルトレイアウトで新しいフォームがレンダリングされます!

Chirpフォーム
スクリーンショットが上記のように表示されない場合は、Tailwind用のVite開発サーバを停止して起動し、先ほど作成した新しいファイルのCSSクラスを検出する必要があるかもしれません。

これ以降、Bladeテンプレートに加えた変更は、npm run dev経由でVite開発サーバーが起動するたびに、ブラウザで自動的にリフレッシュされるようになります。

ナビゲーションメニュー
Breezeが提供するナビゲーションメニューへのリンクを追加してみましょう。

Breezeが提供するnavigation.blade.phpコンポーネントを更新して、デスクトップ画面用のメニュー項目を追加します。


```
resources/views/layouts/navigation.blade.php
<div class="hidden space-x-8 sm:-my-px sm:ml-10 sm:flex">
    <x-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
        {{ __('Dashboard') }}
    </x-nav-link>
    <x-nav-link :href="route('chirps.index')" :active="request()->routeIs('chirps.index')">
        {{ __('Chirps') }}
    </x-nav-link>
</div>
```


また、モバイル画面にも対応。



```
resources/views/layouts/navigation.blade.php
<div class="pt-2 pb-3 space-y-1">
    <x-responsive-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
        {{ __('Dashboard') }}
    </x-responsive-nav-link>
    <x-responsive-nav-link :href="route('chirps.index')" :active="request()->routeIs('chirps.index')">
        {{ __('Chirps') }}
    </x-responsive-nav-link>
</div>
```

Chirp を保存する

先ほど作成した chirps.store ルートにメッセージをポストするようにフォームが設定されました。ChirpController クラスの store メソッドを更新して、データを検証し、新しい Chirp を作成してみましょう。


```
app/Http/Controllers/ChirpController.php
<?php
 ...
class ChirpController extends Controller
{
 ...
    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        //
        $validated = $request->validate([
            'message' => 'required|string|max:255',
        ]);
 
        $request->user()->chirps()->create($validated);
 
        return redirect(route('chirps.index'));
    }
 ...
}
```

Laravelの強力なバリデーション機能を使って、ユーザーがメッセージを提供し、それがこれから作成するデータベースカラムの255文字制限を超えないことを確認することにしています。

そして、chirpsリレーションシップを利用して、ログインしたユーザーに属するレコードを作成しています。このリレーションシップは近日中に定義します。

最後に、ユーザーをchirps.indexルートに送り返すためのリダイレクトレスポンスを返すことができます。

#リレーションシップの作成

前のステップで、$request->user()オブジェクトに対してchirpsメソッドを呼び出したことにお気づきでしょうか。このメソッドをUserモデルに作成して、"has many "のリレーションシップを定義する必要があります。


```
app/Models/User.php
<?php
 ...
class User extends Authenticatable
{
 ...
    public function chirps()
    {
        return $this->hasMany(Chirp::class);
    }
}
```

Laravelには様々なタイプのモデルリレーションシップがあり、Eloquent Relationshipsのドキュメントで詳細を読むことができます。

#大量割り当ての保護

リクエストからモデルへすべてのデータを渡すことは、リスクがあります。ユーザーが自分のプロファイルを編集できるページがあるとします。もしあなたがリクエスト全体をモデルに渡すとしたら、ユーザーは is_admin カラムのような好きなカラムを編集することができます。これは大量割り当ての脆弱性と呼ばれます。

Laravelは、デフォルトで大量代入をブロックすることで、誤ってこのようなことをしないように保護しています。しかし、大量代入は各属性を1つずつ代入する必要がないので、非常に便利です。安全な属性に対して、"fillable "とマークすることで、大量代入を有効にすることができます。

message属性の大量割り当てを有効にするために、Chirpモデルに$fillableプロパティを追加してみましょう。

```
app/Models/Chirp.php
<?php
 ...
class Chirp extends Model
{
 ...
    protected $fillable = [
        'message',
    ];
}
```

Laravelの大量割り当て保護については、ドキュメントで詳しく説明されています。

#マイグレーションの更新

唯一足りないのは、ChirpとそのUserの関係やメッセージそのものを保存するための、データベース内の追加カラムです。先ほど作成したデータベースのマイグレーションを覚えていますか？そのファイルを開いて、追加のカラムを追加するときが来ました。


```
databases/migration/<timestamp>_create_chirps_table.php
<?php
 ...
return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('chirps', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('message');
            $table->timestamps();
        });
    }
 ...
};
```

このマイグレーションを追加してから、データベースの移行をしていないので、今すぐやってしまいましょう。

php artisan migrate


各データベースの移行は一度だけ実行されます。テーブルを追加で変更するには、別のマイグレーションを作成する必要があります。開発中は、展開されていないマイグレーションを更新し、php artisan migrate:fresh コマンドを使用してデータベースをゼロから再構築するとよいでしょう。

#テストする

先ほど作成したフォームを使って、Chirpを送信する準備ができました まだ、既存のChirpを表示していないので、結果を見ることはできません。

メッセージフィールドを空にするか、255文字以上入力すると、バリデーションが実行されるようになります。

#アルティザンティンカー
Laravelアプリケーションで任意のPHPコードを実行できるREPL（Read-eval-print loop）であるArtisan Tinkerについて学ぶには絶好の機会です。

コンソールで、新しいティンカー・セッションを開始します。

php artisan tinker

次に、データベース内のChirpsを表示するために、以下のコードを実行します。

```
Chirp::all();
```

```
=> Illuminate\Database\Eloquent\Collection {#4512
     all: [
       App\Models\Chirp {#4514
         id: 1,
         user_id: 1,
         message: "I'm building Chirper with Laravel!",
         created_at: "2022-08-24 13:37:00",
         updated_at: "2022-08-24 13:37:00",
       },
     ],
   }
```

Tinker を終了するには、exit コマンドを使用するか、Ctrl + c を押してください。

Chirpsの表示を開始するために続ける...