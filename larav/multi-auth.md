# Multiple Authenticated

> note: hal pertama yang harus diingat adalah = import jangan lupa apa yang sudah kita gunakan. karena disini saya menggunakan extension intelephense dan php resolver yang akan mengenerate otomatis hal tsb.

## Model & Migration

```
art make:model Admin -m
```

> note: mengenerate model dan migration sesuai `naming convention`

- admins table

  ```php
  <?php

  use Illuminate\Database\Migrations\Migration;
  use Illuminate\Database\Schema\Blueprint;
  use Illuminate\Support\Facades\Schema;

  class CreateAdminsTable extends Migration
  {
      /**
      * Run the migrations.
      *
      * @return void
      */
      public function up()
      {
          Schema::create('admins', function (Blueprint $table) {
              $table->id();
              $table->string('name');
              $table->string('email')->unique();
              $table->timestamp('email_verified_at')->nullable();
              $table->string('password');
              $table->rememberToken();
              $table->timestamps();
          });
      }

      /**
      * Reverse the migrations.
      *
      * @return void
      */
      public function down()
      {
          Schema::dropIfExists('admins');
      }
  }
  ```

  kita samakan dengan model user

- Model

  ```php
  <?php

  namespace App;

  use Illuminate\Foundation\Auth\User as Authenticatable;
  use Illuminate\Notifications\Notifiable;

  class Admin extends Authenticatable
  {
      use Notifiable;

      /**
      * The attributes that are mass assignable.
      *
      * @var array
      */
      protected $fillable = [
          'name', 'email', 'password',
      ];

      /**
      * The attributes that should be hidden for arrays.
      *
      * @var array
      */
      protected $hidden = [
          'password', 'remember_token',
      ];

      /**
      * The attributes that should be cast to native types.
      *
      * @var array
      */
      protected $casts = [
          'email_verified_at' => 'datetime',
      ];
  }
  ```

## Guard

kita akan membuka file `auth.php` berada di `config`. Kita akan menambahkan `admin` sebagai guard dimana pada `auth.php` guard defaultnya adalah `web` yaitu punya `user`.

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'token',
        'provider' => 'users',
        'hash' => false,
    ],

    'admin' => [
        'driver' => 'session',
        'provider' => 'admins'
    ],

    // if need api just create here. like 'api-admin'
],
```

kita menambahkan admin jika perlu menggunakan `api` samakan saja seperti `api` diatas `admin`.

selanjutnya kita perlu membuat provider `admins` yang akan didefenisikan dibawahnya

```php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\User::class,
    ],

    'admins' => [
        'driver' => 'eloquent',
        'model' => App\Admin::class
    ]

    // 'users' => [
    //     'driver' => 'database',
    //     'table' => 'users',
    // ],
],
```

disini kita menambahkan admin dan driver menggunakan `eluquent` jika tidak ingin menggunakan eloquent bisa menggunakan `database` sebagai gantinya.

lalu kita juga akan menambahkan `Resetting Passwords`

```php
'passwords' => [
    'users' => [
        'provider' => 'users',
        'table' => 'password_resets',
        'expire' => 60,
        'throttle' => 60,
    ],

    'admins' => [
        'provider' => 'admins',
        'table' => 'password_resets',
        'expire' => 15,
        'throttle' => 15
    ]
],
```

disini kita akan menggunakan table yang sama seperti user terlebih dahulu dan `expire` 15 menit karena alasan security.

## Controller

selanjutnya kita akan membuat `admin` controller.

```
art make:controller AdminController
```

setelah itu kita samakan isinya dengan `HomeController` namun kita akan merubah beberapa diantaranya bagian `__construct` kita akan menambahkan guard `admin` pada middleware supaya `user` atau heker tidak dapat masuk sebelum login. lalu kita juga menambahkan view pada method `index` untuk halaman dashboard admin

```php
<?php

namespace App\Http\Controllers;

