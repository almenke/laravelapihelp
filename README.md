# Laravel API Help
The new [Laravel](https://laravel.com/) v5.3 has some new routing features, and is a great platform to develop your server (Controller and Model) layer for your mobile clients for REST API's.  Chances are that if you chose the Laravel PHP framework to develop a new web application you will love it and inevitably want to start using all that backend logic you created as API’s for your mobile platform.  There are a few pieces that I have struggled with in getting up and running with building API’s.  

Because of that, I have been creating a markdown file to capture these little nuggets of understanding.  Believe me, I don’t suggest that I am the authority on Laravel, not even close.  Frameworks are like learning different languages, patterns, and architecture.  You never really are an expert on everything, you only are an expert for a short time on the parts you use.  In order to not try and remember everything, because I can’t, but also to not start from scratch or reinvent the wheel every day, I like to dribble little crumbs so that I can get back to the information that I need. So this time I decided to put my dribble on a public repository.

Without further wasting my time or yours, here we go. This information is for anyone interested in developing RESTful API's using Laravel 5.3 at the service layer and passing a token on calls.  

*Note: I don't claim to be an expert and there are many sophisticated ways of doing this so if you read this and disagree with the way I have figured things out then you probably don't need to be here in the first place because you are beyound this help.*

## Goal
Develop API's to utilize the previously developed Laravel 5.3 web application utilizing the authorization and routing capabilities that exist out-of-the-box.

## Objectives
1. Use the Laravel 5.3 authentication, adding an api_token to the users table, to pass back to the client.
2. Try not to hack up the framework.
3. Do not start with Passport and OAuth2, but leave the door open to using it in the future.

##Authentication & Authorization
How do we call login as an API and receive a token?  I found a lot of great documents and Q&A throughout the web on how to do the routing and token passing after login but after scouring the web I did not find out how people were logging in.  Also, I knew I had to be over-thinking this and there should just be a way to utilize the existing routing and authorization that was already built out.  With the help of Jacob Bennitt's GistLog at [JacobBennett API Token Authentication in Laravel](https://gistlog.co/JacobBennett/090369fbab0b31130b51) I decided quickly understood how to modify the users table and then extrapolate that into a few more steps to get up and running.

Here are all the steps that I took to get the login working, returning the token.

***Side Note:*** *I suggest using* [Google Chrome Postman](https://www.getpostman.com) to test your API.  This is a great tool from within Chrome and saves a ton of time.  You don't have to write a test app or use Curl from the command line.

***And one more Side Note before we start, most of this readme is just based on pulling out the obvious forest that I didn't see because I was standing in the trees.***

###API Login Modification Steps and Testing using Postman
1. Add `api_token` string to the users table by modifying the users migration file.

  ~~~~
  $table->string('api_token', 60)->unique();
  ~~~~
  
2. Change the User.php model and add the column to the fillable array.  I saw a video where a suggestion was to also put this in the $hidden array.  I didn't do this because aftre login, I wanted to just return the entire user and obviously we need the token on the login return.
3. If you have  a user seeder then you will want to add the following line to init your api_token column in the user seeder code.  My user seeder looks something like this.

 ~~~~
  User::create(
            array(
                'fname' => 'Al',
                'lname' => 'Menke',
                'role_id' => 5,
                'email' => 'al.menke@gmail.com',
                'password' => Hash::make('mypassword'),
                'active' => 1,
                'api_token' => str_random(60)
            )
        );
 ~~~~
  You might notice that I have a few other columns that you in my users table that you don't see in the default like role_id and an active flag.  Don't worry about those because they have nothing to do with this API information.
4. After you do these modifications, don't forget that you will have to rebuild your database.  If you are using the migrate and seeder from artisan then run the commands below.  You might have to follow up with a restore from a backup also but if that is the case then don't forget to modify the user table appropriately.

 ~~~~
 php artisan migrate:rollback
 php artisan migrate
 php artisan db:seed
 ~~~~
 
5. At this time, you should be able to look into your database at your newly seeded users table and see your user with a new huge randomly generated api_token.
6. Update the create method in the /Controllers/Auth/RegisterController.php add `'api_token' => str_random(60)` to the create values.  My code block looks like the following.

 ~~~~
 // /Controllers/Auth/RegisterController.php
 protected function create(array $data)
    {
        return User::create([
            'fname' => $data['fname'],
            'lname' => $data['lname'],
            'role_id' => \Config::get('constants.ROLE_MEMBER'),
            'email' => $data['email'],
            'active' => 1,
            'password' => bcrypt($data['password']),
            'api_token' => str_random(60)
        ]);
    }
 ~~~~
 
7. When there is an unsuccessful login or a successful login through the api, you will want it to return json and not redirect to a default webpage or back to a login page like the web application does.  So, I decided to just modify the login method.  When you open the LoginController.php supplied in the authorization code you will not see a login method.  That is beause it uses the AuthenticatUsers trait.  So just modify the login method in the AuthenticateUsers.php trait. in the login method add the following code within the successful login side of the if attempt statement.

     ~~~~
     if ($request->ajax() || $request->expectsJson()) {
       return \Response::json(['rc' => 201, 'message' => 'Success', 'user' => Auth::user()]);
     }
     ~~~~
     
8. Now we can modify the method sendFailedLoginResponse with the following code to handle login failures.

 ~~~~
  if ($request->ajax() || $request->expectsJson()) { 
     return \Response::json(['rc' => 403, 'message' => 'Unauthorized.']);
  }
  ~~~~
  
9. To call the login from the API client (Laravel 5.3) we need to add a route to the api.php file.  In the new 5.3 we do not have to prefix with api in the routes because Laravel automatically routes api prefix requests through the api.php route file.  However, if we wanted to ad a version level to the URI then we need to prefex with that version.  To start out I did not add a version level for the v1 code.  I figured I could start adding the version level in routes when that becomes an issue. The new route for login is as follows.  

 ~~~~
 Route::post('/login', 'Auth\LoginController@login');
 ~~~~

  *Note: This route looks just like other the routes in the web.php file however we did not have an api route for login in the web.php file.*
  
10. All that's left is to test.  Assuming that your webserver is up and running then open your PostMan route and add the following URL with the POST method selected and include the parameters.

 ~~~~
 http://yourserver/api/login
 
    email = yourseededemail
    password = unencryptedpasswordtext
 ~~~~
 
11. Now send the request.  If you have the right url and key value pairs it should.... what happened?  It routed to the home page or the page in your login controller after the successful login just like the web does.  That is because you need to add to the header Accept = application/json.  Remember we added if $request->Json() in our successful return.  After that you should receive a successful Json string.

###API Routing with Authorization using API Auth and a Token
After you have successfully setup your api_token and performed a login, you will get a user back which has a token.  On every subsequent call you to your API's you will put the token in as a posted key value pair or on the command line for GET requests.  I suggest not putting it in the GET request because it would be visible for anyone to use.  If you put it into the key value pairs then it would be better. 

***Side Note:*** *I suggest using Passport with OAuth2 for "real" security and standardization.  I just wanted to get up and running and be able to actually code disconnected from the Internet.  I know most of you are probably turning your head sideways on that but some of us code remotely where there is not wifi like in planes, trains, and moving automobiies.*

####Steps to add the token to all API routing
1. We can add a routing group into the api.php file which has the auth:api closure or we can add it to the end of each route.  I chose to add it to a group which looks like the following.

 ~~~~
    Route::group(['middleware' => 'auth:api'], function() {
      Route::get('/altest', function () {
        return \Response::json(['rc' => 201, 'message' => 'Hello, Al from group']);}); 
      Route::resource('/firm', 'FirmController');
    });
 ~~~~
 
2. If you want to include a version then all you have to do is add a prefix parameter like as follows.

 ~~~~
    Route::group(['prefix' => '/v1', 'middleware' => 'auth:api'], function() {
      Route::get('/altest', function () {
        return \Response::json(['rc' => 201, 'message' => 'Hello, Al from group']);}); 
      Route::resource('/firm', 'FirmController');
    });
 ~~~~
 
3. The think that I got caught on and burned a lot of time is in all the examples that I saw, everyone was prefixing with api/v1 or api/.  Since Laravel 5.3 has the api.php route file, this is not necessary because laravel already routes all the api prefixes to the api.php route file.
4. One final note.  If you are going to use the same controllers and repository providers, etc. in your API calls that you use in your web application then you will need to constantly check how you were called and reroute or send back json appropriately.  There are many ways to handle this, it might be appropriate to build a return method in your parent controller. App\Http\Controllers\Controller.php file.  You could just pass either a route or a json string to that return and have it determine how your controller was called.  You could also build a return trait and use that in your controllers.  I suppose many of you will think of other ways.


