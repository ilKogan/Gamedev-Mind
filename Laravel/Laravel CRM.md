Я понимаю ваше беспокойство. Давайте разберем задачу пошагово - это сделает её менее пугающей. Я помогу вам понять структуру и реализовать необходимый функционал.

Начнем с самого основного - создания базовой структуры проекта и системы ролей.
```php
<?php

// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    protected $fillable = [
        'first_name',
        'last_name',
        'email',
        'password',
        'role',
        'is_verified',
        'status',
        'date_of_birth',
        'address',
        'city',
        'region',
        'zip_code',
        'country'
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'is_verified' => 'boolean',
        'email_verified_at' => 'datetime',
    ];

    public function bankAccount()
    {
        return $this->hasOne(BankAccount::class);
    }

    public function customFields()
    {
        return $this->hasMany(UserCustomField::class);
    }

    public function isSuperAdmin()
    {
        return $this->role === 'superadmin';
    }

    public function isAdmin()
    {
        return $this->role === 'admin';
    }
}
```

```php
// app/Models/BankAccount.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class BankAccount extends Model
{
    protected $fillable = [
        'user_id',
        'bank_name',
        'swift_code',
        'account_number',
        'is_confirmed'
    ];

    protected $casts = [
        'is_confirmed' => 'boolean',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

```php
// app/Models/UserCustomField.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class UserCustomField extends Model
{
    protected $fillable = [
        'user_id',
        'field_name',
        'field_value'
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

```
Теперь давайте создадим необходимые миграции:

```php
<?php

// database/migrations/xxxx_xx_xx_create_users_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateUsersTable extends Migration
{
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('first_name')->nullable();
            $table->string('last_name')->nullable();
            $table->string('email')->unique();
            $table->string('password');
            $table->enum('role', ['superadmin', 'admin', 'user'])->default('user');
            $table->boolean('is_verified')->default(false);
            $table->enum('status', ['enabled', 'disabled'])->default('enabled');
            $table->date('date_of_birth')->nullable();
            $table->string('address')->nullable();
            $table->string('city')->nullable();
            $table->string('region')->nullable();
            $table->string('zip_code')->nullable();
            $table->string('country')->nullable();
            $table->timestamp('email_verified_at')->nullable();
            $table->rememberToken();
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('users');
    }
}
```

```php
// database/migrations/xxxx_xx_xx_create_bank_accounts_table.php
class CreateBankAccountsTable extends Migration
{
    public function up()
    {
        Schema::create('bank_accounts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('bank_name');
            $table->string('swift_code');
            $table->string('account_number');
            $table->boolean('is_confirmed')->default(false);
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('bank_accounts');
    }
}
```

```php
// database/migrations/xxxx_xx_xx_create_user_custom_fields_table.php
class CreateUserCustomFieldsTable extends Migration
{
    public function up()
    {
        Schema::create('user_custom_fields', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('field_name');
            $table->text('field_value');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('user_custom_fields');
    }
}
```
Это базовая структура. Теперь давайте я объясню, как это работает и что делать дальше:

1. Сначала установите новый проект Laravel:
```bash
composer create-project laravel/laravel your-project-name
cd your-project-name
```

2. Скопируйте код миграций в соответствующие файлы в папке `database/migrations/`

3. Скопируйте модели в папку `app/Models/`

4. Запустите миграции:
```bash
php artisan migrate
```

Дальше мы можем:
1. Настроить аутентификацию с email-верификацией
2. Создать контроллеры для управления профилем
3. Реализовать админ-панель

Отлично. Давайте настроим аутентификацию и верификацию email.

```php
<?php

// app/Http/Controllers/Auth/RegisterController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use App\Notifications\VerifyEmailNotification;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class RegisterController extends Controller
{
    public function register(Request $request)
    {
        $request->validate([
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
        ]);

        $user = User::create([
            'email' => $request->email,
            'password' => Hash::make($request->password),
            'first_name' => $request->first_name,
            'last_name' => $request->last_name,
            'verification_code' => Str::random(6),
        ]);

        $user->notify(new VerifyEmailNotification($user->verification_code));

        return response()->json([
            'message' => 'Registration successful. Please check your email for verification code.',
        ]);
    }

    public function verify(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'verification_code' => 'required|string',
        ]);

        $user = User::where('email', $request->email)
            ->where('verification_code', $request->verification_code)
            ->first();

        if (!$user) {
            return response()->json([
                'message' => 'Invalid verification code',
            ], 400);
        }

        $user->update([
            'is_verified' => true,
            'email_verified_at' => now(),
            'verification_code' => null,
        ]);

        return response()->json([
            'message' => 'Email verified successfully',
        ]);
    }
}
```


```php
// app/Notifications/VerifyEmailNotification.php
namespace App\Notifications;

use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;

class VerifyEmailNotification extends Notification
{
    private $verificationCode;

    public function __construct($verificationCode)
    {
        $this->verificationCode = $verificationCode;
    }

    public function via($notifiable)
    {
        return ['mail'];
    }

    public function toMail($notifiable)
    {
        return (new MailMessage)
            ->subject('Verify Your Email Address')
            ->line('Your verification code is: ' . $this->verificationCode)
            ->line('Please enter this code on the website to verify your email address.')
            ->line('If you did not create an account, no further action is required.');
    }
}
```

```php
// app/Http/Middleware/EnsureEmailIsVerified.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class EnsureEmailIsVerified
{
    public function handle(Request $request, Closure $next)
    {
        if (!$request->user() || !$request->user()->is_verified) {
            return response()->json([
                'message' => 'Your email address is not verified.',
            ], 403);
        }

        return $next($request);
    }
}

Теперь добавим маршруты в `routes/api.php`:

```php
Route::post('/register', [RegisterController::class, 'register']);
Route::post('/verify-email', [RegisterController::class, 'verify']);
```

Чтобы это заработало, нужно:

1. Добавить поле `verification_code` в миграцию users:
```php
$table->string('verification_code')->nullable();
```

2. Добавить его в `$fillable` в модели User:
```php
protected $fillable = [
    // ... существующие поля ...
    'verification_code',
];
```

1. Настроить отправку email в файле `.env`:
```c
MAIL_MAILER=smtp
MAIL_HOST=ваш.smtp.сервер
MAIL_PORT=587
MAIL_USERNAME=ваш_email
MAIL_PASSWORD=ваш_пароль
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=ваш_email
MAIL_FROM_NAME="${APP_NAME}"
```

2. Если вы хотите защитить какие-то маршруты и требовать верификацию email, добавьте middleware в `app/Http/Kernel.php`:
```php
protected $routeMiddleware = [
    // ... другие middleware ...
    'verified' => \App\Http\Middleware\EnsureEmailIsVerified::class,
];
```

И используйте его в маршрутах:
```php
Route::middleware(['auth', 'verified'])->group(function () {
    // защищенные маршруты
});
```

Теперь процесс регистрации работает так:
3. Пользователь регистрируется с email и паролем
4. Система генерирует случайный код верификации
5. Код отправляется на email пользователя
6. Пользователь вводит код на сайте
7. После успешной верификации статус пользователя обновляется

Давайте реализуем систему ролей и базовую админ-панель.
```php
<?php

// app/Http/Middleware/CheckRole.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckRole
{
    public function handle(Request $request, Closure $next, ...$roles)
    {
        if (!$request->user() || !in_array($request->user()->role, $roles)) {
            abort(403, 'Unauthorized action.');
        }

        return $next($request);
    }
}
```

```php
// app/Http/Controllers/Admin/UserController.php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        $users = User::paginate(10);
        return view('admin.users.index', compact('users'));
    }

    public function edit(User $user)
    {
        return view('admin.users.edit', compact('user'));
    }

    public function update(Request $request, User $user)
    {
        $data = $request->validate([
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email,' . $user->id,
            'status' => 'required|in:enabled,disabled',
            'is_verified' => 'boolean'
        ]);
        // Только superadmin может менять роли
        if (auth()->user()->isSuperAdmin()) {
            $data['role'] = $request->validate([
                'role' => 'required|in:admin,user'
            ])['role'];
        }

        $user->update($data);

        return redirect()->route('admin.users.index')
            ->with('success', 'User updated successfully');
    }

    public function destroy(User $user)
    {
        if ($user->id === auth()->id()) {
            return back()->with('error', 'You cannot delete your own account');
        }

        $user->delete();
        return back()->with('success', 'User deleted successfully');
    }
}
```

```php
// app/Http/Controllers/Admin/CustomFieldController.php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\UserCustomField;
use Illuminate\Http\Request;

class CustomFieldController extends Controller
{
    public function store(Request $request)
    {
        $data = $request->validate([
            'user_id' => 'required|exists:users,id',
            'field_name' => 'required|string|max:255',
            'field_value' => 'required|string'
        ]);

        UserCustomField::create($data);

        return back()->with('success', 'Custom field added successfully');
    }

    public function destroy(UserCustomField $field)
    {
        $field->delete();
        return back()->with('success', 'Custom field deleted successfully');
    }
}

А теперь создадим базовые представления для админ-панели:

<!-- resources/views/admin/users/index.blade.php -->
@extends('layouts.admin')

@section('content')
<div class="container mx-auto px-4">
    <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold">Users Management</h1>
    </div>

    <div class="bg-white shadow-md rounded-lg overflow-hidden">
        <table class="min-w-full leading-normal">
            <thead>
                <tr>
                    <th class="px-5 py-3 border-b-2 border-gray-200 bg-gray-100 text-left text-xs font-semibold text-gray-600 uppercase tracking-wider">
                        Name
                    </th>
                    <th class="px-5 py-3 border-b-2 border-gray-200 bg-gray-100 text-left text-xs font-semibold text-gray-600 uppercase tracking-wider">
                        Email
                    </th>
                    <th class="px-5 py-3 border-b-2 border-gray-200 bg-gray-100 text-left text-xs font-semibold text-gray-600 uppercase tracking-wider">
                        Role
                    </th>
                    <th class="px-5 py-3 border-b-2 border-gray-200 bg-gray-100 text-left text-xs font-semibold text-gray-600 uppercase tracking-wider">
                        Status
                    </th>
                    <th class="px-5 py-3 border-b-2 border-gray-200 bg-gray-100 text-left text-xs font-semibold text-gray-600 uppercase tracking-wider">
                        Actions
                    </th>
                </tr>
            </thead>
            <tbody>
                @foreach($users as $user)
                <tr>
                    <td class="px-5 py-5 border-b border-gray-200 bg-white text-sm">
                        {{ $user->first_name }} {{ $user->last_name }}
                    </td>
                    <td class="px-5 py-5 border-b border-gray-200 bg-white text-sm">
                        {{ $user->email }}
                    </td>
                    <td class="px-5 py-5 border-b border-gray-200 bg-white text-sm">
                        {{ ucfirst($user->role) }}
                    </td>
                    <td class="px-5 py-5 border-b border-gray-200 bg-white text-sm">
                        <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full {{ $user->status === 'enabled' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800' }}">
                            {{ $user->status }}
                        </span>
                    </td>
                    <td class="px-5 py-5 border-b border-gray-200 bg-white text-sm">
                        <a href="{{ route('admin.users.edit', $user) }}" class="text-indigo-600 hover:text-indigo-900">Edit</a>
                        
                        @if(auth()->user()->isSuperAdmin() && auth()->id() !== $user->id)
                        <form action="{{ route('admin.users.destroy', $user) }}" method="POST" class="inline">
                            @csrf
                            @method('DELETE')
                            <button type="submit" class="ml-2 text-red-600 hover:text-red-900">Delete</button>
                        </form>
                        @endif
                    </td>
                </tr>
                @endforeach
            </tbody>
        </table>
        <div class="px-5 py-5 bg-white border-t flex flex-col xs:flex-row items-center xs:justify-between">
            {{ $users->links() }}
        </div>
    </div>
</div>
@endsection
```


Добавьте следующие маршруты в `routes/web.php`:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    // Маршруты для админ-панели
    Route::middleware(['check.role:superadmin,admin'])->prefix('admin')->name('admin.')->group(function () {
        Route::resource('users', UserController::class);
        Route::resource('custom-fields', CustomFieldController::class)->only(['store', 'destroy']);
    });
});
```

Также добавьте middleware в `app/Http/Kernel.php`:

```php
protected $routeMiddleware = [
    // ... другие middleware ...
    'check.role' => \App\Http\Middleware\CheckRole::class,
];
```

Теперь у вас есть:
1. Система ролей с middleware для проверки прав доступа
2. Контроллеры для управления пользователями и кастомными полями
3. Базовый интерфейс админ-панели со списком пользователей

Отлично. Создадим контроллеры и представления для управления профилем и банковскими данными.

```php
<?php

