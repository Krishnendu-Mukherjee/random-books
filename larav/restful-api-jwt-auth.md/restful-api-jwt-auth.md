# RESTful API JWT Auth

## Installation

menginstall project

```shell
$ laravel new basic-rest-api-jwt-auth
```

## Installation JWT Auth

```shell
$ restful-api-jwt-auth/; composer require tymon/jwt-auth
```

## Configuration JWT Auth

publish the config

```shell
$ art vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
```

generate secret key untuk dot env

```shell
$ art jwt:secret
```

lalu pada bagian dot env paling bawah akan ada tambahan

```
JWT_SECRET=BwLLlCW3DNYiQvGAVb7CQCTvCTwqdGp2aSMty4wg9YnY2l2wxWT7kT86DU3kDplq
```

didalam `jwt.php` yang sudah digenerate melalui artisan
terdapat `JWT time to live` atau `'ttl'` yang mengarah pada `env(JWT_TTL)` dan by default waktunya 1 jam `env(JWT_TTL, 60)` kita akan tambah agak lama pada bagian `env` diatas `JWT_SECRET`

```
JWT_TTL=999999
```

teserah aja brapa lama 🤣

## JWT Auth pada Model User

pada model User kita tambahkan `implements` setelah `extends Authenticatable`

```php
class User extends Authenticatable implements JWTSubject
```

jangan lupa import `JWTSubject` di bawah

```php
use Illuminate\Notifications\Notifiable;
use Tymon\JWTAuth\Contracts\JWTSubject; // disini
```

kita akan tambahkan paling bawahnya

```php
/**
 * Get the identifier that will be stored in the  subject claim of the JWT.
 *
 * @return mixed
 */
public function getJWTIdentifier()
{
    return $this->getKey();
}

/**
 * Return a key value array, containing any custom claims to be added to the JWT.
 *
 * @return array
 */
public function getJWTCustomClaims()
{
    return [];
}
```

`$this->getKey()` berguna ketika setiap melakukan request untuk mengidentifier menggunakan token dari `JWT Auth`.

## Guard Configuration

kita akan merubah beberapa konfigurasi karena menggunakan JWT Auth di `auth.php` config

```php
'defaults' => [
    'guard' => 'api', // rubah
    'passwords' => 'users',
],

.......

'guards' => [
    'api' => [
        'driver' => 'jwt', // rubah
        'provider' => 'users',
        'hash' => false,
    ],
],
```

## Merubah structure Model ke folder Models

```shell
$ mkdir app/Models; mv app/User.php app/Models
```

lalu merubah `namespace` User

```php
namespace App\Models;
```

lalu juga kita akan merubah `providers` di `auth.php`

```php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\Models\User::class, // disini tambahin
    ],

    // 'users' => [
    //     'driver' => 'database',
    //     'table' => 'users',
    // ],
],
```

## Membuat migration

disini kita akan membuat sebuah `post` sederhana dimana `post` ini dimiliki oleh `user` dan satu `user` bisa memiliki banyak `post` dan setiap `post` memili banyak `tag`. Jika dianaloginkan bahwa `user` dan `post` memiliki hubungan relasi `one-to-many` dan juga banyak `post` bisa dimiliki oleh banyak `tag`, dengan kata lain bahwa `post` dan `tag` memiliki hubungan `many to many`.

pertama kita perlu menambahkan `username` pada table `user`

```shell
$ art make:migration add_username_to_users_table --table=users
```

lalu buka filenya.

```php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('username')->after('id');
    });
}
```

dan pada method `down`

```php
public function down()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('username');
    });
}
```

lansung saja kita akan membuat migration untuk `post`

```shell
$ art make:migration create_posts_table --create=posts
```

lalu kita buka filenya. Dan mengisini seperti ini.

```php
public function up()
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->string('title');
        $table->string('slug');
        $table->text('body');
        $table->timestamps();
    });
}
```

disini kita membuat sebuah foreign key yang berelasi ke table `users` melalui `constrained` menggunakan default name sesuai `naming convention` bisa juga `constrained('some_table')` lalu di ikuti `cascadeOnDelete` ketika `user` dihapus maka `posts`nya juga akan dihapus.

migration untuk `tag`

```shell
$ art make:migration create_tags_table --create=tags
```

kita hanya perlu menambahkan field `name`.

```php
$table->string('name');
```

karena `posts` dan `tags` memiliki hubungan `many-to-many` maka akan kita buat table pivotnya.

```shell
$ art make:migration create_post_tag_table --create=post_tag
```

jika diperhatikan bahwa kita membuat sesuai naming convention yaitu `alphabet` dan `singular`.

lalu didalamnya.

```php
public function up()
{
    Schema::create('post_tag', function (Blueprint $table) {
        $table->unsignedBigInteger('tag_id');
        $table->unsignedBigInteger('post_id');
    });
}
```

## Migrate

sebelum itu silahkan sesuaikan dot envnya kepada mysql