class AdminController extends Controller
{
    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('auth:admin'); // guard admin
    }

    /**
     * Show the application dashboard.
     *
     * @return \Illuminate\Contracts\Support\Renderable
     */
    public function index()
    {
        return view('admin');
    }
}
```

lalu kita akan membuat halaman dashboard admin terlebih dahulu.

kita akan copy paste punya `home`

```html
@extends('layouts.app') @section('content')
<div class="container">
  <div class="row justify-content-center">
    <div class="col-md-8">
      <div class="card">
        <div class="card-header">{{ __('Admin Dashboard') }}</div>

        <div class="card-body">
          @if (session('status'))
          <div class="alert alert-success" role="alert">
            {{ session('status') }}
          </div>
          @endif {{ __('You are logged in!') }}
        </div>
      </div>
    </div>
  </div>
</div>
@endsection
```

lalu kita daftarkan route `admin` pad `web.php`

```php
Route::get('admin', 'AdminController@index')->name('admin.home');
```

namun akan lebih baik kita akan grouping dengan prefix agar nanti route login dan lainya lebih terstructure.

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
})
```

sekarang kalau kita lihat di artisan route list

```
multiple-auth on 🚧 mast [📝] via ⬢ v14.5.0 via 🐘 v7.4.7 took 9s
❯ art route:list -c
+----------+------------------------+------------------------------------------------------------------------+
| Method   | URI                    | Action                                                                 |
+----------+------------------------+------------------------------------------------------------------------+
| GET|HEAD | /                      | Closure                                                                |
| GET|HEAD | admin                  | App\Http\Controllers\AdminController@index                             |
| GET|HEAD | api/user               | Closure                                                                |
| GET|HEAD | home                   | App\Http\Controllers\HomeController@index                              |
| GET|HEAD | login                  | App\Http\Controllers\Auth\LoginController@showLoginForm                |
| POST     | login                  | App\Http\Controllers\Auth\LoginController@login                        |
| POST     | logout                 | App\Http\Controllers\Auth\LoginController@logout                       |
| GET|HEAD | password/confirm       | App\Http\Controllers\Auth\ConfirmPasswordController@showConfirmForm    |
| POST     | password/confirm       | App\Http\Controllers\Auth\ConfirmPasswordController@confirm            |
| POST     | password/email         | App\Http\Controllers\Auth\ForgotPasswordController@sendResetLinkEmail  |
| GET|HEAD | password/reset         | App\Http\Controllers\Auth\ForgotPasswordController@showLinkRequestForm |
| POST     | password/reset         | App\Http\Controllers\Auth\ResetPasswordController@reset                |
| GET|HEAD | password/reset/{token} | App\Http\Controllers\Auth\ResetPasswordController@showResetForm        |
| GET|HEAD | register               | App\Http\Controllers\Auth\RegisterController@showRegistrationForm      |
| POST     | register               | App\Http\Controllers\Auth\RegisterController@register                  |
+----------+------------------------+------------------------------------------------------------------------+
```

terdapat URI `admin` yang sudah terdaftar pada route. jika kita akses halaman tersebut tidak akan bisa karena sudah kita daftarkan guard pada middleware yang bisa mengakses halaman dashboard admin hanyalah yang sudah login. jika teman2 mengapus guardnya maka akan masuk tanda security apapun.

nah sebelumnya kita sudah membuat migration admin adabaiknya kita buat seeder terlebih dahulu agar memudahkan mendevelop aplikasi.

## Seeder

```
art make:seed DummyTableSeeder
```

lalu kita buka file tsb didalam `database/seeds` lalu kita tambahkan data user dan admin di `run` method

```php
User::create([
    'name' => 'Filasi',
    'email' => 'user@mail.com',
    'password' => Hash::make('laravel7')
]);

Admin::create([
    'name' => 'Admin',
    'email' => 'admin@mail.com',
    'password' => Hash::make('laravel7')
]);
```

lalu kita panggil class seeder yang dibuat ke file `DatabaseSeeder.php` pada method `run` juga

```php
 // $this->call(UserSeeder::class);
    $this->call(DummyTableSeeder::class);
```

lalu jalankan seedernya

```
art db:seed
```

dan pada database sudah terdapat data admin dan user sekarang kita bisa login kehalaman user tapi tidak akan bisa login admin menggunakan login user.

## Login Controller & Flow

