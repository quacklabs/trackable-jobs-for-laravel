# Trackable Jobs For Laravel
![Trackable jobs for laravel](https://banners.beyondco.de/Laravel%20Trackable%20Jobs.png?theme=light&packageManager=composer+require&packageName=mateusjunges%2Flaravel-trackable-jobs&pattern=architect&style=style_1&description=This+package+allows+you+to+track+your+laravel+jobs%21&md=1&showWatermark=1&fontSize=100px&images=https%3A%2F%2Flaravel.com%2Fimg%2Flogomark.min.svg)

[![Latest Version on Packagist](https://img.shields.io/packagist/v/mateusjunges/laravel-trackable-jobs.svg?style=flat)](https://packagist.org/packages/mateusjunges/laravel-trackable-jobs)
[![Total Downloads](https://img.shields.io/packagist/dt/mateusjunges/laravel-trackable-jobs.svg?style=flat)](https://packagist.org/packages/mateusjunges/laravel-trackable-jobs)
[![MIT Licensed](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat)](LICENSE.md)
[![StyleCI](https://github.styleci.io/repos/355262680/shield?style=flat)](https://styleci.io/repos/355262680)
![](https://github.com/mateusjunges/trackable-jobs-for-laravel/actions/workflows/run-tests.yml)

This package allows you to track your laravel jobs!
Using this package, you can easily persist the output and the status of any job in your application.

- [1. Installation](#installation)
- [2. Usage](#usage)
    - [2.1 Tracking jobs](#tracking-jobs)
    - [2.2 Tracking job chains](#tracking-job-chains)
    - [2.3 Extending the `TrackedJob` model](#extending-the-trackedjob-model)
- [3. Tests](#tests)
- [4. Contributing](#contributing)
- [5. Changelog](#changelog)
- [6. Credits](#credits)
- [7. License](#license)

# Installation
To install this package, use composer:
```bash
composer require mateusjunges/laravel-trackable-jobs
```

You can publish the configuration file with this command:

```bash
php artisan vendor:publish --tag=trackable-jobs-config
```

Now you are good to go!

# Usage
## Tracking jobs
To start tracking your jobs, you just need to use the `Junges\TrackableJobs\Traits\Trackable` trait in the job you want to track.
For example, let's say you want to track the status of `ProcessPodcastJob`, just add the `Trackable` trait into your job:

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Junges\TrackableJobs\Traits\Trackable;

class ProcessPodcastJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels, Trackable;

    public function handle()
    {
        //
    }
}
```

This trait provides 3 methods to your job: `__construct`, `failed` and `middleware`. It also adds a `model` public property to the job class.
If you want to override any of the methods, you must copy and paste (because you can't use `parent` for traits) the content of each one inside your class,
so this package still work as intended.

For example: if you need to change the constructor of your job, copy the code from `Junges\TrackableJobs\Traits\Trackable` to your new constructor:

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Junges\TrackableJobs\Traits\Trackable;
use App\Models\Podcast;
use Junges\TrackableJobs\Models\TrackedJob;

class ProcessPodcastJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels, Trackable;

    public function __construct(Podcast $podcast)
    {
         $this->model = $podcast;
         
         $this->trackedJob = TrackedJob::create([
            'trackable_id'   => $this->model->id,
            'trackable_type' => get_class($this->model),
            'name'           => class_basename(static::class),
         ]);
         
         // Add your code here.
    }

    public function handle()
    {
        //
    }
}
```

This package will store the last status of your job, which can be `queued`, `started`, `failed` or `finished`. Also, it stores the 
`started_at` and `finished_at` timestamps for each tracked job.

To use it, you just need to pass any model to your Job constructor:

```php
dispatch(new ProcessPodcastJob($podcast));
```

Once this trait is added to your job, your job progress will be persisted to the database. You can configure the table name by publishing this package configuration file:

```shell
php artisan vendor:publish --tag=trackable-jobs-config
```

This command will create a new config file in `config/trackable-jobs.php`, with this content:

```php
<?php

return [
    /*
     | The table where the tracked jobs will be stored.
     | By default, it's called 'tracked_jobs'.
     */
    'tables' => array(
        'tracked_jobs' => 'tracked_jobs',
    ),
];
```


## Tracking job chains
Laravel supports job chaining out of the box:

```php
Bus::dispatchChain([
    new OptimizePodcast($podcast),
    new CompressPodcast($podcast),
    new ReleasePodcast($podcast)
])->dispatch();
```

It's a nice, fluent way of saying "Run this jobs sequentially, one after the previous one is complete.".

If you have a task which takes some steps to be completed, you can track the job chain used to do that and know the 
status for each job. 
If you are releasing a new podcast, for example, and it has to be optimized, compressed and released, you can track 
this steps by adding a `steps` relationship to your `Podcast` model:

```php
public function steps()
{
    return $this->morphMany(Junges\TrackableJobs\Models\TrackedJob::class, 'trackable');
}
```

Now, you can have the status of each job that should be processed to release your podcast:

```php
$steps = Podcast::find($id)->steps()->get();
```

## Extending the `TrackedJob` model.
If, for some reason, you need to use your own custom model to the TrackedJob table, you can just create a new model
and extend the existing `Junges\TrackableJobs\Models\TrackedJob::class`.
Then, you need to bind the `Junges\TrackableJobs\Contracts\TrackableJobContract` to the new model, within your `AppServiceProvider`:

```php
<?php

namespace App\Providers;

use App\Models\YourCustomModel;
use Illuminate\Support\ServiceProvider;
use Junges\TrackableJobs\Contracts\TrackableJobContract;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(TrackableJobContract::class, YourCustomModel::class);
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}
```

# Tests
Run `composer test` to test this package.

# Contributing
Thank you for consider contributing for the Laravel Trackable Jobs package! The contribution guide can
be found [here][contributing].

# Changelog
Please see the [changelog][changelog] for more information about the changes on this package.

# Credits
- [All contributors][contributors]

# License
The laravel trackable jobs is open-sourced software licensed under the terms of [MIT License][mit]. Please see the [license file][license] for more information.

[contributing]: CONTRIBUTING.md
[changelog]: CHANGELOG.md
[mit]: https://opensource.org/licenses/MIT
[license]: LICENSE
[contributors]: https://github.com/mateusjunges/trackable-jobs-for-laravel/graphs/contributors