```
DB_DATABASE=lv-restful-api-jwt-auth
DB_USERNAME=root
DB_PASSWORD=root
```

pastikan buat databasenya terlebih dahulu, jika `mysql`nya tidak menggunakan password maka kosongkan saja.

lalu kita migrate

```shell
$ art migrate
```

## Models

kita akan membuat relasi.

```shell
$ art make:model Models\\Post; art make:model Models\\Tag
```

sudah dibuat, selanjutnya buka model User dan tambahkan paling bawah.

```php
public function posts()
{
    return $this->hasMany(Post::class, 'user_id');
}
```

lalu dibagian model `Post`

```php
public function user()
{
    return $this->belongsTo(User::class, 'user_id', 'id');
}
```

selanjutnya kita akan merelasikan antara `post` dan `tag`

pada model `post` dibawah method `user`.

```php
public function tags()
{
    return $this->belongsToMany(Tag::class, 'post_tag');
}
```

dan pada model `tag`

```php
public function posts()
{
    return $this->belongsToMany(Post::class, 'post_tag');
}
```

## Mass Assignment

tentu saja ketika kita sudah mendeklarasikan sebuah migration dan sebuah relasi pada model tentu saja kita akan membuat sebuah mass assignment untuk mengijinkan sebuah field dapat dirubah dan di manipulasi kedalam database. karena sebelumnya di `migration` kita menambahkan `username` kita akan menambahkan di model `user` pada `$fillable`

```php
protected $fillable = [
    'name', 'email', 'password', 'username'
];
```

selanjutnya pada model `posts`

```php
protected $fillable = [
    'title', 'slug', 'body'
];
```

perlu diketahui yang perlu dimasukkan adalah hal yang perlu kita manipulasi seperti `crud` jika sebuah `id` maupun `foreign key` bakal digenerate otomatis tanpa perlu memanipulasinya menggunakan `Eloquent ORM`.

lalu pada model `tag`

```php
protected $fillable = [
    'name'
];
```

## Registerkan user

lansung saja kita buat sebuah controller `login`

```shell
$ art make:controller Auth\\RegisterController -i
```

kita menggunakan `__invoke`

lalu juga membuat sebuah `request`

```
$ art make:request Auth\\RegisterRequest
```

pertama kita buka file request dari Register pada `app/Http/Request/Auth`.

lalu method `authorize()` ubah menjadi `true` dan method `rules` adalah tempat memvalidasi user.

```php
public function rules()
{
    return [
        'username' => ['min:6', 'max:25', 'required'],
        'name' => ['string', 'required', 'max:255'],
        'email' => ['required', 'email'],
        'password' => ['required', 'string', 'min:6'],
    ];
}
```

lalu kita import bagian `register` controller

```php
use App\Http\Requests\Auth\RegisterRequest;
```

dan pada method `__invoke` didalam parameter kita ubah menjadi

```php
public function __invoke(RegisterRequest $request)
```

dan hapus.

```php
use Illuminate\Http\Request;
```

karena kita tidak membutuhkannya lagi.

lalu kita tambahkan fungsi untuk meregister pada `__invoke`

```php
public function __invoke(RegisterRequest $request)
{
    return $this->createUser($request);
}
```

dan dibawah method `__invoke` kita buat sebuah method `createUser`

```php
public function createUser($request)
{
    return User::create([
        'name' => $request->name,
        'username' => $request->username,
        'email' => $request->email,
        'password' => bcrypt($request->password)
    ]);
}
```

## Mendaftarkan route Register

pada `routes/api.php`.

```php
Route::post('register', 'Auth\RegisterController');
```

tapi ada baiknya kita akan coba grouping menggunakan namespace `Auth`

```php
Route::group(
    ['prefix' => 'user', 'namespace' => 'Auth'],
    function () {
        Route::post('register', 'RegisterController');
    }
);
```

jika kita melakukan artisan melihat route

```
restful-api-jwt-auth on 🚧 mast [🗑📝⚠ ] via ⬢ v14.5.0 via 🐘 v7.4.7
✗  art route:list -c
+----------+-------------------+----------------------------------------------+
| Method   | URI               | Action                                       |
+----------+-------------------+----------------------------------------------+
| GET|HEAD | /                 | Closure                                      |
| GET|HEAD | api/user          | Closure                                      |
| POST     | api/user/register | App\Http\Controllers\Auth\RegisterController |
+----------+-------------------+----------------------------------------------+
```

penjelasan `namespace` akan mempersingkat penulisan controller dari pada `Auth\RegisterController` lebih baik `RegisterController` saja dan `prefix` perguna melakukan membuat `root` url andaikata prefixnya adalah `user` maka `url.art/api/user/` dan route didalamnya misal `register` dia akan menggabungkannya dengan url user tsb seperti `route:list` diatas.