kita akan membuat folder di `Controllers` bernama `AdminAuth` dan didalamnya dibuat controller bernama `LoginController`.

```php
<?php

namespace App\Controllers\AdminAuth;

use App\Http\Controllers\Controller;
use App\Http\Requests\AdminAuth\LoginRequest;
use Illuminate\Support\Facades\Auth;

class LoginController extends Controller
{
    public function __construct()
    {
        $this->middleware('guest:admin')->except('logout');
    }

    public function showLoginForm()
    {
        return view('admin-auth.login');
    }

    public function login(LoginRequest $request)
    {
        $credentials = [
            'email' => $request->email,
            'password' => $request->password
        ];

        if (Auth::guard('admin')->attempt($credential))
            return redirect()->route('admin.login');

        return redirect()->back()->withInput($request->only('email'));
    }
}
```

disini kita mempunyai 3 method yaitu `__construct` untuk middleware dimana paramsnya `guest:admin` dan itu berfungsi jika admin misalnya sudah login maka tidak akan bisa mengakses login lagi lalu di ikuti chaining methodnya yaitu `except` kecuali `logout` yang masih dapat dipergunakan.

lalu method `showLoginForm` untuk menampilkan halaman login dimana kita belum membuatnya.

lalu method `login` yang menerima `response` dari method `showLoginForm` disini parameternya adalah request yang belum kita buat. kita akan menangkap 2 request yaitu `email` dan `password` yang kita bungkus kedalam array karena akan dipergunakan untuk melakukan percobaan login menggunakan `Auth`. setelah itu terdapat kondisional yg tadi disebutkan menggunakan Auth lalu kita mengecek `guard`nya apakah guardnya `admin` jika iya maka akan dilakukan percobaan menggunakan `attempt` beserta kridensialnya.

nah langkah selanjutnya mungkin kita akan mendaftarkan terlebih dahulu controller yang sudah dibuat pada `web.php` dimana kita akan menambahkannya pada `prefix` yang sudah dibuat didalam `group`

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
    Route::get('login', 'AdminAuth\LoginController@showLoginForm')->name('admin.login');
    Route::post('login', 'AdminAuth\LoginController@login');
})
```

selanjutnya kita akan membuat `LoginRequest`

```
art make:request AdminAuth\\LoginRequest
```

lalu didalamnya method `authorize` dibuat `true` dan validasinya sebagai berikut.

```php
public function rules()
{
    return [
        'email' => 'email|required|exists:admins,email',
        'password' => 'string|min:6|required'
    ];
}
```

selanjutnya kita membuat halaman view `login` didalam `views` dengan nama `admin-auth` dan bladenya yaitu `login` dan kita akan meniru `views/auth/login`

```html
@extends('layouts.app') @section('content')
<div class="container">
  <div class="row justify-content-center">
    <div class="col-md-8">
      <div class="card">
        <div class="card-header">{{ __('Admin Login') }}</div>

        <div class="card-body">
          <form method="POST" action="{{ route('admin.login') }}">
            @csrf .................... disembunyikan agar ringkas
          </form>
        </div>
      </div>
    </div>
  </div>