// app/Http/Controllers/ProfileController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests\UpdateProfileRequest;
use Illuminate\Support\Facades\Hash;

class ProfileController extends Controller
{
    public function show()
    {
        $user = auth()->user();
        return view('profile.show', compact('user'));
    }

    public function update(UpdateProfileRequest $request)
    {
        $user = auth()->user();
        
        $data = $request->validated();
        
        // Если пароль не предоставлен, удаляем его из данных
        if (empty($data['password'])) {
            unset($data['password']);
        } else {
            $data['password'] = Hash::make($data['password']);
        }

        $user->update($data);

        // Обновляем кастомные поля
        if (!empty($data['custom_fields'])) {
            foreach ($data['custom_fields'] as $field_name => $field_value) {
                $user->customFields()->updateOrCreate(
                    ['field_name' => $field_name],
                    ['field_value' => $field_value]
                );
            }
        }

        return redirect()->route('profile.show')
            ->with('success', 'Profile updated successfully');
    }
}
```

```php
// app/Http/Controllers/BankAccountController.php
namespace App\Http\Controllers;

use App\Models\BankAccount;
use Illuminate\Http\Request;
use App\Http\Requests\BankAccountRequest;

class BankAccountController extends Controller
{
    public function store(BankAccountRequest $request)
    {
        $user = auth()->user();
        
        // Создаем или обновляем банковский аккаунт
        $user->bankAccount()->updateOrCreate(
            ['user_id' => $user->id],
            $request->validated()
        );

        return redirect()->route('profile.show')
            ->with('success', 'Bank account information updated successfully');
    }

