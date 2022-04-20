## Handling Concurrency attacks in laravel


A harsh truth many developers do not realize early is that for every application you move to production,
there are people who only seek ways to exploit your product for their own benefit.  
This consciousness is one that is required in building truly secure applications. Concurrency attacks are not new, especially in the financial world. Many companies have suffered due to such attacks and in this post, we would see how laravel solves this problem without a developer breaking a sweat!  Start by installing a fresh laravel application

```
composer create-project laravel/laravel concurrency
```

Next, set up the database. First, you should add your database credentials in your .env file


![Screenshot 2022-04-06 at 00.04.29.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649199887936/S1ppKf-kO.png)

For the purpose of this demonstration, we would need two models, The User model which comes with every fresh laravel installation and the Transaction model. To create the Transaction model alongside the migration file, run

```
php artisan make:model Transaction -m
```
Go ahead to modify the migration files created to look like the ones shown below.

```
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
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
        Schema::dropIfExists('users');
    }
};
```

```
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('transactions', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained();
            $table->decimal('amount');
            $table->string('description');
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
        Schema::dropIfExists('transactions');
    }
};
```

Next, you need to seed some records into the database. The code below creates two users from the user factory which is already included in every fresh laravel installation. We would also need to seed the transaction table with one record which credits one of the users with a sum of $5000.

```
<?php

namespace Database\Seeders;

use App\Models\Transaction;
use App\Models\User;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     *
     * @return void
     */
    public function run()
    {
         User::factory(1)->create([
             'email' => 'anis@mail.com',
             'name' => 'Anis'
         ]);

        $user = User::factory(1)->create([
            'email' => 'tobexkee@mail.com',
            'name' => 'Tobexkee'
        ])->first();

        Transaction::create([
            'user_id' => $user->id,
            'amount' => 5000,
            'description' => 'Initial deposit'
        ]);
    }
}
```

Having set up the basic things required, I would now explain what we are trying to build.  We are building a very basic wallet system. There are two users, Anis and Tobexkee. Tobexkee has $5000 in his wallet while Anis has $0. The goal is to write an API for Tobexkee to transfer some money to Anis. . We would define an endpoint in the api.php route file. The request would then be handled in a SendMoneyController.

```
<?php

use App\Http\Controllers\SendMoneyController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/


Route::post("send", SendMoneyController::class);
```
The basic function of the controller would be to debit the sender and credit the receiver. But before that, we need to ensure that the sender has sufficient money in his wallet to send the funds to the receiver. To do this,Create a request class by running the command below

```
php artisan make:request SendMoneyRequest
```  
This would create a request class which can be used in the controller method as shown below

```
<?php

namespace App\Http\Controllers;

use App\Http\Requests\SendMoneyRequest;
use App\Jobs\UserTransactionManagerJob;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class SendMoneyController extends Controller
{
    public function __invoke(SendMoneyRequest $request)
    {
        $user = User::where('email', $request->get('email'))->first();

        DB::beginTransaction();

        auth()->user()->transactions()->create([
            'amount' => -1 * $this->amount,
            'description' => "You sent {$this->amount} to {$user->name}"
        ]);

        $user->transactions()->create([
            'amount' => 1 * $this->amount,
            'description' => "You received {$this->amount} to ".auth()->user()->name
        ]);

        DB::commit();

        return response()->json(['status' => 'success!']);
    }
}
```
The request class would then look like this
```
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class SendMoneyRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }
    
    public function rules()
    {
        return [
            'amount' => ['required', function ($attr, $val, $fail) {
                if (auth()->user()->balance() < $val) {
                    return $fail('Insufficient balance');
                }
            }],
            'email' => ['required', Rule::exists('users', 'email')]
        ];
    }
}
```

This ensures that when a user tries to carry out a transaction that exceeds his wallet balance, the transaction would fail and the user would receive a 422 validation error with the message insufficient funds.

Now that the API is ready, we now need to test it. While postman is more suitable for this, we would rather use a Command class for the sake of what we want to achieve. This can be done by running
```
PHP artisan make:command SendMoneyCommand
```
All we would be doing in the command class is calling the API we have created. Do not forget that the goal is to send money from Tobexkee wallet to Anis wallet. We can structure the command class handle method  to look like this

 ```
<?php

namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Http;

class SampleCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'money:send';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Command description';

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        $response = Http::withHeaders([
            'accept' => 'application/json'
        ])->post('http://concurrency.test/api/send', [
            'email' => 'anis@mail.com',
            'amount' => 5000
        ])->json();
      
        dd($response, User::where('email', 'tobexkee@mail.com')->first()->balance());
    }
}

```
Here what is being done is just calling the API and passing Anis's email to it as the recipient. Now we would have an issue here, we need to ensure that Tobexkee is the authenticated user so that everything would work as expected. We can choose to use JWT or Sanctum or even something custom but I would not want to take that route as that is not the focus of this article. Instead, I would just tell laravel to use Tobexkee as the authenticated user for all requests. I can do this in the AppServiceProvider by adding this line of code to the boot method