</div>
```

nah kita hanya merubah `action` pada `form` menggunakan `route` yang sudah dibuat menggunakan method `post` dan card-header dengan tambahan `Admin Login` saja.

nah kita akan merubah sedikit flow `UX` ketika membuka login admin dimana jika diurl untuk mengakses halaman login admin dibutuhkan `url.com/admin/login` sebagai gantinya kita akan persingkat menjadi `url.com/admin` menggunakan `ExceptionHandler`. jika dibuka `app/Exception/Handler.php` maka akan terdapat beberapa method dan jika diperhatikan `Exception` meng`extends` `ExceptionHandler` jika dibuka lalu kita cari method `unauthenticated` kita akan menggunakan method itu lalu kita `override` pada `Handler.php` tambahkan dibawahnya.

```php
protected function unauthenticated($request, AuthenticationException $exception)
{
    if ($request->wantsJson())
            return response()->json(['message' => $exception->getMessage()], 401);

    dd($exception->guards());
}
```

mari kita test terlebih dahulu halaman `url.com/admin`. oh ya conditional if itu untuk menampilkan pesan error ketika user meresponse header sebagai `json/application` maka kita akan tampilkan error.

maka jika dibuka url tsb akan menampilkan array didalamnya.

```php
array:1 [▼
  0 => "admin"
]
```

seperti ini. Kita akan mengambil valuenya dengan menggunakan helper laravel `Arr::get(array, index)`.

```php
protected function unauthenticated($request, AuthenticationException $exception)
{
    if ($request->wantsJson())
            return response()->json(['message' => $exception->getMessage()], 401);

    $guard = Arr::get($exception->guards, 0);
}
```

nah `0` itu adalah indexnya untuk mendapatkan value `admin`

selanjutnya kita aka nmenggunakan `switch` untuk mengatur flow ketika url `url/admin/` dibuka.

```php
protected function unauthenticated($request, AuthenticationException $exception)
{
    if ($request->wantsJson())
            return response()->json(['message' => $exception->getMessage()], 401);

    $guard = Arr::get($exception->guards, 0);

    switch ($guard) {
        case 'admin':
            $login = 'admin.login';
            break;
        default:
            $login = 'admin';
            break;
    }

    return redirect()->guest(route($login));
}
```

nah `route('admin.login')` akan me redirect ke /admin/login. selanjutnya kita coba login dengan data `dummy` yang sudah ada maka akan berhasil kehalaman `admin` meggunakan `AdminAuth/LoginController` pada method `login` jika gagal maka akan diteruskan ke return selanjutnya

```php
return redirect()->back()->withInput($request->only('email'));
```

disini akan menredirect kembali kehalaman login dengan inputan yang sebelum diketik yaitu email lalu didalamnya yang boleh melakukan feedback data tetap ada di login hanya `email` dengan kata lain `password` menjadi kosong. sejauh ini kita berhasil melakukan login. tetapi apa yang terjadi jika kita mengakses halaman `login` sementara kita sudah `login`? nah disitu kita akan melakukan sesuatu agar tidak dapat kembali mengakses halaman login, caranya kita akan merubah sesuatu pada `app/Http/Middleware/RedirectIfAuthenticated.php` pada method `handle`.

```php
public function handle ($request, Closure $next, $guard = null)
{
    switch ($guard) {
        case 'admin':
            if (Auth::guard($guard)->check())
                return redirect()->route('admin.home');
            break;
        default:
            if (Auth::guard($guard)->check())
                return redirect()->route(RouteServiceProvider::HOME);
            break;
    }

    return $next($request)
}
```

disini kita mencheck apakah guard tersebut sudah login atau belom, kalau sudah tidak akan bisa ke halaman login dengan guard yang sama. jika belum login maka akan diteruskan ke `$next($request)` yang diminta.

## Components for logout

sebelum kita membuat fitur logout pada multiple auth kita akan membuat sebuah component yang akan mengecek admin dan user apakah sedang login atau logout, langsung saja pada artisan.

```
art make:components has_active
```

maka akan mengenerate folder `components` pada `views` lalu didalamnya terdapat blade `has_active.blade.php` kita akan menggunakan blade tsb.

```html
@if (Auth::guard('web')->check())
<div class="text-success">You're Logged In as a <strong>User</strong></div>
@else
<div class="text-danger">You're Logged Out as a <strong>User</strong></div>
@endif @if (Auth::guard('admin')->check())
<div class="text-success">You're Logged In as a <strong>Admin</strong></div>
@else
<div class="text-danger">You're Logged Out as a <strong>Admin</strong></div>
@endif
```

baik tentu saja kita akan mengimport partial component yang sudah dibuat ke halaman admin, home dan halaman utama kita hanya perlu menambahkan `@component('components.has_active')@endcomponent`

```html
..........disembunyikan
<div class="card-body">
  @if (session('status'))
  <div class="alert alert-success" role="alert">
    {{ session('status') }}
  </div>
  @endif @component('components.has_active') @endcomponent