    public function destroy()
    {
        $user = auth()->user();
        $user->bankAccount()->delete();

        return redirect()->route('profile.show')
            ->with('success', 'Bank account information removed');
    }
}
```

```php
// app/Http/Requests/UpdateProfileRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UpdateProfileRequest extends FormRequest
{
    public function rules()
    {
        return [
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
            'date_of_birth' => 'nullable|date',
            'address' => 'nullable|string|max:255',
            'city' => 'nullable|string|max:255',
            'region' => 'nullable|string|max:255',
            'zip_code' => 'nullable|string|max:20',
            'country' => 'nullable|string|max:255',
            'password' => 'nullable|min:8|confirmed',
            'custom_fields.*' => 'nullable|string|max:255',
        ];
    }
}
```

```php
// app/Http/Requests/BankAccountRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class BankAccountRequest extends FormRequest
{
    public function rules()
    {
        return [
            'bank_name' => 'required|string|max:255',
            'swift_code' => 'required|string|max:11',
            'account_number' => 'required|string|max:50',
            'is_confirmed' => 'required|boolean',
        ];
    }
}
```

```html
Теперь создадим представления для профиля:

<!-- resources/views/profile/show.blade.php -->
@extends('layouts.app')

@section('content')
<div class="container mx-auto px-4 py-8">
    <div class="max-w-3xl mx-auto">
        @if (session('success'))
            <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative mb-4">
                {{ session('success') }}
            </div>
        @endif

        <!-- Профиль -->
        <div class="bg-white shadow rounded-lg mb-6">
            <div class="px-6 py-4 border-b border-gray-200">
                <h2 class="text-lg font-medium text-gray-900">Profile Information</h2>
            </div>
            
            <form action="{{ route('profile.update') }}" method="POST" class="p-6">
                @csrf
                @method('PUT')

                <div class="grid grid-cols-2 gap-6">
                    <div>
                        <label class="block text-sm font-medium text-gray-700">First Name</label>
                        <input type="text" name="first_name" value="{{ old('first_name', $user->first_name) }}" 
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Last Name</label>
                        <input type="text" name="last_name" value="{{ old('last_name', $user->last_name) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Date of Birth</label>
                        <input type="date" name="date_of_birth" value="{{ old('date_of_birth', $user->date_of_birth) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Email</label>
                        <input type="email" value="{{ $user->email }}" disabled
                               class="mt-1 block w-full rounded-md border-gray-300 bg-gray-50 shadow-sm">
                    </div>
                </div>

                <div class="mt-6">
                    <label class="block text-sm font-medium text-gray-700">Address</label>
                    <input type="text" name="address" value="{{ old('address', $user->address) }}"
                           class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                </div>

                <div class="grid grid-cols-2 gap-6 mt-6">
                    <div>
                        <label class="block text-sm font-medium text-gray-700">City</label>
                        <input type="text" name="city" value="{{ old('city', $user->city) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Region</label>
                        <input type="text" name="region" value="{{ old('region', $user->region) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Zip Code</label>
                        <input type="text" name="zip_code" value="{{ old('zip_code', $user->zip_code) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Country</label>
                        <input type="text" name="country" value="{{ old('country', $user->country) }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>
                </div>

                <div class="mt-6">
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700">
                        Update Profile
                    </button>
                </div>
            </form>
        </div>

        <!-- Банковская информация -->
        <div class="bg-white shadow rounded-lg">
            <div class="px-6 py-4 border-b border-gray-200">
                <h2 class="text-lg font-medium text-gray-900">Bank Account Information</h2>
            </div>
            
            <form action="{{ route('bank-account.store') }}" method="POST" class="p-6">
                @csrf

                <div class="grid grid-cols-2 gap-6">
                    <div>
                        <label class="block text-sm font-medium text-gray-700">Bank Name</label>
                        <input type="text" name="bank_name" value="{{ old('bank_name', $user->bankAccount->bank_name ?? '') }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">SWIFT Code / BIC</label>
                        <input type="text" name="swift_code" value="{{ old('swift_code', $user->bankAccount->swift_code ?? '') }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>

                    <div class="col-span-2">
                        <label class="block text-sm font-medium text-gray-700">Account Number</label>
                        <input type="text" name="account_number" value="{{ old('account_number', $user->bankAccount->account_number ?? '') }}"
                               class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
                    </div>
                </div>

                <div class="mt-6">
                    <label class="flex items-center">
                        <input type="checkbox" name="is_confirmed" value="1" 
                               {{ old('is_confirmed', $user->bankAccount->is_confirmed ?? false) ? 'checked' : '' }}
                               class="rounded border-gray-300 text-blue-600 shadow-sm">
                        <span class="ml-2 text-sm text-gray-600">I confirm the bank account details above</span>
                    </label>
                </div>

                <div class="mt-6 flex items-center justify-between">
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700">
                        Save Bank Information
                    </button>

                    @if($user->bankAccount)
                        <form action="{{ route('bank-account.destroy') }}" method="POST" class="inline">
                            @csrf
                            @method('DELETE')
                            <button type="submit" class="text-red-600 hover:text-red-900">
                                Remove Bank Information
                            </button>
                        </form>
                    @endif
                </div>
            </form>
        </div>
    </div>