```
public function boot()
{
        Auth::login(User::where('email', 'tobexkee@mail.com')->first());
}
```

So now we are good to go. Running the command we added would give the result how below. 
 ```
PHP artisan money:send
```

```
array:1 [
  "status" => "success!"
]
"0.00"
```
The focus is on the server response as well as the balance of the user which we are dumping at the end of the request.  When you run it for the first time, the response obtained is expected. Tobexkee has $5000 so he can afford to send the  $5000 to Anis. 

Tobexkee new balance would then be 0.00. If you run the command again, The response should now look like what we have below.
```
"message" => "Insufficient balance"
  "errors" => array:1 [
    "amount" => array:1 [
      0 => "Insufficient balance"
    ]
  ]
]
"0.00"
```
this is also a desired response since Tobexkee no longer has enough to afford another transaction. 
This is how a typical system would be expected to function. However, if some conditions were to change, it would be quickly obvious that this system is faulty. Start by refreshing the database and seed the default records in the database again
```
php artisan migrate:refresh --seed
```
Then slightly adjust the code in the command class. The change to make this time around is that instead of sending one request at a time, three requests would be sent asynchronously so that the server would receive them as concurrent requests.  To do this, we would use the pool method on the laravel HTTP client facade which utilizes Guzzle Http Promises under the hood. The new code would now look like this

```
public function handle()
    {
        $responses = Http::pool(fn (Pool $pool) => [
            $pool->post('http://concurrency.test/api/send', [
                'email' => 'anis@mail.com',
                'amount' => 5000
            ]),
            $pool->post('http://concurrency.test/api/send', [
                'email' => 'anis@mail.com',
                'amount' => 5000
            ]),
            $pool->post('http://concurrency.test/api/send', [
                'email' => 'anis@mail.com',
                'amount' => 5000
            ]),
        ]); 


        $format = collect($responses)->map(fn ($response) => $response->throw()->json());

        dd($format, User::where('email', 'tobexkee@mail.com')->first()->balance());
    }
```
Running the command again, the response would look like this

```
Illuminate\Support\Collection {#1433
  #items: array:3 [
    0 => array:1 [
      "status" => "success!"
    ]
    1 => array:1 [
      "status" => "success!"
    ]
    2 => array:1 [
      "status" => "success!"
    ]
  ]
  #escapeWhenCastingToString: false
}
"-10000.00"
```

The first part of the result shows the response of the three concurrent requests made to the API. The first thing to notice here is that the three API calls all returned a successful response This implies that Tobexkee must have been debited three times.
This also means that Anis would have now been credited three times. Meaning Tobexkee has transferred $15,000 to Anis even though all he had was just $5000. This means some $10,000 must have been sent from thin air. Anis can then go ahead to Cash out the $15000 he must now have in his wallet to enjoy a cold bottle of beer with Tobexkee. If we increase the concurrent request to ten, we would get a similar result where Tobexkee would have transferred $5,000 to Anis ten times when he in fact can not afford to transfer more than once. The extra buck of cash flow here results in a loss for the company. This is a concurrency attack. 

The other part of the response shows that Tobexkee now has a negative balance which is not something that looks good! Double spending is always a nightmare for programmers working on fintech apps. 

### WHAT HAPPENED TO THE VALIDATION CHECK?
First, nothing is wrong with the validation. If one request was sent at a time or even concurrent requests
from different users, it properly validates the amount being sent and informs the user when the wallet balance is insufficient.
But it appears when we have concurrent requests from the same user, the validation is not sufficient to prevent users from accessing more than they have. To fully understand this, we need to understand how servers handle requests. 