</div>
...........disembunyikan
```

saya yakin kamu bisa menyesuaikannya.

## Logout without remove all login session

disini kita akan mencoba untuk membuat fitur logout dimana jika kita menglogout akun user tidak akan menglogout akun admin.

pertama kita akan mengoverride method `logout` pada `LoginController` pada user. dimana method ini milik trait `use AuthenticatesUsers;`

```php
public function logout()
{
    $this->guard()->logout();

    redirect('/');
}
```

method `$this->guard()` mengarahkan ke`trait` jika tidak menggunakan trait bawaan bisa menggunakan `Auth::guard('web')->logout();`

lalu kita coba untuk mengloginkan `admin` dan `user`.

langsung saja kita coba untuk menglogout pada user menggunakan logout yang berada di panel atau menggunakan url `url.com/logout`.

maka yang berada diadmin tidak akan keluar.

selanjutnya kita akan membuatkan untuk `admin` berada di method `LoginController` kita akan menambahkan method `logout` juga.

```php
public function logout()
{
    Auth::guard('admin')->logout();

    return redirect('/');
}
```

lalu kita daftarkan ke `route` pada `prefix` admin.

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
    Route::get('login', 'AdminAuth\LoginController@showLoginForm')->name('admin.login');
    Route::post('login', 'AdminAuth\LoginController@login');
    Route::get('logout', 'AdminAuth\LoginController@logout'); // tambahan logout
});
```

langsung saja kita coba untuk mengakses halaman `url.art/admin/logout`, Maka akan logout tanpa mendestroy semua session yang login.

## Forgot Password

Kita akan membuat sebuah forgot password untuk admin. dimana kita akan mengcopy terlebih dahulu controller milik user `Auth/ForgotPasswordController.php` pada `AdminAuth`

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Foundation\Auth\SendsPasswordResetEmails;

class ForgotPasswordController extends Controller
{
    /*
    |--------------------------------------------------------------------------
    | Password Reset Controller
    |--------------------------------------------------------------------------
    |
    | This controller is responsible for handling password reset emails and
    | includes a trait which assists in sending these notifications from
    | your application to your users. Feel free to explore this trait.
    |
    */

    use SendsPasswordResetEmails;
}
```

terlihat by default seperti ini tidak terdapat method apapun selain `trait`. Jika kita buka trait tsb maka akan banyak method2 untuk menangani `forgot password` kita membutuh kan 2 method yaitu `showLinkRequestForm` digunakan untuk menampilkan halaman forgot password dan `ResetLinkEmail` menerima `response` dari `forgot password`. Pertama kita buat construct terlebih dahulu untuk membuat `guest` yang berguna menangani jika sudah login maka tidak dapat login kembali.

```
public function __construct()
{
    $this->middleware('guest:admin');
}
```

lalu dibawahnya method yang kita copy.

```php
/**
* Display the form to request a password reset link.
*
* @return \Illuminate\View\View
*/
public function showLinkRequestForm()
{
    return view('admin-auth.passwords.email');
}

