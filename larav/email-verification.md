# Email Verification

## Scaffolding

menginstall laravel

```shell
laravel new email-verification --auth
```

> note: `--auth` menginstall laravel beserta authentication

## Set up migration

pastikan sudah di directory `email-verification`

```shell
art make:migration add_activation_columns_to_users --table=users
```

lalu di dalam migration `activation_columns` di isi seperti ini

```php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->boolean('active')->default(false);
        $table->string('activation_token')->nullable();
    });
}
```

untuk table drop kolom

```php
public function down()
{
    Schema::table('user', function (Blueprint $table) {
        $table->dropColumn('active', 'activation_token');
    });
}
```

## User Model

buka bagian `App\User.php` dan tambahkan `mass assignmenable`

```php
protected $fillable = [
    'name', 'email', 'password', 'active', 'activation_token'
];
```

sebelum migration kita harus meng setup `.env` terlebih dahulu

## .env & mailtrap

silahkan daftarkan dulu ke [mailtrap](https://mailtrap.io) setelah itu buat sebuah email dummy dan buka bagian SMTP Settings dan copy paste `Username` dan `Password`

```txt
APP_URL=http://email-verification.art

DB_DATABASE=lv-email-verification
DB_USERNAME=root
DB_PASSWORD=root

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=9692e8db221b06
MAIL_PASSWORD=0151460ea5e255
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=kevariable@gmail.com
MAIL_FROM_NAME="${APP_NAME}"
```

## Preventing login for new user

mencegah user setelah daftar akan diredirect ke login dan mendapatkan `session success` menampilkan pesan untuk aktifasi. buka `LoginController` pada `app/Http/Controllers/Auth/`

disana terdapat `use AuthenticatesUsers;` ini adalah trait silahkan klik dan membuka filenya akan diarahkan ke file tsb.

didalamnya terdapat sebuah method

```php
protected function validateLogin(Request $request)
{
    $request->validate([
        $this->username() => 'required|string',
        'password' => 'required|string',
    ]);
}
```

dan jika diperhatikan terdapat sebuah instance `$this->username` yang merujuk ke method `username` di `AuthenticatesUsers`

```php
public function username()
{
    return 'email';
}
```

kita akan melakukan perubahan pada method tsb tapi kita tidak akan merubah method ini di `AuthenticatesUsers`
melainkan kita akan mengoverride pada `LoginController`
langsung saja kita merubah seperti ini

```php
protected function validateLogin(Request $request)
{
    $request->validate([
        // https://laravel.com/docs/7.x/validation#rule-email
        $this->username() => [
            'required', 'string',
            Rule::exists('users')->where(function ($query) {
                $query->where('active', true);
            })
        ],
        'password' => 'required|string'
    ], $this->validateError());
}

/*
 * Get validation error for login
 * @return array
 * */
protected function validateError()
{
    return [
        $this->username() . '.exists' => 'The selected email is invalid or you need to activate your account'
    ];
}
```

> note: pastikan sudah mengimport yang akan kita gunakan

silahkan register dan logout lalu kita login kira2 begini gambarannya

![alt](https://i.imgur.com/kih5Yan.png)

## Preventing login after register

buka `RegisterController` dan kita melihat trait `use RegistersUsers` silahkan menuju file tsb.

di dalam file tsb disana ada method bernama `register` dan didalam register ada sebuah prosses

```php
if ($response = $this->registered($request, $user)) {
    return $response;
}
```

diman ketika sebuah kondisional `$this->registered` terdapat sebuah aksi maka akan bernilai true jika tidak akan dilanjutkan pada method dibawahnya

```php
return $request->wantsJson()
                ? new Response('', 201)
                : redirect($this->redirectPath());
```

dimana ini akan mengredirect kehalaman login nah disini kita akan mencegah hal itu menggunakan method `registered`
langsung saja kita buat di `RegisterController` kita tambahkan method `registered` yang didapat di `RegistersUsers`

```php
/**
* Preventing after registered and redirect to login with session success
*/
protected function registered(Request $request, $user)
{
    $this->guard()->logout();

    return redirect()->route('login')->withSuccess(
        'Registered. Please check your email to activate your account'
    );
}
```

disini kita melihat `$this->guard()` yang membuat session user logout setelah daftar, karena setelah daftar sudah otomatis login oleh fungsi `$this->guard()->login()` pada `register` di `RegistersUsers`

## Alert partials after register

membuat folder `resoureces/views/layouts/partials/` dan didalam `partials` di isi dengan blade `_alert.blade.php`

```php
@if ($message = session('success'))
    <div class="alert alert-success">
        {{ $message }}
    </div>
@endif
```

dan ktia akan menambahkan `include('layouts.partials._alert')` pada `resources/views/layouts/app.blade.php` pada bagian `@yield('content')` di atasnya.

```html
<main class="py-4">

    <div class="container">
        @include('layouts.partials._alert')
    </div>

    @yield('content')
</main>
```

dan kita coba untuk register lalu akan diarahkan ke login page dengan session `success` yang menampilkan alert menggunakan `_alert.blade.php` dimana session ini dibuat pada `RegisterController` tepatnya pada bagian method `registered` ketika me `return`

![alt](https://i.imgur.com/edKjimh.png)

## Event and Listener for sending mail

kita akan membuat sebuah `event`. Sebelum itu kita akan menambahkan ketika register pada bagian `RegisterController` pada method untuk `craete` user dengan menambahkan `active` bernilai `false` dan `activation_token` akan menggunakan helper `Str` lalu dengan static method `random` untuk mengenerate string sebanyak `255` karakter

```php
protected function create(array $data)
{
    return User::create([
        'name' => $data['name'],
        'email' => $data['email'],
        'password' => Hash::make($data['password']),
        'active' => false,
        'activation_token' => Str::random(255)
    ]);
}
```

> note: jangan lupa untuk mengimport `Str`

selanjutnya kita membuat sebuah event di artisan command

```text
art make:event Auth\\UserActivationEmail
```

dan akan membuat directory `app\Event\Auth\UserActivationEmail.php`. disini kita akan membuka file tsb dan menghapus beberapa yang tidak digunakan yaitu `broadcast` atau `websocket` seperti ini

```php
<?php

namespace App\Events\Auth;

use App\User;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserActivationEmail
{
    use Dispatchable, SerializesModels;

    public $user;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }
}
```

dimana sebuah __construct akan menerima parameter dari `RegisterController` di method `registered` dimana kita akan menambahkan sebuah `event`

```php
event(new UserActivationEmail($user));
```

tambahkan `event` ini seebelum `$this->guard()->logout()`

> note jangan lupa untuk mengimport `UserActivationEmail`

setelah itu kita akan membuat sebuah `listener` untuk mengambil data `user` pada event `UserActivationEmail`

```shell
art make:listener SendingActivationEmail --event=Auth\\UserActivationEmail
```

> note: disini kita menambahkan params untuk memberitahu `laravel` bahwa `listener` akan mengambil `event` dari `UserActivationEmail` dan artisan diatas akan mengenerate folder `app\Listener` dan didalamnya `Auth\SendingActivationEmail.php`

sebelum melakukan uji coba `event` & `listener` kita akan mendaftarkan terlebih dahulu di `app/Provider/EventServiceProvider` di `protected $listen`

```php
protected $listen = [
    UserActivationEmail::class => [
        SendingActivationEmail::class,

        // other listener here
    ]
];
```

> note: jangan lupa untuk mengimport `event` dan `listener` tsb.

dan didalam `SendingActivationEmail` terdapat method handle yang sudah berisi `event` dan kita akan mencoba `dump and die / dd()` untuk melihat apakah sebuah `listener` bisa mengambil data dari `event`

```php
public function handle(UserActivationEmail $event)
{
    dd($event);
}
```

sekarang coba `register` dan melihat apakah `listener`nya berhasil mengambil data tsb akan muncul data dari `user`

![alt](https://i.imgur.com/J9eSUQm.png)

## Sending Mail with Activation token

kita akan membuat sebuah template email untuk aktifasi

```shell
art make:mail Auth\\ActivationEmail --markdown=emails.auth.activation
```

ini akan mengenerate sebuah folder `app/Mail` dan didalamnya `Auth/SendingActivationEmail` lalu pada params `--markdown` akan mengenerate di `resources` lalu yaitu `emails.auth.activation` yaitu `resources/emails/auth/activation.blade.php`

lalu didalam `Mail/Auth/SendingActivationEmail` terdapat method `build` dan me`return $this->markdown('emails.auth.activation');` yang akan mengarahkan ke `view` pada markdown yang telah dibuat pada `views/emails/auth/activation.blade.php`

dan kita akan melakukan sedikit perubahan pada template markdown yang sudah ada pada `activation.blade.php`

```markdown
@component('mail::message')
# Introduction

Thanks for sign up, please activate your account

@component('mail::button', ['url' => ''])
Activation
@endcomponent

Thanks,<br>
{{ config('app.name') }}
@endcomponent
```

lalu kita kembali kepada `SendingActivationEmail` di `listener` pada method `handle` sebelumnya

```php
public function handle(UserActivationEmail $event)
{
    dd($event);
}
```

kita akan melakukan perubahan untuk mengecek. Ketika user melakukan register lalu mengecek column `active` jika `false` maka kita akan melakukan sending email pada email tsb. Jika `active` bernilai `true` kita akan me `return;` dan melanjutkan proses `log out`.

```php
public function handle(UserActivationEmail $event)
{
    if ($event->user->active) // karena $event menghasilkan object user lalu mengambil active
        return;
}
```

dan untuk menghandle ketika `active` bernilai false ktia tambahkan dibawahnya conditional seperti ini

```php
public function handle(UserActivationEmail $event)
{
    if ($event->user->active) // karena $event menghasilkan object user lalu mengambil active
        return;

    return Mail::to($event->user->email)->send(new ActivationEmail($event->user));
}
```

> penjelasan: pertama jangan lupa untuk mengimport `Mail` lalu menggunakan method static `to` untuk mengirim ke email tujuan `$event->user->email` dan yang kita kirim berupa markdown yang akan dihandle `ActivationEmail` yang kita generate sebelumnya menggunakan `artisan` console pada folder `Mail/Auth/ActivationEmail.php`

karena kita menerima parameter dari instance object `new ActivationEmail($event->user)` yang berupa data user maka kita perlu untuk mengatur `__cosntruct` seperti ini

```php
public $user;

public function __construct(User $user)
{
    $this->user = $user;
}
```

> jgn lupa di import dulu `User`nya yah.

mungkin saja kita perlu merubah markdown pada `views` di `activation.blade.php` menambahkan `{{ $user->name }}` setelah kata `Thanks,` dan kita bisa mengetest untuk melakukan sebuah event Mail ketika `register` dan kira2 setelah `register` akan menghasilkan seperti ini.

![alt](https://i.imgur.com/foeEdDD.png)

## Handling `Activation Button`

selanjutnya kita akan mencari cara bagaimana menghandle ketika user akan mengklik button activation. kita akan membuat sebuah controller

```php
art make:controller Auth\\ActivationController
```

dan didalam controller tersebut kita buat method
`__invoke` ini berguna untuk membuat kode lebih singkat karena kita hanya perlu menggunakan single function dalam controller

```php
public function __invoke(Request $request)
{
    dd($request);
}
```

dan kita defenisikan di bagian route dibawah `Auth::routes();`

```php
Route::get('auth/activate', 'Auth\ActivationController')->name('auth.activate');
```

dan kita kembali file markdown `activation.blade.php` dengan menambahkan helper `route` pada `component mail::button` kita merubah seperti ini

```php
@component(
    'mail::button',
    [
        'url' => route('auth.activate', [
            'email' => $user->email,
            'token' => $user->activation_token
        ])
    ]
)
Activation
@endcomponent
```

dimana data `email` dan `token` kita dapat dari `ActivationEmail` pada `__construct`

jika kita coba kembali melakukan `register` dan kita tangkap request dari `activation.blade.php` pada route `auth.activate` dengan 2 parameter `email` dan `token` karena kita menggunakan `get` maka akan muncul sederatan kode panjang pada `url` dan sebuah `request` dihalaman karena dalam method `__invoke` kita membuat `dd($request)`.
lalu kita akan mencoba membuat sebuah activation dengan melakukan perubahan pada method `__invoke`

```php
public function __invoke(Request $request)
{
    $user = User::whereEmail($request->email)
        ->whereActivationToken($request->token)
        ->firstOrFail();

    $user->update([
        'active' => true, // supaya bisa login
        'activation_token' => null
    ]);

    Auth::loginUsingId($user->id);

    return redirect()->route('home')->withSuccess(
        'Activated!, You\'are now sign in'
    );
}
```

nah disini kita melakukan pencarian data menggunakan `where` dan jika ketemu akan dilanjutkan ke method `update` dgn mengisi `actice` jadi `true` dan tokennya dibikin `null` karena sudah diktifasi, lalu kita membuat user tsb login menggunakan `Auth::loginUsingId` lalu kita redirect kehalaman `home` page dengan session success dengan pesannya. sekarang kita sudah login

![alt](https://i.imgur.com/KNifLq5.png)

## Resend Activation

Disini kita akan membuat sebuah `controller` dan `request` untuk menangani resend activation.

```shell
art make:controller ResendActivationController
```

dan

```shell
art make:request ResendActivationRequest
```

pertama kita akan membuka controller yang sudah dibuat. dan mengisinya seperti ini.

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Events\Auth\UserActivationEmail;
use App\Http\Controllers\Controller;
use App\Http\Requests\ResendActivationRequest;
use App\User;

class ResendActivationController extends Controller
{
    /*
     *  membuat halaman view untuk resend activation
     * */
    public function index()
    {
        return view('auth.activation.resend');
    }

    public function resendActivation(ResendActivationRequest $request)
    {
        // mencari email berdasarkan request yang diterima
        $user = User::whereEmail($request->email)->first();

        // resend dengan event yang sebelumnya sudah dibuat
        event(new UserActivationEmail($user));

        return redirect()->route('login')->withSuccess(
            'Resend success, check your email.'
        );
    }
}
```

disini terdapat 2 method yaitu `index` dan `resendActivation`. `index` untuk menampilkan halaman resend dan `resendActivation` berguna untuk menangani request dari `index` dengan method `POST`

lalu kita akan membuat validasi request yang sudah dibuat di `ResendActivationRequest`

```php
public function authorize()
{
    return true;
}
```

kita merubah `authorize` menjadi true. agar bisa memvalidasi. dan selanjutnya mengisi rule pada method `rules`.

```php
public function rules()
{
    return [
        'email' => 'required|string|exists:users,email'
    ];
}
```

untuk mengecek apakah email yang di`request` ada di dalam table `users` di column `email`.

dan kita akan menghandle pesan error jika tidak temukan. langsung saja tambahkan method `messages`

<https://laravel.com/docs/7.x/validation#customizing-the-error-messages>

```php
public function messages()
{
    return [
        'email.exists' => 'This email not exists. Please check your email'
    ];
}
```

tentu saja sebelumnya kita akan membuat route di `web.php` terlebih dahulu.

```php
Route::get('auth/activate/resend', 'ResendActivationController@index')->name('auth.activate.resend');
Route::post('auth/activate/resend', 'ResendActivationController@resendActivation')->name('auth.activate.resened');
```

> note: letakan di bawah `Auth::routes()`

setelah itu tentu saja kita akan membuat blade engine untuk activation resend yang sudah dibuat di method `index`.

```shell
mkdir -p resources/views/auth/activation/ \
; touch resources/views/auth/activation/resend.blade.php
```

disini saya membuat folder bernama `activation` dan didalamnya terdapat file blade `resend`.

kita buat dengan meng`copy` `paste` halaman `login.blade.php` dan merubahnya dengan yang diperlukan saja.

```html
@extends('layouts.app')

@section('content')
    ......... diatas di sembunyikan agar lebih ringkas
                    <form method="POST" action="{{ route('auth.activate.resend') }}">
                        @csrf

                        <div class="form-group row">
                            <label for="email" class="col-md-4 col-form-label text-md-right">{{ __('E-Mail Address') }}</label>

                            <div class="col-md-6">
                                <input id="email" type="email" class="form-control @error('email') is-invalid @enderror" name="email" value="{{ old('email') }}" required autocomplete="email" autofocus>

                                @error('email')
                                    <span class="invalid-feedback" role="alert">
                                        <strong>{{ $message }}</strong>
                                    </span>
                                @enderror
                            </div>
                        </div>

                        <div class="form-group row mb-0">
                            <div class="col-md-8 offset-md-4">
                                <button type="submit" class="btn btn-primary">
                                    {{ __('Resend?') }}
                                </button>
                            </div>
                        </div>
                    </form>
.................dibawah disembunyikan agar ringkas
@endsection
```

dan di dalam `form` yaitu attribute `action` diarahkan ke route `auth.activate.resend` agar diterima request pada mehtod `POST`

dan didalamnya kita hanya butuh `form-group` dari `email` dan `button` login yang dirubah namanya menjadi `Resend`

tentu saja kita akan merubah terlebih dahulu halaman `login.blade.php` dengan menambahkan `button` `Resend Activation?` dibawah `Forgor Your Password?`

```html
@if (Route::has('auth.activate.resend'))
    <a class="btn btn-link" href="{{ route('auth.activate.resend') }}">
        {{ __('Resend Activation?') }}
    </a>
@endif
```

kita mengecek apakah ada terdapat `route` dengan `auth.activate.resend` jika ada akan titampilkan dan juga kita mengarahkan kehalaman `auth.activate.resend` pada controller `ResendActivationController` pada method `index` ke halaman blade `activate.blade.php`

langsung saja dicoba isi emailnya dan kita akan mendapatkan email activation jika email ada didalam database lalu diarahkan ke halaman login dengan session success beserta pesannya.

jika tidak ada didalam database akan menampilkan pesan yang di handle oleh `ResendActivationRequest` pada method `messages`

## Preventing resend activation if account already activation

mudah saja. kita kembali ke controller `ResendActivationController` dan pada method `resendActivation` kita akan melakukan perubahan untuk mengecek apakah email yang ingin diresend activation sudah diaktifasi sebelumnya.

```php
public function resendActivation(ResendActivationRequest $request)
{
    $user = User::whereEmail($request->email)->first();

    // resend with event
    event(new UserActivationEmail($user));

    return redirect()->route('login')->withSuccess(
        'Resend success, check your email.'
    );
}
```

lalu kita ubah seperti ini

```php
public function resendActivation(ResendActivationRequest $request)
{
    $user = User::whereEmail($request->email)->first();

    if ($user->activation_token === null)
        return redirect()->route('auth.activate.resend')->withFailed('Your account already activation!');

    // resend with event
    event(new UserActivationEmail($user));

    return redirect()->route('login')->withSuccess(
        'Resend success, check your email.'
    );
}
```

lalu pada bagian blade `_alert` karena kita akan melakukan session dengan failed kita harus menambahkan sedikit pada bagian blade `_alert`

```html
@if ($message = session('success'))
    <div class="alert alert-success">
        {{ $message }}
    </div>
@endif
```

lalu kita tambahkan

```html
@if ($message = session('success'))
    <div class="alert alert-success">
        {{ $message }}
    </div>
@endif

@if ($message = session('failed'))
    <div class="alert alert-danger">
        {{ $message }}
    </div>
@endif
```

silahkan coba dengan user yang sudah diaktifasi dan meresend ulang.

![alt](https://i.imgur.com/txVV3sy.png)

## Preventing login through reset password

defaultnya pada bagian login page terdapat reset password yang sudah jadi/ bawaan laravel jika di klik lalu akan tampil ke halaman reset link password jika email ada di database akan dikirim jika tidak akan tampil feedbacknya. nah kita dapat email pada mailtrap yang akan mengarahkan ke halaman reset password. misalkan saja saya dapet url `http://email-verification.art/password/reset/a6a6e5dff70ecd6933b68ba58d47b775ca01596dddf443436f0a7b08d3c0827a?email=example%40mail.com` diarahkan kehalaman tsb. nah disini kita akan mencegah ketika reset password apakah user tsb sudah melakukan activation sebelumnya? jika tidak kita akan mengredirect kehalaman login dengan pesannya jika sudah activation maka akan langsung sign in pada halaman `home` page. lansung saja ke studi kasus.

kita akan membuka file `ResetPasswordController` kita akan melihat sedikit method karena fungsi2nya / methodnya berada di `trait` pada `use ResetsPasswords;` jika kita membuka file tsb pada `Illuminate\Foundation\Auth\ResetsPasswords` atau menglik tahan dan akan lgsg terbuka. disana banyak terdapat banyak method seperti `showResetForm` yaitu untuk menampilkan halaman reset password dan ada juga `reset` nah disini kita akan mengoprek method tsb kita akan melihat didalamnya terdapat sebuah `validasi` lalu ada `$response` dan didalamnya terdapat `$this->resetPassword($user, $password);` nah method pada `resetPassowrd` ini akan kita gunakan. dan juga terdapat `sendResetResponse` ini juga akan kita gunakan karena didalam method `reset` pada bagian `return` akan megredirect menggunakan `$this->sendResetResponse($request, $response)`. lsg saja kita bikin di controller `ResetPasswordController` pada bagian bawah. tapi sebelum itu defaultnya seperti ini pada trait.

```php
protected function resetPassword($user, $password)
{
    $this->setUserPassword($user, $password);

    $user->setRememberToken(Str::random(60));

    $user->save();

    event(new PasswordReset($user));

    $this->guard()->login($user);
}
```

defaultnya seperti ini didalam `trait` lalu kita akan merubah sedikit logic menjadi seperti ini

```php
protected function resetPassword($user, $password)
{
    $this->setUserPassword($user, $password);

    $user->setRememberToken(Str::random(60));

    $user->save();

    // tambahan jika true maka akan diloginkan, anjay login
    if ($user->active)
        $this->guard()->login($user);
}
```

> note: kita merubah ini jgn di `trait` tapi kita tambahkan pada controller `ResetPasswordController` bawa bagian bawah untuk mengoverride

disini kita cek apa column active pada user bernilai 1 / true jika iya akan dilanjutkan untuk login jika tidak kita akan membuat dibawahnya method `$this->sendResetResponse($request, $response)`

```php
protected function sendResetResponse(Request $request, $response)
{
    if ($request->wantsJson()) {
        return new JsonResponse(['message' => trans($response)], 200);
    }

    return redirect()->route('login')->withSuccess(
        'Your password has been reset sucessfully'
    );
}
```

lalu kita ubah bagian `redirect()` seperti biasa yang akan mengarahkan ke `login` page dan mengirim session success beserta pesan password sudah dirubah.

## conclusion

kita belajar banyak hal dalam melakukan email verification. dengan memanfaatkan fitur bawaan laravel kita bisa mempermudah melakukan hal tsb. kita juga memanfaatkan mailtrap sebagai provider untuk development.

## Contact

- email `->` kevariable@gmail.com
- telegram `->` https://t.me/kevariable
- github `->` https://github.com/kevariable
- twitter `->` https://twitter.com/kevariable