selanjutnya buka postman untuk mengetestnya dan buat sebuah `Collection` bernama `restful-api-jwt-auth` lalu `Create` dan kita tambahkan sebuah `request` bernama `Register` lalu buka requestnya dan arahkan ke `url` pada `register` dengan method `POST` pada postman

```
http://restful-api-jwt-auth.art/api/user/register
```

dan kita isi `body` menggunakan `raw` dan typenya `json`

```json
{
    "name": "Kevin Abrar Khansa",
    "username": "kevariable",
    "password": "example",
    "email": "kevariable@mail.com"
}
```

jika di send maka akan berhasil yang akan menampilkan data yang telah kita buat.

## Login dulu.

setelah register sudah tentu kita akan melakukan sebuah login nah kita akan mencoba login menggunakan helper `Auth` ataupun `Miscellaneous` dengan `auth()`.

```shell
$ art make:controller Auth\\LoginController -i
```

dan juga kita buatkan requestnya

```shell
$ art make:request Auth\\LoginRequest
```

selanjutnya kita akan membuka file request yang sudah dibuat `LoginRequest`, seperti biasa pada bagian method `authorize` rubah menjadi `true` dan `rules`nya seperti ini.

```php
public function rules()
{
    return [
        'username' => ['min:6', 'max:25', 'required'],
        'password' => ['min:6', 'required']
    ];
}
```

selanjutnya bagian controller `LoginController`. kita rubah `request` menjadi `LoginRequest` dan mengimportnya.

```php
....
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest; // import
....

public function __invoke(LoginRequest $request) // tambah disini
```

selanjutnya kita akan melakukan percobaan login didalam method `__invoke`.

```php
public function __invoke(LoginRequest $request)
{
    if (!$token = $this->attemptLogin($request))
        return response(null, 401); // unauthorized
    
    return $this->respondWithToken($token);
}
```

nah disini kita akan menambahkan method `attemptLogin` dan `respondWithToken` dimana `attemptLogin` akan kita buat untuk melakukan percobaan login melalui `credential` yang diberikan sebagai parameter dan `respondWithToken` berguna setelah melakukan percobaan login lalu hasilnya akan di `assign` kevariable `$token` yang akan menampil token `jwt` yang akan digenerate otomatis.

tambahkan method dibawah `__invoke`.

```php
private function attemptLogin($request)
{
    return auth()->attempt($request->only('username', 'password'));
}
```

ini berarti didalam `postman` nanti kita akan mengirimkan data `username` dan `password` saja untuk login

selanjutnya kita beri responsenya berupa token `jwt` menggunakan method `respondWithToken` yang akan kita baut dibawahnya.

```php
private function respondWithToken($token)
{
    return response()->json([
        'token' => $token,
        'token_type' => 'bearer',
        'expires_in' => auth()->factory()->getTTL() // masa expired login
    ])
}
```

## Mendaftarkan route login

pada bagian controller kita sudah berhasil mengkonfigurasinya dengan baik selanjutnya kita tambahkan route pada `api.php` dibagian grouping yang sudah dibuat.

```php
Route::group(
    ['prefix' => 'user', 'namespace' => 'Auth'],
    function () {
        Route::post('register', 'RegisterController');
        Route::post('login', 'LoginController'); // tambahkan disini
    }
);
```

## Percobaan login

selanjutnya bukak postman dan buat request didalam collection yang sudah ada dengan nama `Login`, lalu tambahkan url `http://restful-api-jwt-auth.art/api/user/login` dengan methodnya sebagai `POST`,
lalu pada bagian `Headers` jgn lupa tambahkan `Accept` sebagai `key` dan `application/json` sebagai `value`.
Lalu kita buka bagian `body` dan pilih `raw` dengan type `json` dan isi.

```json
{
    "username": "kevariable",
    "password": "example"
}
```

lalu kita send dan akan tampil kira2 seperti ini.

```json
{
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOlwvXC9yZXN0ZnVsLWFwaS1qd3QtYXV0aC5hcnRcL2FwaVwvdXNlclwvbG9naW4iLCJpYXQiOjE1OTUyNjE4MTQsImV4cCI6MTYwMTI2MTc1NCwibmJmIjoxNTk1MjYxODE0LCJqdGkiOiJBZE5DMHpGRXlCYUNlMFFuIiwic3ViIjoxLCJwcnYiOiIyM2JkNWM4OTQ5ZjYwMGFkYjM5ZTcwMWM0MDA4NzJkYjdhNTk3NmY3In0.t4W_KyeQseJahiajWyOihhNGLIVIvBz7IipT27ruLXI",
    "token_type": "bearer",
    "expires_in": "99999"
}
```

token kita akan berbeda2 jgn khawatir.

## Memastikan apakah kita sudah login?

kita akan membuat sebuah controller untuk `user` untuk mendapatkan nama user yang login.

```shell
$ art make:controller UserController -i
```

lalu kita buka filenya dan rubah.