### NGINX REQUESTS
When a request is made from either the browser or an HTTP client to an NGNIX server, the request is picked up by a worker process the worker process is responsible for handling the request. Note that there are other worker processes that can concurrently and asynchronously handle requests. In fact, NGINX can handle up to 500K requests per second as some benchmarks indicate. Once a worker process picks this up, it goes ahead to hand over the request processing to a PHP-FPM process. PHP-FPM either spawns a new process or used an already spawned but available process. This then goes ahead to process all of the PHP code. The focus here is that requests handling in NGINX are non-blocking meaning NGINX does not handle one request at a time hence rather is has multiple workers that handle requests asynchronously.

In the scenario above where we are sending asynchronous requests, NGINX would also handle them asynchronously. This implies that when the code is checking the user's balance to ensure that it is enough to carry out the transaction, there is a possibility that all the requests would pass the validation check since the user's wallet is not updated until the first request is completed which is when the user balance would now be Zero since the requests are running at almost (or the same) the same time. Each of the requests would then go ahead to initiate a transfer from Tobexkee to Anis and create transaction records in the database. 
Since the requests are running concurrently, each request is not aware of the changes made by other requests until all the concurrent requests are completed.

### CAN THIS REALLY HAPPEN ? 
Companies have lost a good chunk of cash to malicious users who look for software susceptible to these attacks.  Seeing that the main idea behind this attack is to trigger the same request concurrently. In the case of this app where Tobekee is sending money to Anis, a user can take advantage of the system in any of the following ways
- send money from one account to another on a mobile and app version of the app at exactly the same time. 
- Send money from one account to another using different computers logged in to the same account at exactly the same time.
- If we have another means where the user wallet can be altered e.g airtime purchase, a user can try purchasing airtime on one device while trying to send money on another
- If a user is able to click on a button to send money to another account multiple times assuming the button was not disabled after the first click.
- There are tons of other ways depending on the uniqueness of your application.

### HANDLING CONCURRENCY ATTACKS IN LARAVEL
Laravel provides several solutions out of the box to handle concurrency attacks. Some of these include

