Creating a website with Lumen (a micro-framework by Laravel) that contains information about bus travels, including a departure schedule, login page, and payment pages, involves several key steps. I’ll guide you through the process to set up the basic structure, but you can customize and expand it further depending on your needs.

Here's a step-by-step guide on how to do it:

### 1. **Setting up Lumen**

First, install Lumen by following the steps below.

1. Install Composer (if not already installed):  
    [Install Composer](https://getcomposer.org/doc/00-intro.md)

2. Create a new Lumen project:

    ```bash
    composer create-project --prefer-dist laravel/lumen bus-travel-site
    cd bus-travel-site
    ```

3. Configure your `.env` file for database connection (you will likely use MySQL or SQLite for simplicity):

    ```bash
    cp .env.example .env
    ```

4. Configure the database settings in the `.env` file:

    ```env
    DB_CONNECTION=mysql
    DB_HOST=127.0.0.1
    DB_PORT=3306
    DB_DATABASE=bus_travel_db
    DB_USERNAME=root
    DB_PASSWORD=
    ```


### 2. **Database Structure**

You will need several tables to manage users, bus schedules, and payments. Create a migration file for each of them.

- **Users Table:** Store user data (for login/authentication).
- **Bus Schedules Table:** Store bus departure schedules.
- **Payments Table:** Store user payments.

Run migrations to set up these tables.

1. **Users Migration** (authentication data):

    Run:

    ```bash
    php artisan make:migration create_users_table
    ```

    Add the following schema:

    ```php
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->timestamps();
        });
    }
    ```

2. **Bus Schedules Migration:**

    Run:

    ```bash
    php artisan make:migration create_bus_schedules_table
    ```

    Add the following schema:

    ```php
    public function up()
    {
        Schema::create('bus_schedules', function (Blueprint $table) {
            $table->id();
            $table->string('origin');
            $table->string('destination');
            $table->timestamp('departure_time');
            $table->integer('available_seats');
            $table->decimal('price', 8, 2);
            $table->timestamps();
        });
    }
    ```

3. **Payments Migration:**

    Run:

    ```bash
    php artisan make:migration create_payments_table
    ```

    Add the following schema:

    ```php
    public function up()
    {
        Schema::create('payments', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->foreignId('bus_schedule_id')->constrained()->onDelete('cascade');
            $table->decimal('amount', 8, 2);
            $table->timestamps();
        });
    }
    ```


### 3. **Authentication (Login Page)**

Install Laravel’s built-in authentication package.

```bash
composer require laravel/ui
php artisan ui vue --auth
npm install && npm run dev
```

You can then use the default login, register, and password reset functionality out of the box. Adjust the controllers if needed to add custom fields (like profile pictures, etc.).

### 4. **Creating Bus Schedule Page**

Create a route, controller, and view for displaying bus schedules.

1. **Route:**

    In `routes/web.php`, add the following route for the bus schedules:

    ```php
    $router->get('/bus-schedules', 'BusScheduleController@index');
    ```

2. **Controller:**

    Create the controller with:

    ```bash
    php artisan make:controller BusScheduleController
    ```

    In `BusScheduleController.php`:

    ```php
    <?php

    namespace App\Http\Controllers;

    use App\Models\BusSchedule;
    use Illuminate\Http\Request;

    class BusScheduleController extends Controller
    {
        public function index()
        {
            $schedules = BusSchedule::all();
            return response()->json($schedules);
        }
    }
    ```

3. **View:**

    You can use a simple view (e.g., `resources/views/bus-schedules.blade.php`) to display the data. For simplicity, let's return JSON for now, but you can create a Blade template for a more complex view later.

    ```php
    <table>
        <thead>
            <tr>
                <th>Origin</th>
                <th>Destination</th>
                <th>Departure Time</th>
                <th>Available Seats</th>
                <th>Price</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($schedules as $schedule)
                <tr>
                    <td>{{ $schedule->origin }}</td>
                    <td>{{ $schedule->destination }}</td>
                    <td>{{ $schedule->departure_time }}</td>
                    <td>{{ $schedule->available_seats }}</td>
                    <td>{{ $schedule->price }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>
    ```


### 5. **Payment System**

To process payments, you can integrate a payment gateway like Stripe.

1. Install Stripe package via Composer:

    ```bash
    composer require stripe/stripe-php
    ```

2. Set up the Stripe keys in `.env`:

    ```env
    STRIPE_KEY=your-stripe-public-key
    STRIPE_SECRET=your-stripe-secret-key
    ```

3. Create a Payment Controller to handle the payment:

    ```bash
    php artisan make:controller PaymentController
    ```

    In `PaymentController.php`:

    ```php
    use Stripe\Stripe;
    use Stripe\PaymentIntent;

    public function processPayment(Request $request)
    {
        Stripe::setApiKey(env('STRIPE_SECRET'));

        $paymentIntent = PaymentIntent::create([
            'amount' => $request->amount * 100, // Convert to cents
            'currency' => 'usd',
            'payment_method' => $request->payment_method_id,
            'confirmation_method' => 'manual',
            'confirm' => true,
        ]);

        return response()->json($paymentIntent);
    }
    ```

    Add this route in `routes/web.php`:

    ```php
    $router->post('/pay', 'PaymentController@processPayment');
    ```

    Make sure to add a front-end that uses the Stripe JavaScript API to collect the payment information.


### 6. **Frontend & Views**

You can use Vue.js, Blade, or any other front-end framework to build your views. Lumen is primarily for the backend, so you will have to handle the frontend separately.

1. **Bus Schedule View**: Use Blade templates to display bus schedules.
2. **Payment View**: Create a Stripe checkout form using the Stripe.js library to collect payment information.

For example, on the `payment.blade.php` view:

```html
<form id="payment-form">
    <div id="card-element">
        <!-- A Stripe Element will be inserted here. -->
    </div>

    <button id="submit">Pay</button>
</form>

<script src="https://js.stripe.com/v3/"></script>
<script>
    var stripe = Stripe('{{ env("STRIPE_KEY") }}');
    var elements = stripe.elements();
    var card = elements.create("card");
    card.mount("#card-element");

    var form = document.getElementById("payment-form");

    form.addEventListener("submit", async function(event) {
        event.preventDefault();

        const {token, error} = await stripe.createToken(card);
        if (error) {
            console.log(error);
        } else {
            // Send token to server for payment processing
            // You will need to implement an API endpoint on your server to handle this
        }
    });
</script>
```

### 7. **Conclusion**

With these steps, you'll have a basic Lumen-powered website that includes:

- A user login system (with authentication).
- A bus schedule page (displaying bus departures).
- A payment system (using Stripe).

You can then expand this further with additional features like booking, admin management, real-time schedule updates, etc. If you need more specific implementation or help with any step, feel free to ask!

#################################

Certainly! To make your Lumen-based website multilingual (supporting two languages) and to add an "About Us" page, we'll take the following steps:

1. **Add Support for Multiple Languages**
2. **Create an "About Us" Page**
3. **Update Views to Display Content in Multiple Languages**

Let's break down each step:

---

### 1. **Add Support for Multiple Languages**

Lumen supports localization (multi-language) via language files, similar to Laravel. To implement multilingual support for your website, you need to set up language files and load them dynamically based on the user's preferences.

#### Step 1.1: Create Language Files

In the `resources/lang` folder, create separate folders for the languages you want to support (e.g., `en` for English and `es` for Spanish).

For example:

- `resources/lang/en/messages.php` (for English translations)
- `resources/lang/es/messages.php` (for Spanish translations)

In each of these files, you can define translation keys. Here’s how the files might look:

**`resources/lang/en/messages.php`:**

```php
<?php

return [
    'welcome' => 'Welcome to our Bus Travel Service!',
    'about_us' => 'About Us',
    'departure_schedule' => 'Departure Schedule',
    'contact' => 'Contact Us',
    'book_ticket' => 'Book Ticket',
];
```

**`resources/lang/es/messages.php`:**

```php
<?php

return [
    'welcome' => '¡Bienvenido a nuestro servicio de viajes en autobús!',
    'about_us' => 'Sobre Nosotros',
    'departure_schedule' => 'Horario de Salidas',
    'contact' => 'Contáctenos',
    'book_ticket' => 'Reservar Boleto',
];
```

#### Step 1.2: Update `AppServiceProvider` to Set Locale

In your `app/Providers/AppServiceProvider.php`, you can set the locale based on user preferences (for example, via the `Accept-Language` header, a query parameter, or a session variable).

```php
use Illuminate\Support\Facades\App;

public function boot()
{
    $locale = session('locale', 'en');  // Default language is English
    App::setLocale($locale);
}
```

You can set the locale in the session when a user selects a language. For instance, add this route in `routes/web.php` to change the language:

```php
$router->get('/lang/{locale}', function ($locale) {
    session(['locale' => $locale]);
    return redirect()->back();
});
```

This way, you can change the language dynamically by visiting `/lang/en` or `/lang/es`.

#### Step 1.3: Using Translations in Views

Now you can use the `__()` helper function in your views to display the text in the selected language.

For example, in your Blade views, use:

```php
<h1>{{ __('messages.welcome') }}</h1>
```

This will automatically display the English or Spanish text based on the selected language.

#### Step 1.4: Creating Language Switcher

You can add a simple language switcher in your views, such as in a navigation bar or footer.

```html
<nav>
    <a href="/lang/en">English</a> | <a href="/lang/es">Español</a>
</nav>
```

---

### 2. **Create an "About Us" Page**

#### Step 2.1: Add a Route for the "About Us" Page

In `routes/web.php`, add a route for the "About Us" page:

```php
$router->get('/about-us', 'AboutUsController@index');
```

#### Step 2.2: Create an "About Us" Controller

Generate a controller to handle this page.

```bash
php artisan make:controller AboutUsController
```

In the `AboutUsController.php`, add:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class AboutUsController extends Controller
{
    public function index()
    {
        return view('about-us');
    }
}
```

#### Step 2.3: Create the View for the "About Us" Page

Create a new Blade view for the "About Us" page in `resources/views/about-us.blade.php`. You can use translation keys to make it multilingual.

**`resources/views/about-us.blade.php`:**

```html
<!DOCTYPE html>
<html lang="{{ app()->getLocale() }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ __('messages.about_us') }}</title>
</head>
<body>
    <h1>{{ __('messages.about_us') }}</h1>
    <p>
        {{ __('messages.about_us_description') }}
    </p>
</body>
</html>
```

#### Step 2.4: Add Translations for "About Us" Content

Update the `resources/lang/en/messages.php` and `resources/lang/es/messages.php` files to include the description for the "About Us" page.

**In `resources/lang/en/messages.php`:**

```php
return [
    'welcome' => 'Welcome to our Bus Travel Service!',
    'about_us' => 'About Us',
    'about_us_description' => 'We are a company providing bus travel services with comfortable and reliable transportation.',
    'departure_schedule' => 'Departure Schedule',
    'contact' => 'Contact Us',
    'book_ticket' => 'Book Ticket',
];
```

**In `resources/lang/es/messages.php`:**

```php
return [
    'welcome' => '¡Bienvenido a nuestro servicio de viajes en autobús!',
    'about_us' => 'Sobre Nosotros',
    'about_us_description' => 'Somos una empresa que ofrece servicios de viajes en autobús con transporte cómodo y confiable.',
    'departure_schedule' => 'Horario de Salidas',
    'contact' => 'Contáctenos',
    'book_ticket' => 'Reservar Boleto',
];
```

---

### 3. **Update Views for Multilingual Content**

Let’s go over updating other pages to support multiple languages.

#### Step 3.1: Update the Bus Schedule Page

In the `bus-schedules.blade.php` view (or wherever you display your schedule):

```html
<h1>{{ __('messages.departure_schedule') }}</h1>

<table>
    <thead>
        <tr>
            <th>{{ __('messages.origin') }}</th>
            <th>{{ __('messages.destination') }}</th>
            <th>{{ __('messages.departure_time') }}</th>
            <th>{{ __('messages.available_seats') }}</th>
            <th>{{ __('messages.price') }}</th>
        </tr>
    </thead>
    <tbody>
        @foreach ($schedules as $schedule)
            <tr>
                <td>{{ $schedule->origin }}</td>
                <td>{{ $schedule->destination }}</td>
                <td>{{ $schedule->departure_time }}</td>
                <td>{{ $schedule->available_seats }}</td>
                <td>{{ $schedule->price }}</td>
            </tr>
        @endforeach
    </tbody>
</table>
```

Make sure that any other text (like column names) is translated using the `__()` helper, so it's displayed based on the active language.

---

### 4. **Final Touches**

- **Session Handling**: To persist language preferences between page loads, ensure the language is stored in a session (as shown in step 1.2). You can also use cookies or the database if you want to persist it for users long-term.

- **URL Structure**: You might want to structure the URLs like `/en/about-us` or `/es/about-us` to make the language switch more intuitive for users and search engines. You can achieve this by adding a locale parameter to your routes or using middleware.


---

### Conclusion

With these steps, you've added:

1. **Multilingual Support**: Now your website can handle two languages (English and Spanish), and you can easily extend this to more languages by adding more language files.
2. **About Us Page**: A new "About Us" page, which is also multilingual, has been added to your site.
3. **Dynamic Language Switching**: You can dynamically switch between languages using a language switcher.

Feel free to ask if you need further clarification or help with any step!

######################

Sure! Below is an example of the folder structure for your Lumen project, which includes the multilingual functionality, "About Us" page, bus schedules, and other necessary components.

Here’s how the structure would look like:

```
bus-travel-site/                    # Root of the project
├── app/                             # Contains application logic
│   ├── Http/                        # Controllers, Middleware, etc.
│   │   ├── Controllers/             # All controllers for handling requests
│   │   │   ├── AboutUsController.php # Handles the "About Us" page
│   │   │   ├── BusScheduleController.php # Handles the bus schedules
│   │   │   ├── PaymentController.php   # Handles payments
│   │   │   ├── AuthController.php      # Handles authentication logic
│   ├── Providers/                    # Service providers like AppServiceProvider
│   │   └── AppServiceProvider.php
│   ├── Models/                       # Eloquent models
│   │   ├── BusSchedule.php            # Bus schedule model
│   │   ├── User.php                   # User model (for authentication)
│   │   └── Payment.php                # Payment model
├── bootstrap/                        # Lumen startup files
│   ├── app.php                        # Lumen app configuration
├── config/                           # Configuration files (not many in Lumen by default)
├── database/                         # Database migrations and seeds
│   ├── migrations/                    # Migrations for creating tables
│   │   ├── 2024_01_01_create_users_table.php
│   │   ├── 2024_01_01_create_bus_schedules_table.php
│   │   ├── 2024_01_01_create_payments_table.php
│   ├── seeds/                        # Seed files for demo data
├── public/                           # Public folder where your assets are served
│   ├── index.php                     # Entry point for Lumen application
│   ├── css/                          # CSS files (if any)
│   ├── js/                           # JS files (if any)
│   └── images/                       # Images (if any)
├── resources/                        # Resources like views and language files
│   ├── lang/                         # Language files for translations
│   │   ├── en/                       # English language folder
│   │   │   └── messages.php          # English translations (e.g., 'welcome', 'about_us')
│   │   ├── es/                       # Spanish language folder
│   │   │   └── messages.php          # Spanish translations
│   ├── views/                        # Views (HTML, Blade templates, etc.)
│   │   ├── bus-schedules.blade.php   # Displays bus schedule
│   │   ├── about-us.blade.php        # "About Us" page
│   │   ├── payment.blade.php         # Payment page (with Stripe integration)
│   │   ├── auth/                     # Auth views (login, register, etc.)
│   │   │   ├── login.blade.php
│   │   │   ├── register.blade.php
├── routes/                           # Route definitions
│   ├── web.php                       # Main routes (including /about-us, /bus-schedules, /lang)
├── storage/                          # Cache, logs, sessions
├── .env                              # Environment configuration (database, Stripe keys)
├── composer.json                     # Composer dependencies file
└── artisan                           # Lumen command-line tool
```

### Folder Breakdown:

#### 1. **app/**

- **Controllers/**: Contains controllers for handling business logic (e.g., `AboutUsController`, `BusScheduleController`, `PaymentController`, and `AuthController` for authentication).
- **Models/**: The Eloquent models for handling data from your database (e.g., `User.php`, `BusSchedule.php`, `Payment.php`).
- **Providers/**: Contains `AppServiceProvider.php`, which is where you would set up your application settings, including the language localization in `boot()`.

#### 2. **bootstrap/**

- **app.php**: This file bootstraps your Lumen application. If you're using localization, you'll configure language settings here.

#### 3. **config/**

- Lumen has minimal configuration by default, but you can add custom configuration files here if needed (e.g., for payment gateways, caching, etc.).

#### 4. **database/**

- **migrations/**: Contains migration files for creating the database schema (e.g., creating `users`, `bus_schedules`, and `payments` tables).
- **seeds/**: Seeds for populating the database with sample data, if needed.

#### 5. **public/**

- **index.php**: The entry point of the application.
- **css/**, **js/**, **images/**: These folders will store your CSS, JavaScript, and image files (for frontend assets).

#### 6. **resources/**

- **lang/**: Language files for translations. You can store translation strings in the `en` and `es` folders (or more languages if needed). For example, `messages.php` would contain keys like `'welcome'`, `'departure_schedule'`, etc., for different languages.
- **views/**: Contains Blade views for displaying the pages on the website. You have views like `bus-schedules.blade.php` (for the bus schedule), `about-us.blade.php` (for the About Us page), and `payment.blade.php` (for the payment page). You can also store authentication views (e.g., `login.blade.php`, `register.blade.php`).

#### 7. **routes/**

- **web.php**: The main routing file. All the routes like `/about-us`, `/bus-schedules`, `/lang`, and others are defined here.

#### 8. **.env**

- The `.env` file contains environment variables like database connection settings, API keys (e.g., Stripe), and the default language settings.

---

### Example File Content:

#### `resources/lang/en/messages.php` (English Translations)

```php
<?php

return [
    'welcome' => 'Welcome to our Bus Travel Service!',
    'about_us' => 'About Us',
    'about_us_description' => 'We are a company providing bus travel services with comfortable and reliable transportation.',
    'departure_schedule' => 'Departure Schedule',
    'contact' => 'Contact Us',
    'book_ticket' => 'Book Ticket',
];
```

#### `resources/lang/es/messages.php` (Spanish Translations)

```php
<?php

return [
    'welcome' => '¡Bienvenido a nuestro servicio de viajes en autobús!',
    'about_us' => 'Sobre Nosotros',
    'about_us_description' => 'Somos una empresa que ofrece servicios de viajes en autobús con transporte cómodo y confiable.',
    'departure_schedule' => 'Horario de Salidas',
    'contact' => 'Contáctenos',
    'book_ticket' => 'Reservar Boleto',
];
```

#### `resources/views/about-us.blade.php` (About Us Page)

```html
<!DOCTYPE html>
<html lang="{{ app()->getLocale() }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ __('messages.about_us') }}</title>
</head>
<body>
    <h1>{{ __('messages.about_us') }}</h1>
    <p>{{ __('messages.about_us_description') }}</p>
</body>
</html>
```

---

### Summary:

This folder structure provides a well-organized and maintainable approach for your multilingual bus travel website. It contains the necessary components to handle user authentication, bus schedules, payments, and dynamic language switching between English and Spanish. As your application grows, you can easily add more language files and expand the functionality by following this structure.

Let me know if you need help with specific files or further explanations!


################################

Absolutely! To add an **Admin Control Panel** to your Lumen-based bus travel website, we can implement a simple interface where admins can:

- Manage **bus schedules** (add, edit, delete).
- View **user data** (view registered users).
- View and manage **payments** (view completed payments).
- Change the **status of bus trips** (e.g., mark trips as "sold out").

Since Lumen is a micro-framework and doesn't come with built-in admin features like Laravel, we will need to build this manually, but I’ll guide you through a simple approach to create an admin panel.

### Steps to Create an Admin Control Panel

1. **Authentication for Admin**
2. **Create Admin Routes**
3. **Create Admin Controllers**
4. **Create Admin Views**
5. **Implement CRUD for Bus Schedules**

---

### 1. **Authentication for Admin**

The first step is to set up a way to authenticate an admin. Since Lumen doesn’t have a built-in admin authentication system like Laravel, we’ll manually check if the logged-in user has an admin role. For simplicity, let's assume the `users` table has an `is_admin` boolean field that determines whether a user is an admin.

#### Step 1.1: Modify Users Table

If you haven’t already, update the `users` table to include an `is_admin` column. You can add this via a new migration.

```bash
php artisan make:migration add_is_admin_to_users_table
```

In the migration file:

```php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->boolean('is_admin')->default(false);  // Add this line
    });
}
```

Run the migration:

```bash
php artisan migrate
```

#### Step 1.2: Authentication Middleware

You’ll need to create middleware to ensure that only admins can access the admin panel.

Create a middleware file:

```bash
php artisan make:middleware AdminMiddleware
```

In `app/Http/Middleware/AdminMiddleware.php`, write the logic to check if the user is an admin:

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Auth;

class AdminMiddleware
{
    public function handle($request, Closure $next)
    {
        if (Auth::check() && Auth::user()->is_admin) {
            return $next($request);
        }

        return response()->json(['error' => 'Unauthorized'], 403); // Or redirect to login
    }
}
```

