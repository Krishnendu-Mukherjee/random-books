# Laravel Excel `->` Export

## Introduction

🚀 Laravel Excel dimaksudkan untuk menjadi PhpSpreadsheet yang diberi rasa Laravel: pembungkus sederhana namun elegan di sekitar PhpSpread dengan tujuan menyederhanakan ekspor dan impor.

## 🔥 PhpSpreadsheet

adalah perpustakaan yang ditulis dalam PHP murni dan menyediakan sekumpulan kelas yang memungkinkan Anda membaca dan menulis ke berbagai format file spreadsheet, seperti Excel dan LibreOffice Calc.

## Requirement

buka file `php.ini` menggunakan editor andalan tentunya diawali dengan `sudo`.

```
sudo vi /etc/php/php.ini
```

lalu uncomment bagian `gd` dan `iconv`

```ini
;extension=bcmath
;extension=bz2
;extension=calendar
extension=curl
;extension=dba
;extension=enchant
;extension=exif
;extension=ffi
;extension=ftp
extension=gd
;extension=gettext
;extension=gmp
extension=iconv
;extension=imap
;extension=intl
;extension=ldap
extension=mysqli
;extension=odbc
;zend_extension=opcache
;extension=pdo_dblib
extension=pdo_mysql
;extension=pdo_odbc
;extension=pdo_pgsql
;extension=pdo_sqlite
;extension=pgsql
;extension=pspell
;extension=shmop
;extension=snmp
;extension=soap
;extension=sockets
;extension=sodium
;extension=sqlite3
;extension=sysvmsg
;extension=sysvsem
;extension=sysvshm
;extension=tidy
;extension=xmlrpc
extension=xsl
extension=zip
```

by default di `arch linux` belum diinstall `gd` so mari install

```
sudo pacman -S php-gd
```

## Installation

```php
composer require maatwebsite/excel
```

## Factory

kita butuh data dummy User.

```
art tinker
```

dan selanjutnya

```php
>> factory(App\User::class, 50)->create()
```

## Export Quick Start

```
art make:export UsersExport --model=User
```

dan didalamnya terdapat sebuah method `collection` dimana disitu kita akan mengambil sebuah data melalui Model

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromCollection;

class UsersExport implements FromCollection
{
    /**
    * @return \Illuminate\Support\Collection
    */
    public function collection()
    {
        return User::select('id', 'name')->where('id', '>', 25)->get();
    }
}
```

yap teman2 bisa mencustom querynya seperti apapun.

lalu kita akan membuat sebuah controller untuk mengexport data `collection`

```
art make:controller UserController -i
```

dan didalamnya buat sebuah method `__invoke` yang di generate menggunakan parameter `-i`.

```php
<?php

namespace App\Http\Controllers;

use App\Exports\UsersExport;
use Maatwebsite\Excel\Facades\Excel;

class UserController extends Controller
{
    public function __invoke()
    {
        return Excel::download(new UsersExport, 'users.xlsx');
    }
}
```

lalu bagian route kita akan tambahkan.

```php
Route::get('users/export', 'UserController');
```

dan coba buka urlnya.

```
http://laravel-excel.art/users/export
```

dan berhasil mengexport data `User` yang akan di export secara otomatis.

## Storing export on the disk

kita akan mencoba untuk mengexport ke dalam storage laravel atau project kita bisa dimanapun seperti `s3`.

kita akan coba storing by default yang mana akan mengexport ke `storage/app/public`.

kita coba tambahkan method `__invoke` menjadi seperti ini.

```php
public function exportOnDisk()
{
    Excel::store(new UsersExport, 'users.xlsx'); // menggunakan store

    return 'tersimpan distorage /storage/app/users.xlsx';
}
```

lalu dibagian route tetap sama karena kita akan gunakan lagi.

```php
Route::get('users/export', 'UserController');
```

> note: perlu diketahui jika filenya namanya sama maka dia akan mereplace data yang sebelumnya sudah ada.

format bisa diganti sesuai yang di inginkan ganti di method static `Excel::` pada `users.xslx` misal jadi `users.cvs` seperti ini

```php
public function __invoke()
{
    Excel::store(new UsersExport, 'users.cvs');

    return 'tersimpan distorage /storage/app/users.cvs';
}
```

jika coba diakses url yang tadi maka bisa kita lihat filenya di `storage/app/users.xlsx`.

## From Query

sama halnya dengan fungsi yang kita buat sebelumnya hanya saja disini dikhususkan jika kita mempunyai big data atau data yang sangat banyak yang nanti diolah oleh chunk, disini from query akan mempersiapkan dibelakang layar agar setelah dieksekusi performancenya akan stabil dan kencang.

perlu diketahui ketika menggunakan `from query` kita tidak dapat menggunakan akhiran query seperti `get()` karena nanti akan lgsg dieksekusi juka melakukan itu sementara `from query` mempersiapkan dibelakang layar.

mari kita kembali kefile `UsersExport` dan tambahkan beberapa.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromCollection;

class UsersExport implements FromCollection
{
    /**
     * @return \Illuminate\Support\Collection
     */
    public function collection()
    {
        return User::select('id', 'name')->where('id', '>', 25)->get();
    }
}
```