#### Disabling accessing an account on multiple devices. 
One of the easiest means for a malicious user to exploit your an in this manner is when the user can be logged into a single account on multiple devices.  For mobile apps that use token-based authentication, Laravel Sanctum, Sanctum already provides an easy way to revoke all tokens belonging to a user [here](https://laravel.com/docs/9.x/sanctum#revoking-tokens). All the developer needs to do Is revoke existing tokens whenever a user logs in to the application. This way, the user would be logged out of any other device and can only use the app on a single device at a time. In the case of web applications, Laravel also provides the same solution [here](https://laravel.com/docs/9.x/authentication#invalidating-sessions-on-other-devices). 

#### Laravel Jobs and Queues
Queues are Laravel's way of running some tasks in the background of your application outside of the request-response cycle. You can read the official documentation on queues [here](https://laravel.com/docs/9.x/queues)

To demonstrate how queues can be applied here,  create a new job in Laravel app

`php artisan make:job UserTransactionManagerJob
`

The job class would need three arguments, the sender model, the receiver model and the amount to be transferred. 
```
<?php

namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\DB;

class UserTransactionManagerJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct(protected User $sender, protected User $reciever, protected float $amount)
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        if ($this->sender->balance() >= $this->amount) {

            DB::beginTransaction();

            $this->sender->transactions()->create([
                'amount' => -1 * $this->amount,
                'description' => "You sent {$this->amount} to {$this->reciever->name}"
            ]);

            $this->reciever->transactions()->create([
                'amount' => -1 * $this->amount,
                'description' => "You received {$this->amount} to {$this->sender->name}"
            ]);

            DB::commit();

        }
    }
}
```
As you can see above, the  logic for sending the money from the sender's wallet to the 
receiver's wallet has been moved into this job class. We can then go ahead to remove the logic from the controller and dispatch this job instead. The controller would now look like this 

```
<?php

namespace App\Http\Controllers;

use App\Http\Requests\SendMoneyRequest;
use App\Jobs\UserTransactionManagerJob;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\DB;

class SendMoneyController extends Controller
{
    public function __invoke(SendMoneyRequest $request)
    {
        $user = User::where('email', $request->get('email'))->first();
        
        UserTransactionManagerJob::dispatch(Auth::user(), $user, $request->get('amount'));

        return response()->json(['status' => 'success!']);
    }
}
```
 This does not solve our problem yet because each of the requests would just dispatch new instances of the job. However, laravel provides two powerful features in Job classes that can be leveraged to avoid concurrency attacks. 


First is the [ShouldBeUnique interface](https://laravel.com/docs/9.x/queues#unique-jobs) . Implementing this interface on a job class tells laravel to ensure that only
one instance of this job should be running at any given time. Hence at no point in time would two of this job run concurrently. Under the hood, what this does is to obtain a lock in the database for the job and this lock wont be released until the job is completed. Now while the job is running, any other instance of that job would not be dispatched. This implies that even if multiple requests are dispatching our UserTransactionManagerJob, we now have some kind of speed breaker that ensure that the jobs are queued and not dispatched while another instance of it is running. You can find the implementation in the laravel framework [here](https://github.com/laravel/framework/blob/df67f44f58fd7fbdbb05960c1497dcd30cdb7fd0/src/Illuminate/Foundation/Bus/PendingDispatch.php#L157). Now we most likely do not want to 
force only a single instance of this job running at any point in time while others are not dispatched because this cannot work in a real-world application where this job handles
such transaction for multiple users, we do not want only one transaction to be done at a time . The good news is laravel provides an optional uniqueId method. 

```
/**
* The unique ID of the job.
*
* @return string
*/
public function uniqueId()
{
    return $this->sender->id;
}
```
  
Adding this method to the Job class ensures that it makes use of this ID when acquiring the lock for this Job hence in this case now, we can have multiple instances 
of this job running at the same time but they cannot have the same ID. Jobs with an ID of a job already running would not be dispatched. 
One key thing to note though is that this would not work with jobs using the sync driver. you would need a driver that supports locks such as Redis, database etc. By running the money:send command again, we would now see that our code can now handle the concurrency request. We need to adjust our command class by introducing a sleep function to allow the jobs  run before fetching the 
user's balance.

```
public function handle()
{
    $responses = Http::pool(fn (Pool $pool) => [
        $pool->post('http://concurrency.test/api/send', [
        'email' => 'anis@mail.com',
        'amount' => 5000
        ]),
        $pool->post('http://concurrency.test/api/send', [
        'email' => 'anis@mail.com',
        'amount' => 5000
        ]),
        $pool->post('http://concurrency.test/api/send', [
        'email' => 'anis@mail.com',
        'amount' => 5000
        ]),

        $format = collect($responses)->map(fn ($response) => $response->throw()->json());

        sleep(5);
        dd($format, User::where('email', 'tobexkee@mail.com')->first()->balance());
    }
```
Running our command now yields
```
php artisan money:send
```

```
Illuminate\Support\Collection {#1433
  #items: array:3 [
    0 => array:1 [
      "status" => "success!"
    ]
    1 => array:1 [
      "status" => "success!"
    ]
    2 => array:1 [
      "status" => "success!"
    ]
  ]
  #escapeWhenCastingToString: false
}
"0.00"
```


So while the three requests were handled, only one triggered a successful transactions. 

The second feature laravel queues provide is the WithoutOverlapping middleware. This works similarly to the ShouldBeUnique interface except that it is not an interface. It is used to 
avoid two instances of the same job from running at the same time (overlapping). It also achieves this by using a lock under the hood. You can read the official documentation [here](https://laravel.com/docs/9.x/queues#preventing-job-overlaps). The implementation details can also be found [here](https://github.com/laravel/framework/blob/df67f44f58fd7fbdbb05960c1497dcd30cdb7fd0/src/Illuminate/Queue/Middleware/WithoutOverlapping.php)

 you only need to add the middleware method to your Job class

```
public function middleware()
{
    return [new WithoutOverlapping($this->user->id)];
}
```

The user Id being passed behaves like the uniqueId method mentioned earlier.
It ensures only instances with that particular Id do not overlap. 

Now this approach definitely would have it own drawbacks for instance what happens if a Unique Job fails or the transfer is happening over an API hence we do not want the API to be called more than once even when the job fails . However, there are several other great features that laravel queues provides that can help manage all of this edge cases. For instance you can  customize the logic of how failed jobs are handled, control the maximum times a job is retried or if it should even be retried etc. You can read more about them [here](https://laravel.com/docs/9.x/queues) 

### Conclusion 

Beyond a feature working, a good developer wants to ask questions like is the underlying code optimized
for efficiency and most importantly, is it secure? While some applications might not experience serious 
attacks from malicious users, others are soft targets for such attacks.
Generally, I believe any application that has an exit point for money is easily prone to 
general attacks from malicious users. This would include applications with wallet systems where users can withdraw funds or use them in making purchases. Hence such applications would require being extra cautious when taking care of security concerns. 
I am sure there is more to the subject that I might know, I look forward to your thoughts about this in the comment section. 