/**
* Send a reset link to the given user.
*
* @param  \Illuminate\Http\Request  $request
* @return \Illuminate\Http\RedirectResponse|\Illuminate\Http\JsonResponse
*/
public function sendResetLinkEmail(Request $request)
{
    $this->validateEmail($request);

    $response = $this->broker()->sendResetLink(
        $this->credentials($request)
    );

    return $response == Password::RESET_LINK_SENT
        ? $this->sendResetLinkResponse($request, $response)
        : $this->sendResetLinkFailedResponse($request, $response);
}
```

baik. kita akan membahas `sendResetLinkEmail`, setelah mendapatkan response dari halaman forgot password kita mendapatkan request berupa `email` lalu kita akan memvalidasi email tsb dgn method `validateEmail`. Sebenarnya method tsb sudah ada di trait namun kita akan mengoverride supa merubah sedikit dengan menambahkan method tsb dibawah `sendResetLinkEmail`.

```php
/**
* Validate the email for the given request.
*
* @param  \Illuminate\Http\Request  $request
* @return void
*/
public function validateEmail(Request $request)
{
    return $request->validate([
        'email' => 'required|email|exists:admins,email'
    ], $this->validateError());
}
```

lalu kita menghandle error dengan method `$this->validateError()` yang akan kita buat dibawahnya.

```php
/**
* Handlong Validate Error the email for the given request.
*
* @param  \Illuminate\Http\Request  $request
* @return void
*/
public function validateError()
{
    return [
        'email' => 'Your email doesn\'t exists.'
    ];
}
```

selanjutnya pada `sendResetLinkPassword` setelah validasi terdapat variable $response dan didalamnya terdapat sebuah `broker` dan `broker` ini merupakan `Resetting Password` yang sudah di setting pada `auth.php` pada bawah sekali yang mana kita sudah mensettingnya. Nama brokernya adalah `admins` yang menggunakkan provider `admins` juga. Karena menggunakan `$this->broker` kita akan membuat function lagi dibawah `validateError`.

```php
/**
* Get the broker to be used during password reset.
*
* @return \Illuminate\Contracts\Auth\PasswordBroker
*/
public function broker()
{
    return Password::broker('admins');
}
```

lalu ada method chaining dari broker berupa `sendResetLink` dimana method ini juga sudah ada di `trait` berupa method untuk mengirim `notification` yang akan kita buat nanti lalu parameternya berupa `credentials` dari request yang didapat, karena method ini menggunakan this dan terdapat pada `trait` kita akan mengoverridenya kita buat dibawah sekali.

```php
/**
* Get the needed authentication credentials from the request.
*
* @param  \Illuminate\Http\Request  $request
* @return array
*/
public function credentials(Request $request)
{
    return $request->only('email');
}
```

oke sejauh ini kita sudah menambahkan controllernya lalu sekarang kita akan menambahkan `routes`nya.

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
    Route::get('login', 'AdminAuth\LoginController@showLoginForm')->name('admin.login');
    Route::post('login', 'AdminAuth\LoginController@login');
    Route::get('logout', 'AdminAuth\LoginController@logout');
});
```

dan kita tambahkan dibawah `logout`.

```php
Route::get('password/reset', 'AdminAuth\ForgotPasswordController@showLinkRequestForm')->name('admin.password.request');
Route::post('password/email', 'AdminAuth\ForgotPasswordController@sendResetLinkEmail')->name('admin.password.email');
```

lalu kita akan membuat sebuah blade dari route `admin.password.request` dimana foldernya kita tambahkan di `admin-auth` yaitu `passwords` dan mengcopy `email.blade.php` pada `auth/passwords/email.blade.php` dan kita merubah bagian `action` menjadi

```html
<form method="POST" action="{{ route('admin.password.email') }}">
```

lalu agar ada perbedaan mungkin bisa merubah headernya

```html
<div class="card-header">{{ __('Admin Reset Password') }}</div>
```

selanjutnya pada bagian login `admin` pada buttonnya `forgot password` kita akan merubah routenya yang mengarah ke `admin.password.request`.

```html
<a class="btn btn-link" href="{{ route('admin.password.request') }}">
    {{ __('Forgot Your Password?') }}
</a>
```


## Notification

pada method `sendResetLinkEmail` lalu didalamnya terdapat $this->broker()->`sendResetLink(...)` jika kita lihat pada traitnya method tsb berada pada
`vendor/laravel/framework/src/Illuminate/Auth/Passwords`.

```php
/**
* Send a password reset link to a user.
*
* @param  array  $credentials
* @return string
*/
public function sendResetLink(array $credentials)
{
    // First we will check to see if we found a user at the given credentials and
    // if we did not we will redirect back to this current URI with a piece of
    // "flash" data in the session to indicate to the developers the errors.
    $user = $this->getUser($credentials);

    if (is_null($user)) {
        return static::INVALID_USER;
    }

    if ($this->tokens->recentlyCreatedToken($user)) {
        return static::RESET_THROTTLED;
    }

    // Once we have the reset token, we are ready to send the message out to this
    // user with a link to reset their password. We will then redirect back to
    // the current URI having nothing set in the session to indicate errors.
    $user->sendPasswordResetNotification(
        $this->tokens->create($user)
    );

    return static::RESET_LINK_SENT;
}
```