</div>
@endsection
```


Добавьте следующие маршруты в `routes/web.php`:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/profile', [ProfileController::class, 'show'])->name('profile.show');
    Route::put('/profile', [ProfileController::class, 'update'])->name('profile.update');
    
    Route::post('/bank-account', [BankAccountController::class, 'store'])->name('bank-account.store');
    Route::delete('/bank-account', [BankAccountController::class, 'destroy'])->name('bank-account.destroy');
});
```

Теперь у вас есть полноценная система управления профилем и банковскими данными с:
1. Валидацией всех входящих данных
2. Безопасным сохранением банковской информации
3. Возможностью обновления и удаления данных
4. Красивым и удобным интерфейсом
5. Поддержкой кастомных полей

Отлично. Давайте разберем процесс развертывания Laravel проекта на сервере.

# Руководство по развертыванию Laravel проекта

## 1. Подготовка сервера

### Установка необходимого ПО
```bash
# Обновление пакетов
sudo apt update
sudo apt upgrade -y

# Установка необходимых пакетов
sudo apt install -y nginx mysql-server php8.1-fpm php8.1-cli php8.1-mysql \
php8.1-mbstring php8.1-xml php8.1-curl php8.1-zip php8.1-gd \
composer git unzip

# Настройка MySQL
sudo mysql_secure_installation
```

