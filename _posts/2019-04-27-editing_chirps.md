---
title: Why Jekyll with GitBook
author: Tao He
date: 2019-04-27
category: Jekyll
layout: post
---

05. 鳴き声の編集

鳥をテーマにした他の人気マイクロブログプラットフォームに欠けている機能、Chirpsを編集する機能を追加してみましょう

#ルーティング

まず最初に、リソースコントローラで chirps.edit と chirps.update ルーティングを有効にするために routes ファイルを更新します。chirps.edit ルートは Chirp を編集するためのフォームを表示し、一方 chirps.update ルートはフォームからデータを受け取り、モデルを更新します。

```
routes/web.php
<?php
 ...
use App\Http\Controllers\ChirpController;
use Illuminate\Support\Facades\Route;
 
/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
|
| Here is where you can register web routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| contains the "web" middleware group. Now create something great!
|
*/
 
Route::get('/', function () {
    return view('welcome');
});
 
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'verified'])->name('dashboard');
 
Route::resource('chirps', ChirpController::class)
    ->only(['index', 'store'])
    ->only(['index', 'store', 'edit', 'update'])
    ->middleware(['auth', 'verified']);
 ...
```

このコントローラのルートテーブルは、次のようになります。


Verb	URI	Action	Route Name
GET	/chirps	index	chirps.index
POST	/chirps	store	chirps.store
GET	/chirps/{chirp}/edit	edit	chirps.edit
PUT/PATCH	/chirps/{chirp}	update	chirps.update

編集ページへのリンク

次に、新しく作成したchirps.editルートをリンクさせましょう。Breezeに付属するx-dropdownコンポーネントを使用し、Chirpの作者にのみ表示するようにします。また、Chirpのcreated_atの日付とupdated_atの日付を比較して、Chirpが編集されたことを表示するようにします。


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
 
        <div class="mt-6 bg-white shadow-sm rounded-lg divide-y">
            @foreach ($chirps as $chirp)
                <div class="p-6 flex space-x-2">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-gray-600 -scale-x-100" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                        <path stroke-linecap="round" stroke-linejoin="round" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
                    </svg>
                    <div class="flex-1">
                        <div class="flex justify-between items-center">
                            <div>
                                <span class="text-gray-800">{{ $chirp->user->name }}</span>
                                <small class="ml-2 text-sm text-gray-600">{{ $chirp->created_at->format('j M Y, g:i a') }}</small>
                                @unless ($chirp->created_at->eq($chirp->updated_at))
                                    <small class="text-sm text-gray-600"> &middot; {{ __('edited') }}</small>
                                @endunless
                            </div>
                            @if ($chirp->user->is(auth()->user()))
                                <x-dropdown>
                                    <x-slot name="trigger">
                                        <button>
                                            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-gray-400" viewBox="0 0 20 20" fill="currentColor">
                                                <path d="M6 10a2 2 0 11-4 0 2 2 0 014 0zM12 10a2 2 0 11-4 0 2 2 0 014 0zM16 12a2 2 0 100-4 2 2 0 000 4z" />
                                            </svg>
                                        </button>
                                    </x-slot>
                                    <x-slot name="content">
                                        <x-dropdown-link :href="route('chirps.edit', $chirp)">
                                            {{ __('Edit') }}
                                        </x-dropdown-link>
                                    </x-slot>
                                </x-dropdown>
                            @endif
                        </div>
                        <p class="mt-4 text-lg text-gray-900">{{ $chirp->message }}</p>
                    </div>
                </div>
            @endforeach
        </div>
    </div>
</x-app-layout>
```

編集フォームの作成

Chirp を編集するためのフォームを持つ Blade ビューを新規に作成しましょう。これは Chirp を作成するためのフォームと似ていますが、chirps.update ルートに投稿し、@method ディレクティブを使用して "PATCH" リクエストを作成していることを指定します。また、フィールドには既存の Chirp メッセージがあらかじめ入力されています。


```
resources/views/chirps/edit.blade.php
<x-app-layout>
    <div class="max-w-2xl mx-auto p-4 sm:p-6 lg:p-8">
        <form method="POST" action="{{ route('chirps.update', $chirp) }}">
            @csrf
            @method('patch')
            <textarea
                name="message"
                class="block w-full border-gray-300 focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50 rounded-md shadow-sm"
            >{{ old('message', $chirp->message) }}</textarea>
            <x-input-error :messages="$errors->get('message')" class="mt-2" />
            <div class="mt-4 space-x-2">
                <x-primary-button>{{ __('Save') }}</x-primary-button>
                <a href="{{ route('chirps.index') }}">{{ __('Cancel') }}</a>
            </div>
        </form>
    </div>
</x-app-layout>
```



コントローラを更新する

ChirpControllerのeditメソッドを更新して、フォームを表示させましょう。Laravelはルートモデルバインディングを使用して、Chirpモデルをデータベースから自動的にロードするので、ビューに直接渡すことができます。

そして、updateメソッドを更新して、リクエストを検証し、データベースを更新します。

Chirpの作者にのみ編集ボタンを表示するとしても、これらのルートにアクセスするユーザが認証されていることを確認する必要があります。


```
app/Http/Controllers/ChirpController.php
<?php
 ...
class ChirpController extends Controller
{
 ...
    /**
     * Show the form for editing the specified resource.
     *
     * @param  \App\Models\Chirp  $chirp
     * @return \Illuminate\Http\Response
     */
    public function edit(Chirp $chirp)
    {
        //
        $this->authorize('update', $chirp);
 
        return view('chirps.edit', [
            'chirp' => $chirp,
        ]);
    }
    /**
     * Update the specified resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Chirp  $chirp
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, Chirp $chirp)
    {
        //
        $this->authorize('update', $chirp);
 
        $validated = $request->validate([
            'message' => 'required|string|max:255',
        ]);
 
        $chirp->update($validated);
 
        return redirect(route('chirps.index'));
    }
 ...
}
```


バリデーションルールがstoreメソッドと重複していることにお気づきでしょうか。LaravelのForm Request Validationを使えば、バリデーションルールを簡単に再利用でき、コントローラーを軽量化できるので、抽出することを検討されてはいかがでしょうか。

#Authorization

デフォルトでは、authorizeメソッドにより、誰でもChirpを更新できるようにすることができます。以下のコマンドでModel Policyを作成することで、更新を許可する人を指定することができます。

php artisan make:policy ChirpPolicy --model=Chirp です。
これで app/Policies/ChirpPolicy.php にポリシークラスが作成され、これを更新することで、Chirp を更新できるのは作者だけであることを指定することができます。


```
app/Policies/ChirpPolicy.php
<?php
 ...
class ChirpPolicy
{
 ...
    /**
     * Determine whether the user can update the model.
     *
     * @param  \App\Models\User  $user
     * @param  \App\Models\Chirp  $chirp
     * @return \Illuminate\Auth\Access\Response|bool
     */
    public function update(User $user, Chirp $chirp)
    {
        //
        return $chirp->user()->is($user);
    }
 ...
}
```


テストする

いよいよテストです。ドロップダウンメニューを使って、いくつかのチャープを編集してみましょう。他のユーザーアカウントを登録すると、チャープを作成した本人しか編集できないことがわかります。


Chirpsの削除を引き続き許可する...