mencari filenya agak sulit karena methodnya menggunakan `interface` 😡. Oke kita lihat disana terdapat `$user->sendPasswordResetNotification` nah jika kita perhatikan logikanya bahwa `$user` merupakan sebuah model yang mana variable tsb sudah dibuat secara dinamis sesuai `credentials` yang akan dikirim lalu model tsb di chaining dengan `sendPasswordResetNotification` dimana ini adalah sebuah `notification` yang akan kita buat pada model `Admin`

pembuatan notification.

```
art make:notification AdminResetPasswordNotification
```

dan akan membuat folder `Notification` dan didalamnya hasil file yang digenerate.

selanjutnya kita akan menambahkan engkapsulasi $token pada `__construct` dan pada method `toMail`.

```php
public $token;

/**
* Create a new notification instance.
*
* @return void
*/
public function __construct($token)
{
    $this->token = $token;
}
```

lalu pada method `toMail` yang by default

```php
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
        ->line('Thank you for using our application!');
}
```

menjadi

```php
/**
* Get the mail representation of the notification.
*
* @param  mixed  $notifiable
* @return \Illuminate\Notifications\Messages\MailMessage
*/
public function toMail($notifiable)
{
    return (new MailMessage)
        ->line('You are receiving this email because we received a password reset request for your account.')
        ->action('Notification Action', route('admin.password.reset', $this->token))
        ->line('Thank you for using our application!');
}
```

dimana `route` yang kita buat belum terdaftar, akan kita buat nanti. lalu parameternya berupa `token`.

Notification yang sudah kita buat akan kita `inisialisasi` pada model `Admin` karena notification ini akan digunakan model tsb.

buka file Admin model lalu tambahkan dibawahnya.

```php
public function sendPasswordResetNotification($token)
{
    return $this->notify(new AdminResetPasswordNotification($token));
}
```

okeh :v disini kita akan mereplace `sendPasswordResetNotification` pada `$user->sendPasswordResetNotification` dan menggunakan `notify` yang didalamnya sebuah notification yang sudah kita buat. jika kita mencoba untuk membuka halaman forgot password dan melakukan reset password maka akan terdapat error `Route [admin.password.reset] not defined.` karena memang routenya belum kita defenisikan. dimana ini berguna untuk menampilkan halaman `reset passwordnya` beserta `token`.

## env

sebelumnya kita akan merubah tipe driver pada .env dimana kita akan menggunakan `log` bukan `smtp` supaya memudahkan untuk mendapatkan url reset passwordnya

```
MAIL_MAILER=log
```

dimana kita bisa memantaunya di `storage/laravel.loh` kita lihat saja line paling bawah untuk melihat url yang sudah kita submit untuk melakukan reset link.

walaupun kita belum mendefenisikan `admin.password.route` pada notification yang dibuat kita berhasil mendapatkan url untuk aktifasi pada `laravel.log`

```markdown
# Hello!

You are receiving this email because we received a password reset request for your account.

Reset Password: http://www.multiple-auth.art/password/reset/bc02367de940bf4207c6ae2ecc31eea5b1a3e9bed0933ab8869b084aa648ac83?email=admin%40mail.com

This password reset link will expire in 60 minutes.
```

## Reset Password

kita belum mendefenisikan route pada `notification` yang kita buat maka kita akan mengkonfigurasinya. copy method `ResetPasswordController` di `Auth` dan mem`paste` pada `AdminAuth`.

lalu kita akan merubah beberapa yaitu `namespace` menjadi

```php
namespace App\Http\Controllers\AdminAuth;
```

dan bagian `$redirectTo`

```php
protected $redirectTo = '/admin';
```

lalu kita akan menambahkan `guest` dan `guard`

```php
public function __construct()
{
    $this->middleware('guest:admin');
}

public function guard()
{
    return Auth::guard('admin');
}
```

dimana method `guard` ini adalah override dari `trait` pada `use ResetsPasswords;`

selanjutnya kita perlu menambahkan method `broker` kita akan mengoverride juga

```php
public function broker()
{
    return Password::broker('admins');
}
```

selanjutnya kita akan membuat method yang belum didefenisikan. sebelumnya mari kita liat milik user biasa pada `route:list` menggunakan artisan

```
art route:list --columns=Method --columns=Name --columns=Action
```