### Создание базы данных
```bash
mysql -u root -p
CREATE DATABASE your_database;
CREATE USER 'your_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON your_database.* TO 'your_user'@'localhost';
FLUSH PRIVILEGES;
exit;
```

### Настройка Nginx
```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/your-project/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## 2. Развертывание проекта

### Клонирование и настройка проекта
```bash
# Создание директории и настройка прав
sudo mkdir -p /var/www/your-project
sudo chown -R $USER:www-data /var/www/your-project

# Клонирование репозитория
git clone your-repository.git /var/www/your-project

# Переход в директорию проекта
cd /var/www/your-project

# Установка зависимостей
composer install --no-dev --optimize-autoloader

# Копирование и настройка .env
cp .env.example .env
php artisan key:generate

# Настройка прав доступа
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

### Настройка .env файла
```env
APP_NAME=YourApp
APP_ENV=production
APP_KEY=base64:your-key
APP_DEBUG=false
APP_URL=https://your-domain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=your_database
DB_USERNAME=your_user
DB_PASSWORD=your_password

MAIL_MAILER=smtp
MAIL_HOST=your-smtp-server
MAIL_PORT=587
MAIL_USERNAME=your-email
MAIL_PASSWORD=your-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=your-email
MAIL_FROM_NAME="${APP_NAME}"
```