kita rubah seperti ini.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;

class UsersExport implements FromQuery
{
    public function query()
    {
        return User::select('id', 'name')->where('id', '>', 25);
    }
}
```

> note: method `query` wajib dan jangan lupa untuk tidak menggunakan akhiran `get` atapun lainnya seperti `first`

jika kita coba url yang sama maka hasilnya akan sama saja yang membedakan seperti yang sudah disebutkan dari awal.

## Mapping Data

sekarang kita akan belajar mapping data dimana kita akan mengcustom sebuah field mana saja yang akan ditampilkan menggunakan method `map`.

pada bagian `UsersExport` kita akan menambahkan `WithMapping` sebelum itu mungkin kita perlu merubah controller kita menggunakan static method `download` bukan `store` agar hasilnya terlihat jelas dan bisa langsung open file.

```php
<?php

namespace App\Http\Controllers;

use App\Exports\UsersExport;
use Maatwebsite\Excel\Facades\Excel;

class UserController extends Controller
{
    public function __invoke()
    {
        // ubah menjadi download
        return Excel::download(new UsersExport, 'users.xlsx');
    }
}
```

oke kita kembali ke `UsersExport`, kita masih tetap memakai `query` ataupun `collection` pilih salah satu jgn keduanya karena akan mereturn 2 data sekaligus.

jadi kita menggunakan query yang ada di method `query` dari implements `FromQury` dan dibawah method `query` kita buat sebuah method `map` dibawah `query`

```php
public function map($user): array
{
    return [
        'ini id ke - ' . $user->id,
        $user->name,
        $user->email,
        Date::dateTimeToExcel($user->created_at)
    ];
}
```

karena kita menampilkan data `id`, `name`, `email` dan `date time` kita butuh data tsb sedangkan query kita di method `query` saya meng `select('id', 'name')` kita akan mengambil semua data jadi kita akan merubahnya seperti ini,

```php
public function query()
{
    return User::where('id', '>', 25);
}
```

urlnya tetap sama oke kita coba maka akan mengexport otomatis dan jika kita lihat di `excel` maka akan tampil sesuai yang kita `mapping`.

## Mapping data (Multiple rows)

jika ingin beberapa misalkan saja didalam 1 baris terdapat 2 kolom lalu di baris ke 2 terdapat 1 kolom
dan baris ke 3 juga 1 kolom nah kita bisa memanfaatkan `multiple rows` caranya sama seperti mapping data hanya saja dibagian return `array` di buat multi dimensi seperti ini.

```php
public function map($user): array
{
    return [
        [
            // menampilkan 2 kolom dalam 1 baris
            'ini id ke - ' . $user->id,
            $user->name
        ],
        [
            // lalu baris ke 2 menjadi 1 kolom saja
            $user->email,
        ],
        [
            // lalu baris ke 3 menjadi 1 kolom saja
            Date::dateTimeToExcel($user->created_at)
        ]
    ];
}
```

## Mapping data (Adding a heading row)

nah jika ingin menampilkan heading maka kita bisa menggunakan method tambahan untuk `heading`. Tambahkan setelah method `map`

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithHeadings; // import WithHeadings
use Maatwebsite\Excel\Concerns\WithMapping;
use PhpOffice\PhpSpreadsheet\Shared\Date;

// tambahkan WithHeadings
class UsersExport implements FromQuery, WithMapping, WithHeadings {

......

    public function headings(): array
    {
        return [
            '#',
            'Name',
            'Email',
            'Time'
        ];
    }

}
```