Then, register the middleware in `app/Http/Kernel.php` (if it’s not already there):

```php
protected $routeMiddleware = [
    'admin' => \App\Http\Middleware\AdminMiddleware::class,
];
```

#### Step 1.3: Protect Admin Routes

Now, in your `routes/web.php`, you can protect the admin routes by using the `admin` middleware:

```php
$router->group(['middleware' => 'admin'], function () use ($router) {
    // Admin routes
    $router->get('/admin/dashboard', 'AdminController@dashboard');
    $router->get('/admin/bus-schedules', 'AdminController@busSchedules');
    $router->get('/admin/users', 'AdminController@users');
    $router->get('/admin/payments', 'AdminController@payments');
    // Add more routes for CRUD operations, etc.
});
```

---

### 2. **Create Admin Routes**

Let’s create some routes that will serve as the entry points for the admin panel.

#### Step 2.1: Admin Dashboard Route

This will be the landing page of the admin panel. Add the following route in `routes/web.php`:

```php
$router->get('/admin/dashboard', 'AdminController@dashboard');
```

#### Step 2.2: Admin Bus Schedule Route

Add a route for viewing bus schedules:

```php
$router->get('/admin/bus-schedules', 'AdminController@busSchedules');
```

---

### 3. **Create Admin Controller**