### Оптимизация Laravel
```bash
# Очистка и кэширование конфигурации
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Запуск миграций
php artisan migrate --force

# Создание символической ссылки для storage
php artisan storage:link
```

## 3. Настройка SSL (Let's Encrypt)
```bash
# Установка Certbot
sudo apt install -y certbot python3-certbot-nginx

# Получение SSL сертификата
sudo certbot --nginx -d your-domain.com

# Автоматическое обновление сертификата
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

## 4. Настройка Supervisor для очередей Laravel
```bash
# Установка Supervisor
sudo apt install -y supervisor

# Создание конфигурации для Laravel
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Содержимое конфигурации:
```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/your-project/artisan queue:work
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/your-project/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
# Обновление и запуск Supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

## 5. Настройка автоматического деплоя (опционально)

### Создание деплой-скрипта
```bash
#!/bin/bash

# deploy.sh
cd /var/www/your-project

# Pull latest changes
git pull origin main

# Install dependencies
composer install --no-dev --optimize-autoloader

# Clear caches
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Run migrations
php artisan migrate --force

# Update permissions
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache

# Restart queue workers
sudo supervisorctl restart laravel-worker:*

# Restart PHP-FPM
sudo systemctl restart php8.1-fpm
```

Основные шаги при развертывании:

1. **Подготовка сервера:**
   - Установка LEMP стека (Linux, Nginx, MySQL, PHP)
   - Настройка базы данных
   - Настройка веб-сервера

2. **Развертывание кода:**
   - Клонирование репозитория
   - Установка зависимостей
   - Настройка окружения
   - Запуск миграций

3. **Безопасность:**
   - Настройка SSL
   - Правильные права доступа
   - Отключение отладки

4. **Оптимизация:**
   - Кэширование конфигурации
   - Настройка очередей
   - Оптимизация PHP и MySQL

Хотите, чтобы я подробнее объяснил какой-то из этих этапов или показал дополнительные настройки для улучшения производительности?