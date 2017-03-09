#Authentication and Authorization in Laravel
The following provides a brief walkthrough on some of Laravel's authentication and authorization features. It assumes you have aready completed the first two Laravel practicals. 

Laravel offers 'out of the box' authentication (https://laravel.com/docs/5.4/authentication). With some simple artisan commands it will generate an entire login/registration system. We will take a more hands-on approach but still use many of the framework's authentication features to help us. 

##Creating a simple login system
Laravel automatically generates a *users* table for us. If you look in phpMyAdmin you should be able to see it. First we will populate this with some users. 
* Using Artisan create a users table seeder
```
php artisan make:seeder UsersTableSeeder
```
* Open the seeder file and add some insert statements to create some users e.g.
```

use Illuminate\Database\Seeder;

class UsersTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        DB::table('users')->insert(['name' => 'Kate','email' => 'k.l.hutton@hudstudent.ac.uk','password' => bcrypt('password')]);
        DB::table('users')->insert(['name' => 'Yousef','email' => 'y.miandad@hudstudent.ac.uk','password' => bcrypt('letmein')]);
        DB::table('users')->insert(['name' => 'Sunil','email' => 's.laxman@hudstudent.ac.uk','password' => bcrypt('password2')]);
    }
}
```
* Open the DatabaseSeeder and add a call to the UsersTableSeeder
* Next, re-run the migration and re-seed the database

```
php artisan migrate:refresh --seed
```

* Check in phpMyAdmin, you should see some data in your users table. 

* We'll make a new controller to handle logging in, enter the following artisan command

```
php artisan make:controller LoginController
```

* Create a new folder in *resources/views*, name it *login*
* Create a new blade file, a login form
```
@if (session('loginError'))
    <div>
        {{ session('loginError') }}
    </div>
@endif
<form action="{{url('login')}}" method="POST">
{{ csrf_field() }}
<h1>Login</h1>
<div>
<label for="title">Email:</label>
<input type="email" name="email" value="{{old('email')}}" id="email">
</div>
<div>
<label for="password">Password:</label>
<input type="password" name="password" id="password">
</div>
<div>
<input type="submit" name="submitBtn" value="Login">
</div>
</form>
```
* Add two new routes in your web.php routes file
```
Route::get('login', 'LoginController@loginForm');

Route::post('login', 'LoginController@login');

```
* In the LoginController add the corresponding loginForm and login methods.
* Your LoginController will look like the following:

```
namespace App\Http\Controllers;

use Illuminate\Support\Facades\Auth;
use Illuminate\Http\Request;

class LoginController extends Controller
{
    function loginForm()
    {
        return view('login/login');
    }
    function login(Request $request)
    {
        if (Auth::attempt(['email' => $request->email, 'password' => $request->password])) {
            return redirect('/all');
        }
        $request->session()->flash('loginError', "Those details aren't correct");
        return redirect('/login');
    }
}

```

There are a couple of things we haven't seen before
* The Auth facade handles authentication for us.
 ```
 Auth::attempt(['email' => $request->email, 'password' => $request->password])
 ```
 The *attempt* method will check the users table for the email and password see https://laravel.com/docs/5.4/authentication#authenticating-users. 

* We can work with sessions using the session helper, this is made available via the $request object
 ```
 $request->session()->flash('loginError', "Those details aren't correct");
 ```
 
* Flash data is only stored until the next request (https://laravel.com/docs/5.4/session#flash-data). This message will be displayed above the login form if the user enters the wrong details (have a look back at *login.blade.php*)

* Test this works. If the user enters correct login details they should be taken to the list of all the films. If they don't, they will be re-directed back to the login page. 

* Even though the login works, users are still able to access pages without logging in if they enter a correct url e.g.http://localhost/laravel-project/public/all . 

There are a number of different ways of protecting routes in Laravel see (https://laravel.com/docs/5.4/authentication#protecting-routes). In this example we will protect the routes using a controller.  For now let's assume we don't want the users to access any part of the site unless they are logged in. 

* Open the FilmController. Add the following constructor function 

```
function __construct()
    {
        $this->middleware('auth');
    }
```

Test this works. You may have to open a new web browser window. Try and access a route e.g. http://localhost/cit2318/laravel/public/all without logging in first. 

###Giving the user some feedback
Next open up *master.blade.php*. Add the following after the list of links
```
<div>Logged in as : {{Auth::user()->name}}</div>
```
Again test this works. Once the user has logged in their name should be displayed on every page of the site. 

##Logging out
The Auth facade also provides a logout method
```
Auth::logout();
```
* Add a logout method to the LoginController that uses this code and then redirects the user to the login form. 
* Add a logout route in web.php
* Add a logout link to the list of links in *master.blade.php*
* Test this works. 

##Authorization
Auhtorization is all about determining who is allowed to perform certain actions on our site. In our simple application:
* All users (as long as they are logged in) can view films
* If the user is an administrator they also add and delete films 

###Specifying roles
A very simple way of specifying different roles for users is by adding an additional *role* column in the *users* table. A value of 1 will indicate a regular user, a value of 2 will indicate an administrator. 

Open the *users* table migration and add an additional column for the user's role.
```
...
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->tinyInteger('role'); //new column
            $table->rememberToken();
            $table->timestamps();
        });
    }
...
```

Open the users table seeder and add values for the role column e.g.
```
use Illuminate\Database\Seeder;

class UsersTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        DB::table('users')->insert(['name' => 'Kate','email' => 'k.l.hutton@hudstudent.ac.uk','password' => bcrypt('password'),'role'=>1]);
        DB::table('users')->insert(['name' => 'Yousef','email' => 'y.miandad@hudstudent.ac.uk','password' => bcrypt('letmein'),'role'=>2]);
        DB::table('users')->insert(['name' => 'Sunil','email' => 's.laxman@hudstudent.ac.uk','password' => bcrypt('password2'),'role'=>1]);
    }
}

```

* Laravel allows us to specify actions and then determine if a user is allowed to perform that action. 
* One way of doing this is to use a policy (https://laravel.com/docs/5.4/authorization#creating-policies) . Policies allow us to organise authorization around a given model (in our case the Film model).

* Instruct artisan to generate a Film policy for us

```
php artisan make:policy FilmPolicy --model=Film
```

* Have a look in the *app/Policies* folder and open the generated *FilmPolicy.php*.
* We only want to restrict access to creating and deleting films
 * Delete the *view* and *update* methods
* We need to add some logic to the methods to determine if the action is allowed. Modify your FilmPolicy class so that it looks like the following:

```
namespace App\Policies;

use App\User;
use App\Film;
use Illuminate\Auth\Access\HandlesAuthorization;

class FilmPolicy
{
    use HandlesAuthorization;

    public function create(User $user)
    {
        if($user->role==2){
            return true;
        }
        return false;
    }

    public function delete(User $user)
    {
        if($user->role==2){
            return true;
        }
        return false;
    }
}

```

* Next we need register the policy. Open *AuthServiceProvider.php*
* Add the FilmPolicy to the array of policies
```
...
    protected $policies = [
        'App\Model' => 'App\Policies\ModelPolicy',
        Film::class => FilmPolicy::class,
    ];
...
```

###Authorizing Actions
Now the policy is set up we can use it to restrict access to parts of the site 
* Open *master.blade.php*
* Modify the list of hyperlinks so it looks like the following
```
...
 <ul>
                <li><a href="{{url('all')}}">View Films</a></li>

                @can('create', App\Film::class)
                    <li><a href="{{url('deleteform')}}">Delete Films</a></li>
                    <li><a href="{{url('addform')}}">Add Film</a></li>
                @endcan
                
                <li><a href="{{url('logout')}}">Logout</a></li>
            </ul>
...           
```

The *@can* directive specifies that only those users authorised to perform the *create* action (user's role is equal to 2) can view the links to delete films and add film. 

* Test this works. If you are signed is as Yousef you should see the links. If you are signed in as anyone else you shouldn't. 

Users will still be able to access routes directly by entering a valid url. To prevent this from happening open the *web.php* routes file. Add a call to 'middleware' to authorize access to certain routes e.g.

```
Route::get('addform', 'FilmController@addForm')->middleware('can:create,App\Film');
```
* Middleware is code that is run before reaching our controllers (https://laravel.com/docs/5.4/middleware). 
* Test this works. Login in as Kate and try to access the *addform* route. You should get an authorization error.  


##On Your Own
* The authentication system we have built is very simple. Try the following
 * Adding validation to the form e.g. has the user entered a valid email address
 * Allow users to register for the site. 
* There is also a lot more to authorization in Laravel, see https://laravel.com/docs/5.4/authorization for full details. 