kita coba terlebih dahulu mapping data biasa bukan `multi rows` kita rubah dulu seperti ini method `map`

```php
public function map($user): array
{
    return [
        'ini id ke - ' . $user->id,
        $user->name,
        $user->email,
        Date::dateTimeToExcel($user->created_at)
    ];
}
```

dan hasilnya akan ditambahkan headingnya seperti ini hasilnya.

|       #        |     Name      |           Email            |    Time    |
| :------------: | :-----------: | :------------------------: | :--------: |
| ini id ke - 26 | Marcos Cremin | fisher.camilla@example.com | 44035.3405 |

jika multiple bisa baca sendiri di sini.

https://docs.laravel-excel.com/3.1/exports/mapping.html#adding-a-heading-row

## Formatting Columns

Nah sekarang kita akan mencoba untuk memanipulasi atau memformat column yang berinteraksi langsung oleh excel dimana kasusnya misalkan kita mempunyai data di `C` merupakan sebuah `timestamp` menjadi `format` yaitu `Day Mont Year` atau `DDMMYYYY` dan kasus lainnya seperti column ke `B` mempunyai sebuah total `invoice` sebesar `1000000` maka kita bisa memformatnya menjadi rupiah menggunakan `formatting columns` ini.

> note: When working with dates, it's recommended to use `\PhpOffice\PhpSpreadsheet\Shared\Date::dateTimeToExcel()` in your mapping to ensure correct parsing of dates.

kita akan kembali ke `UsersExport`.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use PhpOffice\PhpSpreadsheet\Shared\Date;

class UsersExport implements FromQuery, WithMapping, WithHeadings
{
    public function query()
    {
        return User::where('id', '>', 25);
    }

    public function map($user): array
    {
        return [
            'ini id ke - ' . $user->id,
            $user->name,
            $user->email,
            Date::dateTimeToExcel($user->created_at)
        ];
    }

    public function headings(): array
    {
        return [
            '#',
            'Name',
            'Email',
            'Time'
        ];
    }
}

```

jika kita perhatikan terdapat 4 field yaitu di method heading ada `#` atau `id` lalu `name`, `email`, `time` nah jika disimulasikan datanya diexport ke xlsx maka terdapat `A`, `B`, `C`, `D` dimana column `D` adalah `time` kita akan mencoba memformatnya menggunakan `Formatting columns` langsung saja kita tambahkan implements `WithColumnFormatting`.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithColumnFormatting; // Import
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use PhpOffice\PhpSpreadsheet\Shared\Date;
use PhpOffice\PhpSpreadsheet\Style\NumberFormat;

class UsersExport implements FromQuery, WithMapping, WithHeadings, WithColumnFormatting // implement
{
    public function query()
    {
        return User::where('id', '>', 25);
    }

    public function map($user): array
    {
        return [
            'ini id ke - ' . $user->id,
            $user->name,
            $user->email,
            Date::dateTimeToExcel($user->created_at)
        ];
    }

    public function headings(): array
    {
        return [
            '#',
            'Name',
            'Email',
            'Time'
        ];
    }