```PHP
public function __construct()
{
    $this->middleware('auth:api'); // perlu login sebelum bisa mengakses
}

public function __invoke(Request $request)
{
    return $request->user()->name;
}
```

cukup simple kita menambahkan middleware dan pada method `__invoke` mereturn data `user` yang sedang login, selanjutnya kita akan menambahkan routenya

```php
// Route::middleware('auth:api')->get('/user', function (Request $request) {
//     return $request->user();
// });

Route::get('user', 'UserController'); // tambahkan disini

Route::group(
    ['prefix' => 'user', 'namespace' => 'Auth'],
    function () {
        Route::post('register', 'RegisterController');
        Route::post('login', 'LoginController');
    }
);
```

kita membuatnya terpisah karena kita tidak berada di namespace `Auth` dan kita comment route bawaan diatasnya, sebenarnya route bawaan itu juga akan mereturn data user yang sedang login tetapi kita akan mencoba menggunakan controller yang sudah kita buat.

selanjutnya kita akan coba dipostman. buat sebuah request `User` dan tambahkan headernya menjadi `json` lalu sebelumnya kita akan copy `token` ketika kita menggunakan `Login` request pada postman.

![login dapetkan token](https://i.imgur.com/6Ezmp5h.png)

tokennya kita masukkan ke `User` pada bagian `Authorization` lalu pilih `type` menjadi `Bearer Token` dan pastekan token yang sudah kita copy sebelumnya.

![alt](https://i.imgur.com/9zE0fpe.png)

lalu kita tambahkan urlnya dengan method `GET`

```
http://restful-api-jwt-auth.art/api/user
```

dan kita send, akan muncul nama user yang sudah login seperti gambar sebelumnya.

```
Kevin Abrar Khansa
```

## Logout

sekarang kita coba logoutkan user.

```shell
$ art make:controller Auth\\LogoutController -i
```

lalu buka filenya dan ubah menjadi seperti ini.

```php
public function __invoke()
{
    auth()->logout();

    if (auth()->check() === false)
        return response()->json([
            'status' => 'logout'
        ]);
}
```

pertama kita menglogoutkan menggunakan `auth()->logout()` dan mengecek apakah kita sudah dalam keadaan logout dengan mengecek dengan conditional `auth()->check`.

## daftarkan route logout

```php
Route::group(
    ['prefix' => 'user', 'namespace' => 'Auth'],
    function () {
        Route::post('register', 'RegisterController');
        Route::post('login', 'LoginController');
        Route::post('logout', 'LogoutController'); // tambahkan disini
    }
);
```

## Mencoba logout

lalu kita akan mencoba logout dengan postman, silahkan buat sebuah request dengan nama `Logout` dan isi urlnya dan methodnya `POST`

```
http://restful-api-jwt-auth.art/api/user/logout
```

dan pada authirazition ubah typenya `beraer token` dan isi tokennya yang sudah ada pada request `Login`

selanjutnya send. Dan akan mendapatkan response

```json
{
    "status": "logout"
}
```

yang berarti kita sudah logout.

## Post Controller

disini kita akan membuat sebuah post dengan memanfaatkan sebuah relation antara user dan post.

```shell
$ art make:controller PostController --api
```

artisan --api akan mengenerate sebuah crud seperti `store`, `update`, `index`, `show`, `destroy` kecuali `create` dan `edit` karena halamannya akan di handle di front end.

## Store

kita akan membuka file PostController dan melihat ada method `store` disana kita akan membuat sebuah `post` tapi sebelum itu ada baiknya kita membuat sebuah `request` dengan terpisah agar kode terlihat lebih ramping.

```
$ art make:request Post\\StoreRequest
```

dan kita import ke dalam `PostController` kedalam parameter `store`

```php
use App\Http\Request\Post\StoreRequest;
.....

public function store(StoreRequest $request)
{

}
```

selanjutnya kita membuka file request yang telah dibuat.

```php
public function authorize()
{
    return true; // menjadi true
}
```

lalu bagian `rules`

```php
public function rules()
{
    return [
        'title' => ['min:3', 'max:50', 'required'],
        'body' => ['required'],
        'tags' => ['array', 'required'],
        'tags.*' => ['exists:tags,id']
    ];
}
```

nah karena nanti bagian `tags` akan di isi sebuah array dan didalamnya sebuah `id` tag yang akan di assign menggunaman `sync`

lalu jika `id` tidak ada di `tags` table kita akan handle errornya dengan membuat method baru dibawahnya.

```php
public function messages()
{
    return [
        'tags.*.exists' => 'error cuk gk nemu :v'
    ];
}
```

selanjutnya kita akan susun logic di `store`

```php
public function store(StoreRequest $request)
{
    $userPost = auth()->user()->posts()->create([
        'title' => $request->title,
        'slug' => Str::slug($request->title),
        'body' => $request->body
    ]);

    $userPost->tags()->sync($request->tags);

    return response()->json($userPost);
}
```

kita disini mengambil user yang login lalu merelasikannya dengan method `posts` yang sudah dibuat pada model `User` lalu kita buat dibawahnya method `tags` yang sudah dideclare pada model `Post` dimana kita akan megassign sebuah `id` ketable `pivot` tetapi ktia belum membuat data `tags` secara manual. Tapi jika dilihat ini terlalu gemuk kita perlu membuat `store` lebih ramping dengan membuat method baru yang akan menghandle `request`. Kita buat method `credentials` pada bawah sekali.

```php
public function credentials($request)
{
    return [
        'title' => $request->title,
        'slug' => Str::slug($request->title),
        'body' => $request->body
    ];
}
```

dengan begini kita bisa ubah method `store` menjadi seperti ini.

```php
public function store(StoreRequest $request)
{
    $userPost = auth()->user()->posts()->create(
        $this->credentials($request)
    );

    $userPost->tags()->sync($request->tags);

    return response()->json($userPost);
}
```

nah dengan begitu method `credentials` juga bisa kita terapkan untuk method `update` nanti.

lalu kita mendaftarkan Routenya di `routes/api.php` dibawahnya.

```php
Route::group(
    ['prefix' => 'posts'],
    function () {
        Route::post('/', 'PostController@store');
    }
);
```

karena kita tidak memasang middleware kita akan pasang dibagian `PostController` buat method `__construct`. kenapa tidak di route saja pasang middleware? karena kita sudah menggunakan `group` kita akan menghandle middleware dengan pengecualian method `index` dan `show` karena method ini tak perlu login terlebih dahulu.

```php
public function __construct()
{
    $this->middleware('auth:api')->except('index', 'show');
}
```

nah kita pasang middleware dengan `auth` lalu menggunakan `guard` api.

selanjutnya percobaan menggunakan `postman`

set up seperti dengan nama `Store` lalu dengan method `POST` jangan lupa copykan `Bearer Token` login ke request yang dibuat.

lalu urlnya.

```
http://restful-api-jwt-auth.art/api/posts
```

jgn lupa headernya `Accept | application/json`.

sebelum itu kita buat data dummy untuk `Tags` menggunakan Tinker

```shell
$ art tinker
```

lalu kita create datanya

```php
>>> Tag::create(['name' => 'PHP'])
....
>>> Tag::create(['name' => 'Laravel'])
....
>>> Tag::create(['name' => 'JavaScript'])
```

selanjutnya pada bagian `body` postman pilih `raw` dengan type `JSON`.

```json
{
    "title": "Belajar Rest API",
    "body": "ini body",
    "tags": [
        1, 2
    ]
}
```

nah id pada `tags` 1 dan 2 adalah `PHP` dan `Laravel`.

lalu jika kita send akan menampilkan data json yang telah kita buat.

## Update

lalu pada bagian method `PostController` bagian `update` method tidak jauh beda sama `store`.

```php
public function update(UpdateStore $request, Post $post)
{
    $post->update(
        $this->credentials($request)
    );

    $post->tags()->sync($request->tags);

    return response()->json($post);
}
```

kira2 seperti ini dimana parameternya adalah sebuah request yang belum kita buat silahkan buat sendiri samakan seperti StoreRequest, lah kenapa gk pake StoreRequest aja? yap kita bisa tapi jika ada perubahan kita akan mudah memaintancenya. Selanjutnya yaitu parameter berupa model `Post $post` yang akan menerima parameter berupa `slug` di `route` nanti supaya kita bisa update.

lalu dibagian route kita tambah dibawah `store`

```php
Route::group(
    ['prefix' => 'posts'],
    function () {
        Route::post('/', 'PostController@store');
        Route::patch('{post:slug}', 'PostController@update'); // tambah disini
    }
);
```

bisa dilihat kita menggunakan method `PATCH` karena kita akan mengupdate sebuah data hanya 1 saja tidak sekaligus seperti `PUT` dan karena kita menggunakan `prefix` posts maka kita hanya perlu membuat `{post:slug}` sebagai tambahannya.

lgsg saja mencoba dipostman membuat request `Update` dengan method `PATCH` dengan url

```
http://restful-api-jwt-auth.art/api/posts/belajar-rest-api
```

nah `belajar-rest-api` adalah slugnya yang akan kita update datanya. jangan lupa masukkan `Bearer Token` login ke dalam request yg dibuat beserta header menjadi json.

dan untuk bodynya menggunakan `raw` dgn type `json`

```json
{
    "title": "Belajar Laravel 7",
    "body": "amazing",
    "tags": [1]
}
```

maka data dengan slug `belajar-rest-api` akan berubah seperti data yang kita masukkan.

## Destroy

```php
public function destroy(Post $post)
{
    $post->delete();

    return response()->json([
        'has_deleted' => $post
    ]);
}
```

Lalu buat routenya

```php
Route::group(
    ['prefix' => 'posts'],
    function () {
        Route::post('/', 'PostController@store');
        Route::patch('{post:slug}', 'PostController@update');
        Route::delete('{post:slug}', 'PostController@destroy'); // tambah disini
    }
);
```

lalu untuk `postman` buat request `Delete` lalu dengan `Bearer Token` login dan header sebagai json lalu methodnya `DELETE` dan urlnya.

```
http://restful-api-jwt-auth.art/api/posts/belajar-rest-api
```

`belajar-rest-api` sebagai slug yang akan kita hapus datanya.

## Show & API Resource

selanjutnya pada bagian method `show` kita akan menampulkan sebuah data berdasarkan slug.

```php
public function show(Post $post)
{
    return response()->json($post);
}
```

ini adalah cara paling mudah tapi bukan kepada `approach` json.

maka dari itu kita akan menggunakan `resource`.

```shell
$ art make:resource PostResource
```

ini akan mengenerate file pada `app/Http/Resources/PostResource`.

lalu kita ubah return method `show` tadi menggunakan `resource` yang kita buat.

```php
use App\Http\Resources\PostResource;

.....

public function show(Post $post)
{
    return new PostResource($post);
}
```

kita akan daftarkan routenya.

```php
Route::group(
    ['prefix' => 'posts'],
    function () {
        Route::get('{post:slug}', 'PostController@show'); // tambah disini
        Route::post('/', 'PostController@store');
        Route::patch('{post:slug}', 'PostController@update');
        Route::delete('{post:slug}', 'PostController@destroy');
    }
);
```

selanjutnya uji coba di browser.

```
http://restful-api-jwt-auth.art/api/posts/belajar-rest-api
```

maka akan tampil sebuah json.

```json
{
    "data": {
        "id":5,
        "user_id":1,
        "title":"Belajar Rest API","slug":"belajar-rest-api",
        "body":"nganu in anak orang :v","created_at":"2020-07-21T16:58:03.000000Z","updated_at":"2020-07-21T16:58:03.000000Z"}
    }
}
```

maka akan tampil seperti ini. Nah disini kita akan bermain dengan `resources`.

bagaimana jika kita ingin menambahkan sebuah `meta` ? dibawah data.

sekarang buka `PostResource` pada kita buat sebuah method `with` dibawah method `toArray`.

```php
public function with($request)
{
    return [
        'meta' => [
            'publish_at' => $this->created_at->diffForHumans()
        ]
    ];
}
```

nah kita membuat sebuah meta didalamnya terdapat `publish_at` dimana kita memanggil column `created_at` lalu kita manipulasi atau merubahnya kedalam bentuk yang mudah dibaca.

```json
"publish_at": "5 minutes ago"
```

kira2 seperti ini yang akan tampil. Selain meta atau column bisa dirubah sesuai kebutuhan. jika kita lihat keseluruhan maka akan seperti ini.

```json
{
    "data": {
        "id": 5,
        "user_id": 1,
        "title": "Belajar Rest API",
        "slug": "belajar-rest-api",
        "body": "nganu in anak orang :v",
        "created_at": "2020-07-21T16:58:03.000000Z",
        "updated_at": "2020-07-21T16:58:03.000000Z"
    },
    "meta": {
        "publish_at": "9 minutes ago"
    }
}
```

lalu timbul pertanyaan bagaimana jika kita hanya butuh data `title`, `slug`, `body` saja?. Baik kita akan merubah pada method `toArray`

```php
public function toArray($request)
{
    return parent::toArray($request);
}
```

menjadi seperti ini.

```php
public function toArray($request)
{
    return [
        "title" => $this->title,
        "slug" => $this->slug,
        "body" => $this->body,
    ];
}
```

maka akan tampil seperti ini.

```json
{
    "data": {
        "title": "Belajar Rest API",
        "slug": "belajar-rest-api",
        "body": "nganu in anak orang :v"
    },
    "meta": {
        "publish_at": "12 minutes ago"
    }
}
```

jika ingin menamipulasi atau menambahkan method akan bisa sesuai keinginan misalkan menampilkan sebuah relation yaitu method `tags()` yang sudah ada dimodel `Post`.

```php
public function toArray($request)
{
    return [
        "title" => $this->title,
        "slug" => $this->slug,
        "body" => $this->body,
        "tag_name" => $this->tags
    ];
}
```

maka akan tampil seperti ini.

```json
"data": {
    "title": "Belajar Rest API",
    "slug": "belajar-rest-api",
    "body": "nganu in anak orang :v",
    "tag_name": [
        {
            "id": 1,
            "name": "PHP",
            "created_at": "2020-07-21T08:19:08.000000Z",
            "updated_at": "2020-07-21T08:19:08.000000Z",
            "pivot": {
                "post_id": 5,
                "tag_id": 1
            }
        },
        {
            "id": 2,
            "name": "Laravel",
            "created_at": "2020-07-21T08:19:13.000000Z",
            "updated_at": "2020-07-21T08:19:13.000000Z",
            "pivot": {
                "post_id": 5,
                "tag_id": 2
            }
        }
    ]
},
"meta": {
    "publish_at": "15 hours ago"
}
```

bila ingin specifict lagi misalkan hanya ingin menampilkan nama tagnya saja

```php
public function toArray($request)
{
    return [
        "title" => $this->title,
        "slug" => $this->slug,
        "body" => $this->body,
        "tags" => $this->tags->pluck('name')
    ];
}
```

```json
{
    "data": {
        "title": "Belajar Rest API",
        "slug": "belajar-rest-api",
        "body": "nganu in anak orang :v",
        "tags": [
            "PHP",
            "Laravel"
        ]
    },
    "meta": {
        "publish_at": "22 hours ago"
    }
}
```

## JsonResource Unwrapping

kadang2 kita hanya butuh data tanpa sebuah meta, misalkan saja kita punya json seperti ini di laravel.

```json
{
    "data": {
        "title": "Belajar Rest API",
        "slug": "belajar-rest-api",
        "body": "nganu in anak orang :v",
        "tags": [
            "PHP",
            "Laravel"
        ]
    },
    "meta": {
        "publish_at": "22 hours ago"
    }
}
```

lalu kita ingin merubahnya seperti ini.

```json
{
    "title": "Belajar Rest API",
    "slug": "belajar-rest-api",
    "body": "nganu in anak orang :v",
    "tags": [
        "PHP",
        "Laravel"
    ]
}
```

bagaimana caranya?. Pertama buka folder `app/Providers/AppServiceProvider`

lalu pada bagian `boot` tulis seperti ini.

```php
use Illuminate\Http\Resources\Json\JsonResource;

....

JsonResource::withoutWrapping();
```

jika kita coba maka tadanya tetap sama

```json
{
    "data": {
        "title": "Belajar Rest API",
        "slug": "belajar-rest-api",
        "body": "nganu in anak orang :v",
        "tags": [
            "PHP",
            "Laravel"
        ]
    },
    "meta": {
        "publish_at": "22 hours ago"
    }
}
```

kenapa demikian? karena kita masih mempunyai method `with` didalam PostResource, jika kita comment.

```php
// public function with($request)
// {
//     return [
//         'meta' => [
//             'publish_at' => $this->created_at->diffForHumans()
//         ]
//     ];
// }
```

dan coba lagi.

```json
{
    "title": "Belajar Rest API",
    "slug": "belajar-rest-api",
    "body": "nganu in anak orang :v",
    "tags": [
        "PHP",
        "Laravel"
    ]
}
```

maka datanya seperti ini.

## Index & API Resource Collection

sekarang kita akan menampilkan keseluruhan data. Pada method `index`

```php
public function index()
{
    $post = Post::all();

    return new PostResource($post);
}
```

dan kita coba buat sebuah route di `api.php`

```php
Route::group(
    ['prefix' => 'posts'],
    function () {
        Route::get('/', 'PostController@index'); // tambah disini
        Route::get('/{post:slug}', 'PostController@show');
        Route::post('/', 'PostController@store');
        Route::patch('{post:slug}', 'PostController@update');
        Route::delete('{post:slug}', 'PostController@destroy');
    }
);
```

dan kita buka di browser atau menggunakan postman dengan method `GET`.

maka kita akan mendapatkan error

```json
{
    "message": "Property [title] does not exist on this collection instance.",
    .....
```

bahwa Resouce yang kita punya tidak ada collection maka dari itu kita bisa merubah controller kita di `index` menjadi seperti ini.

```json
{
    "data": [
        {
            "title": "Jago bang",
            "slug": "jago-bang",
            "body": "oke sip",
            "tags": [
                "PHP"
            ]
        },
        {
            "title": "Belajar Rest API",
            "slug": "belajar-rest-api",
            "body": "nganu in anak orang :v",
            "tags": [
                "PHP",
                "Laravel"
            ]
        }
    ]
}
```

> note: saya sudah menghapus `JsonResource::unwrapping()`

akan tampil semua data.

tapi bagaimana jika kita membutuhkan sebuah `meta`?

kita uncomment `method with` yang ada di Resource.

```php
 public function with($request)
{
    return [
        'meta' => [
            'publish_at' => $this->created_at->diffForHumans()
        ]
    ];
}
```

lalu kita coba lagi.

```json
{
    "data": [
        {
            "title": "Jago bang",
            "slug": "jago-bang",
            "body": "oke sip",
            "tags": [
                "PHP"
            ]
        },
        {
            "title": "Belajar Rest API",
            "slug": "belajar-rest-api",
            "body": "nganu in anak orang :v",
            "tags": [
                "PHP",
                "Laravel"
            ]
        }
    ]
}
```

datanay tetap sama. Itu karena kita menggunakan method `static` collection, jika ingin menggunakan `meta` maka kita butuh sebuah `instance` objek `new` keyword untuk melakukan itu, nah solusi apa? solusinya adalah sebuah `Collection` bagaimana cara membuatnya? sama seperti halnya membuat sebuah resource akan tetapi kita akan merubah kara Resource menjadi `Collection` Ex: `PostCollection`.

## Resource Collection

kita akan membuat sebuah collection yang akan menampilkan semua data dengan meta.

```
$ art make:resource PostCollection
```

jika kita buka maka dia tidak lagi meng`extends` ke `JsonResource` melainkan `ResourceCollection` nah jika kita rubah lagi method `index` menjadi seperti ini.

```php
public function index()
{
    $post = Post::all();

    return new PostCollection($post);
}
```

maka jika kita hit lagi akan tampil keseluruhan data.

```json
{
    "data": [
        {
            "title": "Jago bang",
            "slug": "jago-bang",
            "body": "oke sip",
            "tags": [
                "PHP"
            ]
        },
        {
            "title": "Belajar Rest API",
            "slug": "belajar-rest-api",
            "body": "nganu in anak orang :v",
            "tags": [
                "PHP",
                "Laravel"
            ]
        }
    ]
}
```

nah tidak tampil `meta`nya kita pelu buat method `with` lagi.

```php
public function with($request)
{
    return [
        'meta' => [
            'post_count' => $this->collection->count()
        ]
    ];
}
```

disini kita bisa memanfaatkan `$this->collection` untuk mengambil semua data `Post` dan menggunakan chaining method `count()` untuk menghitung brapa buat post yang ada.

jika kita coba lagi.

```json
{
    "data": [
        {
            "id": 1,
            "user_id": 1,
            "title": "Jago bang",
            "slug": "jago-bang",
            "body": "oke sip",
            "created_at": "2020-07-21T08:15:58.000000Z",
            "updated_at": "2020-07-21T08:53:58.000000Z"
        },
        {
            "id": 5,
            "user_id": 1,
            "title": "Belajar Rest API",
            "slug": "belajar-rest-api",
            "body": "nganu in anak orang :v",
            "created_at": "2020-07-21T16:58:03.000000Z",
            "updated_at": "2020-07-21T16:58:03.000000Z"
        }
    ],
    "meta": {
        "post_count": 2
    }
}
```

maka akan tampil.

## Customize API Resource Collection

kita akan belajar bagaimana mengcustomize sebuah collection pada method `toArray()`.

nah jika kita melihat default untuk menampilkan seluruh data.

```php
public function toArray($request)
{
    return parent::toArray($request);
}
```

karena kita tidak bisa mengcustomizenya didalam collection kita pelu mengimport data `PostResource` menggunakan `$this->collection`.

```php
public function toArray($request)
{
    return [
        'data' => PostResource::collection($this->collection)
    ];
}
```

nah kita panggill class `PostResoruce` lalu menggunakan static method collection dan mengirim sebuah data berupa `collection` yang akan di customize pada `PostResource`.

jika kita coba hit lagi endpointnya.

```json
{
    "data": [
        {
            "title": "Jago bang",
            "slug": "jago-bang",
            "body": "oke sip",
            "tags": [
                "PHP"
            ]
        },
        {
            "title": "Belajar Rest API",
            "slug": "belajar-rest-api",
            "body": "nganu in anak orang :v",
            "tags": [
                "PHP",
                "Laravel"
            ]
        }
    ],
    "meta": {
        "post_count": 2
    }
}
```

maka kita berhasil melakukan customize. karena `Resource` tidak bisa menampilkan meta maka method `with` yang ada di `PostResource` tidak akan tampil di `PostCollection` karena itulah kita butuh bantuan dari `PostCollection` untuk menampilkan sebuah `meta` atau data lainnya.

## Menambahkan Resource pada `Store` dan `Update`

kita juga akan merubah return dari method `store` dan `update` ketimbang membuatnya seperti `response()->json()` agar mudah mengcustomize dan tidak juga memberatkan beban `controller`

```php
public function store(StoreRequest $request)
{
    $userPost = auth()->user()->posts()->create(
        $this->credentials($request)
    );

    $userPost->tags()->sync($request->tags);

    return new PostResource($userPost); // rubah seperti ini
}
```

nah karena kita hanya menampilkan 1 data maka kita hanya perlu menggunakan `rersource` saja.

dan method `update`

```php
public function update(UpdateRequest $request, Post $post)
{
    $post->update(
        $this->credentials($request)
    );

    $post->tags()->sync($request->tags);

    return new PostResource($post); // rubah disini
}
```

## Summary

Keep learning what u want!.

- Kevin Abrar Khansa, 22 July 2020, 23:17:22

Thank for reading.

## Contact

- email `->` kevariable@gmail.com
- telegram `->` https://t.me/kevariable
- github `->` https://github.com/kevariable
- twitter `->` https://twitter.com/kevariable