Now, let’s create an `AdminController` where we can define the logic for handling the admin panel's features (dashboard, bus schedules, user management, etc.).

Create the controller:

```bash
php artisan make:controller AdminController
```

#### Step 3.1: Admin Dashboard

In `AdminController.php`, define the `dashboard()` method:

```php
namespace App\Http\Controllers;

use App\Models\BusSchedule;
use App\Models\User;
use App\Models\Payment;

class AdminController extends Controller
{
    public function dashboard()
    {
        $userCount = User::count();
        $busCount = BusSchedule::count();
        $paymentCount = Payment::count();

        return view('admin.dashboard', compact('userCount', 'busCount', 'paymentCount'));
    }
}
```

#### Step 3.2: Admin Bus Schedules

For the bus schedules page, you will define the logic for viewing, adding, and deleting schedules:

```php
public function busSchedules()
{
    $schedules = BusSchedule::all();
    return view('admin.bus-schedules', compact('schedules'));
}
```

---

### 4. **Create Admin Views**

You’ll need to create the views for the admin panel. These views will display the data fetched in the controller methods.

#### Step 4.1: Admin Dashboard View

Create the view file `resources/views/admin/dashboard.blade.php`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Dashboard</title>
</head>
<body>
    <h1>Admin Dashboard</h1>
    <ul>
        <li>Total Users: {{ $userCount }}</li>
        <li>Total Bus Schedules: {{ $busCount }}</li>
        <li>Total Payments: {{ $paymentCount }}</li>
    </ul>
    <a href="/admin/bus-schedules">Manage Bus Schedules</a> |
    <a href="/admin/users">Manage Users</a> |
    <a href="/admin/payments">View Payments</a>