    // tambahkan disini
    public function columnFormats(): array
    {
        return [
            // formatting column D ke format DDMMYYYY
            'D' => NumberFormat::FORMAT_DATE_DDMMYYYY
        ];
    }
}
```

kita coba lagi melalui url yang sama dan hasilnya column `D` pada excel akan berformat `Day Mont Year`

|       #        |     Name      |           Email            |    Time    |
| :------------: | :-----------: | :------------------------: | :--------: |
| ini id ke - 26 | Marcos Cremin | fisher.camilla@example.com | 23/07/2020 |

jika ingin supaya rapi di excel agar auto expand with gunakan implement `ShouldAutoSize` dan import `use Maatwebsite\Excel\Concerns\ShouldAutoSize;`.

##  Multiple Sheets

Untuk membuat sebuah multiple sheet sepanjang tabs di bawah `sheet` atau `worksheet` kita akan menggunakan implement `WithMultipleSheets`.

pertama mungkin kita akan cleankan terlebih dahulu yang ada di UsersExport.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\Exportable;
use Maatwebsite\Excel\Concerns\WithMultipleSheets;

class UsersExport implements WithMultipleSheets
{
    use Exportable;

    public function sheets(): array
    {
        $sheets = [];

        for ($month = 1; $month <= 3; $month++) {
            $sheets[] = new UsersPerMonthSheet();
        }

        return $sheets;
    }
}
```

jadi disini kita mengimport beberapa yaitu implement dari `WithMultipleSheets` lalu kita buat method `sheets` dan kita melakukan iterasi yang mana berapa buah `sheet` yang dibutuhkan disini saya membuatnya menjadi `3` dan memanggil instance `new UsersPerMonthSheet` yang akan kita buat nanti.

lalu kita akan membuat file `UsersPerMonthSheet.php` dengan namespace yang sama.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithTitle;

class UsersPerMonthSheet implements FromQuery, WithTitle
{
    /**
     * @return Builder
     */
    public function query()
    {
        return User
            ::query()
            ->where('id', '>', 40);
    }

    /**
     * @return string
     */
    public function title(): string
    {
        return 'Month';
    }
}
```

okeh jika kita lakukan hit kembali pada endpoint maka hasilnya akan ada 3 halaman dengan nama `Month` karena di titlenya kita buat seperti itu lalu dengan query yang sama.

## Extending

- Events

Bagaiman kita bisa sesudah di export maka heading yang ada akan menjadi bold? atau menjumlahkan total sebuah field, nah kita akan memanfaatkan event menggunakan `WithEvent`

kita akan mencoba bagaimana membuat sebuah headings menjadi bold ataupun menjumlahkan semua `point` yang ada dimana kita belum menambahkannya.

```
$ art make:migration add_point_to_users_table --table=users
```

selanjutnya kita menambahkan integer point dan meletakkannya setelah `name`.

```php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->integer('point')->after('name');
    });
}
```

lalu untuk `down`.

```php
public function down()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('point');
    });
}
```

selanjutnya kita perlu merubah `factory` untuk membuat data dummy. By default `factory user` sudah ada silahkan buka file `UserFactory` pada `database/factories/`.

```php
$factory->define(User::class, function (Faker $faker) {
    return [
        'name' => $faker->name,
        // tambah disini kita buat angka random
        'point' => rand(1, 100),
        'email' => $faker->unique()->safeEmail,
        'email_verified_at' => now(),
        'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
        'remember_token' => Str::random(10),
    ];
});
```

lalu kita panggil factory di seeder agar lebih mudah memanggil factorynya.

buka bagian `DatabaseSeeder` selevel dengan `factories` folder.

```php
use App\User;

......

public function run()
{
    factory(User::class, 10)->create();
    // $this->call(UserSeeder::class);
}
```

lalu migrate

```
$ art migration:fresh --seed
```

maka data dummy akan dibuat 10 buah.

kita akan menggunakan yang sudah ada sebelumnya pada `UserPerMonthSheet`.

```php
<?php

namespace App\Exports;  
use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithColumnFormatting;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use PhpOffice\PhpSpreadsheet\Shared\Date;
use PhpOffice\PhpSpreadsheet\Style\NumberFormat;  
class UsersExport implements FromQuery, WithMapping, WithHeadings, WithColumnFormatting
{
    public function query()
    {
        return User::where('id', '>', 25);
    } 

    public function map($user): array
    {
        return [
            'ini id ke - ' . $user->id,
            $user->name,
            $user->email,
            Date::dateTimeToExcel($user->created_at)
        ];
    }  

    public function headings(): array
    {
        return [
            '#',
            'Name',
            'Email',
            'Time'
        ];
    }  