lalu bisa dilihat disana ada

```
| GET|HEAD | password.reset | App\Http\Controllers\Auth\ResetPasswordController@showResetForm |
```

milik user yang mengarah pada folder yang terteta lalu menggunakan method `showResetForm` milik trait. kita akan mengoverridenya kehalaman admin reset password yang akan kita buat nanti.

tambahkan dibawahnya pada `ResetPasswordController` milik admin.

```php
/**
 * Display the password reset view for the given token.
 *
 * If no token is present, display the link request  form.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  string|null  $token
 * @return \Illuminate\Contracts\View\Factory| \Illuminate\View\View
 */
public function showResetForm(Request $request, $token = null)
{
    return view('admin-auth.passwords.reset')->with(
        ['token' => $token, 'email' => $request->email]
    );
}
```

disini kita mengoverride method `showResetForm` untuk menampilkan halaman reset passwordnya setelah user mendapatkan notification url link untuk mendapatkan halaman reset password.

lalu setelah reset password maka kita akan menggukan route `password.reset` bawaah `trait` karena kita tidak akan merubah apa2 atau mengoverridenya.

kita defenisikan dulu di routenya pada `web.php` pada prefix `admin` didalamnya bagian bawah.

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
    Route::get('login', 'AdminAuth\LoginController@showLoginForm')->name('admin.login');
    Route::post('login', 'AdminAuth\LoginController@login');
    Route::get('logout', 'AdminAuth\LoginController@logout');
    Route::get('password/reset', 'AdminAuth\ForgotPasswordController@showLinkRequestForm')->name('admin.password.request');
    Route::post('password/email', 'AdminAuth\ForgotPasswordController@sendResetLinkEmail')->name('admin.password.email');
});
```

menjadi

```php
Route::group(['prefix' => 'admin'], function () {
    Route::get('/', 'AdminController@index')->name('admin.home');
    Route::get('login', 'AdminAuth\LoginController@showLoginForm')->name('admin.login');
    Route::post('login', 'AdminAuth\LoginController@login');
    Route::get('logout', 'AdminAuth\LoginController@logout');
    Route::get('password/reset', 'AdminAuth\ForgotPasswordController@showLinkRequestForm')->name('admin.password.request');
    Route::post('password/email', 'AdminAuth\ForgotPasswordController@sendResetLinkEmail')->name('admin.password.email');
    Route::get('password/reset/{token}', 'AdminAuth\ResetPasswordController@showResetForm')->name('admin.password.reset');
    Route::post('password/reset', 'AdminAuth\ResetPasswordController@reset')->name('admin.password.update');
});
```

selanjutnya kita buat blade dari `reset` untuk `admin-auth` yang sudah didefenisikan pada route.

```php
Route::get('password/reset/{token}', 'AdminAuth\ResetPasswordController@showResetForm')->name('admin.password.reset');
```

lgsg saja copykan `reset` blade milik user biasa `auth/passwords/reset.blade.php` ke `admin-auth/password/`. lalu kita rubah bagian actionnya saja menjadi

```html
<form method="POST" action="{{ route('admin.password.update') }}">
```

selanjutnya kita akan mencoba membuka halaman forgot password dan mengirim email kepada admin. lalu ktia berhasil mengirimkannya.

selanjutnya buka bagian paling bawah pada log `storage/logs/laravel.log` terdapat link menuju route `admin.password.reset` dimana kita mendefenisikannya urlnya seperti ini `password/reset/{token}` oke.

linknya contoh saya mendapatkan `http://www.multiple-auth.art/admin/password/reset/f75e041a45e748cd4ebebef4c3e63cbc7bb412fe23d0eea5a237509339f50814` pas diakses akan muncul halaman reset passwordnya.

lalu kita coba memasukkan email dan password baru, jika sudah kita akan diredirect ke halaman home milik admin.

## Conclusion

Kita sudah menerapkan multiple authenticated dengan menduplikasi user dan memainkan guard. semoga bermanfaat.

## Contact

- email `->` kevariable@gmail.com
- telegram `->` https://t.me/kevariable
- github `->` https://github.com/kevariable
- twitter `->` https://twitter.com/kevariable