</body>
</html>
```

#### Step 4.2: Bus Schedules View

Create the view file `resources/views/admin/bus-schedules.blade.php`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Manage Bus Schedules</title>
</head>
<body>
    <h1>Manage Bus Schedules</h1>
    <a href="/admin/bus-schedules/create">Add New Schedule</a>
    <table>
        <thead>
            <tr>
                <th>Origin</th>
                <th>Destination</th>
                <th>Departure Time</th>
                <th>Available Seats</th>
                <th>Price</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($schedules as $schedule)
                <tr>
                    <td>{{ $schedule->origin }}</td>
                    <td>{{ $schedule->destination }}</td>
                    <td>{{ $schedule->departure_time }}</td>
                    <td>{{ $schedule->available_seats }}</td>
                    <td>{{ $schedule->price }}</td>
                    <td>
                        <a href="/admin/bus-schedules/edit/{{ $schedule->id }}">Edit</a> |
                        <a href="/admin/bus-schedules/delete/{{ $schedule->id }}">Delete</a>
                    </td>
                </tr>
            @endforeach
        </tbody>
    </table>
</body>
</html>
```

#### Step 4.3: Users Management View

You can create a similar view for managing users. Create the view `resources/views/admin/users.blade.php`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Manage Users</title>
</head>
<body>
    <h1>Manage Users</h1>
    <table>
        <thead>
            <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Role</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($users as $user)
                <tr>
                    <td>{{ $user->name }}</td>
                    <td>{{ $user->email }}</td>
                    <td>{{ $user->is_admin ? 'Admin' : 'User' }}</td>
                    <td>
                        <a href="/admin/users/edit/{{ $user->id }}">Edit</a> |
                        <a href="/admin/users/delete/{{ $user->id }}">Delete</a>
                    </td>
                </tr>
            @endforeach
        </tbody>
    </table>
</body>
</html>
```

---

### 5. **Implement CRUD for Bus Schedules**

Now, to complete the **Create**, **Read**, **Update**, and **Delete** functionality for bus schedules, you'll need to implement the following:

- **Create Route and View**: A form to add new bus schedules.
- **Edit Route and View**: A form to edit existing bus schedules.
- **Delete Route**: A route to delete bus schedules.

For brevity, I'll leave the implementation details of these actions (CRUD) to you, but they will