    public function columnFormats(): array
    {
        return [
            'D' => NumberFormat::FORMAT_DATE_DDMMYYYY
        ];
    }
}
```

kita rubah menjadi seperti ini.

```php
<?php

namespace App\Exports;

use App\User;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithEvents; // import dan gunakan di implement
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithTitle;
use Maatwebsite\Excel\Events\AfterSheet;

class UsersPerMonthSheet implements FromQuery, WithTitle, WithHeadings, WithEvents
{
    /**
    * @return Builder
    */
    public function query()
    {
        return User
            ::query();
    }

    /**
    * @return string
    */
    public function title(): string
    {
        return 'Month '; // nama sheet yang sebelumnya digunakan pada WithMultipleSheet.
    }

    public function headings(): array
    {
        // heading yang akan digunakan nantinya untuk dibuat bold
        return [
            'id',
            'name',
            'point',
            'email',
            'email verification',
            'update_at',
            'created_at'
        ];
    }

    public function registerEvents(): array
    {
        // style yang akan dilakukan event
        $styleArray = [
            'font' => [
                'bold' => true
            ]
        ];

        return [
            // sesudah sheet ada jalankan sebuah fungsi
            AfterSheet::class => function (AfterSheet $event) use ($styleArray) {
                /* membuat bold pada baris pertama sepanjang 
                * A sampai G dan mengapply style yang kita buat */
                $event->sheet->getStyle('A1:G1')->applyFromArray($styleArray);
            }
        ];
    }
}
```

`docs` untuk fungsi2 yang bisa dimanipulasi bisa baca disini.

https://phpspreadsheet.readthedocs.io/en/latest

untuk `UsersExport` tidak perlu dirubah karena kita akan menggunakan yang sebelumnya yaitu `WithMultipleSheet`.

jika kita coba maka bold pada headingnya.

lalu kita mempunyai data 11 termasuk heading jadi pada column `C` atau pada heading `point`
kita akan menjumlahkan seluruhnya dan menaruhnya dibawah tepat pada baris ke 12 di column `point`

kita bisa menambahkan `setCellValue` di `AfterSheet`.

```php
public function registerEvents(): array
{
    // style yang akan dilakukan event
    $styleArray = [
        'font' => [
            'bold' => true
        ]
    ];

    return [
        // sesudah sheet ada jalankan sebuah fungsi
        AfterSheet::class => function (AfterSheet $event) use ($styleArray) {
            /* membuat bold pada baris pertama sepanjang 
            * A sampai G dan mengapply style yang kita buat */
            $event->sheet->getStyle('A1:G1')->applyFromArray($styleArray);
            // tambah disini
            $event->sheet->setCellValue(
            /* nilai yang akan diset */ 'C12', 
            /* penjumlahan */ '=SUM(C2:C11)'
            )
        }
    ];
}
```

maka pada `C` ke baris `12` akan mengenerate nilai otomatis dari `C2` sampai `C11`.

kita juga bisa menambahkan `column` ataupun `row` kosong.

kita akan mencoba merubah event yg sudah dibuat.

```php
public function registerEvents(): array
{
    return [
        AfterSheet::class => function (AfterSheet $event) {
           $event->sheet->insertNewRowBefore(/* line ke 7 */ 7, /* row yang mau ditambah */ 2);
           $event->sheet->insertNewColumnBefore('A', /* row yang mau ditambah */ 2);
        }
    ];
}
```

maka akan dibuat sebuah row dan column baru.

ada juga yang lain seperti `BeforeSheet` dimana akan menambahkan value baru di baris pertama misalnya.

silahkan diexplore.

```php
public function registerEvents(): array
{
    return [
       BeforeSheet::class => function (AfterSheet $event) {
           $event->sheet->setCellValue('A1', 'Hello');
       }
    ];
}
```

jangan lupa diimport `BeforeSheet`nya.

## Contact

- email `->` kevariable@gmail.com
- telegram `->` https://t.me/kevariable
- github `->` https://github.com/kevariable
- twitter `->` https://twitter.com/kevariable
