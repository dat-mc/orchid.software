---
title: Attachments
description: Learn how to use Laravel Orchid's Attachments feature to manage and attach files to your application's database. Improve your app's user experience and organization with this powerful tool.
---


Attachments are files of various formats and extensions related to a recording. They can be attached to any model in your application by adding the `Attachable` trait to the model and using the `attachment()` relationship.

For example, to attach files to a `Hostel` model:


```php
namespace App;

use Illuminate\Database\Eloquent\Model;
use Orchid\Attachment\Attachable;

class Hostel extends Model
{
    use Attachable;
    //
}
```

To retrieve the attachments for a particular hostel, you can use the `attachment()` relationship:

```php
$item = Hostel::find(42);
$item->attachment()->get();
```


## Uploading a File via HTTP

To upload a file via HTTP, you can use the `File` class and the `load()` method. Here's an example of a controller method that handles file uploads:


```php
use Orchid\Attachment\File;

public function upload(Request $request)
{
    $file = new File($request->file('photo'));
    $attachment = $file->load();

    return response()->json()
}
```

This will automatically upload the file to the default repository (usually `public`) and create an entry in the database.

To retrieve the URL of an attachment, you can use the `url()` method:

```php
$image = $item->attachment()->first();

// Get the URL of the file
$image->url();
```

> **Note.** The `url()` method will first check for the path existence, and then get the URL. When using external storage like `s3`, this will make two calls. To improve performance you can use the [caching adapter](https://laravel.com/docs/filesystem#driver-prerequisites) recommended by `Laravel` to improve performance. You can also simply override this method and adjust to your needs.


## Uploading a File via the Console

Sometimes the necessary files are already on the server, and you can use the following code to upload them to the desired storage:

```php
use Illuminate\Http\UploadedFile;
use Orchid\Attachment\File;

$file = new UploadedFile($path, $originalName);

$attachment = (new File($file))->load();
```

## Duplicate Uploaded Files

Thanks to the use of hashes, attachments are not uploaded again if they already exist in the storage. Instead, a link is created in the database to the existing physical file, allowing for efficient use of resources. The file will only be deleted when all links to it are destroyed.


### Allowing Duplicate Files

In some situations, you might want to keep duplicate files (files that have the same hash) and generate different links to request different physical files. The file will be deleted when the link is destroyed. To allow duplicate files, you can use the `allowDuplicates()` method:

```php
use Orchid\Attachment\File;

public function upload(Request $request)
{
    $file = new File($request->file('photo'));
    $attachment = $file->allowDuplicates()->load();
    return response()->json()
}   
```

## Customizing the Upload Path


By default, Orchid has a default upload path for all files of `Y/m/d`, for example: `2022/06/11`. You can change the default path using the `path(string $path)` method:

```php
use Orchid\Attachment\File;

public function upload(Request $request)
{
    $path = "photos"
    $file = new File($request->file('photo'));
    $attachment = $file->path($path)->load();
    return response()->json()
}
```

## Remove

Attachments won't be removed after model removal automatically. In case when your attachments can't exist without a model, you should remove them on model `deleting` events manually. If you delete a row from the `attachments` table, the file won't be deleted. To clear your attachments, you need to use `delete()` function on the `Attachment` model. In that case, an additional check will proceed. If there no link to the file - it will be deleted. You can do it using [relationships](https://laravel.com/docs/master/eloquent-relationships) and [observers](https://laravel.com/docs/master/eloquent#observers).

Let's come back to our example with `hero` relation from ["Manage file attachments"](/en/docs/quickstart-files)

```php
// app/Post.php

use Orchid\Attachment\Models\Attachment;

public function hero()
{
    return $this->hasOne(Attachment::class, 'id', 'hero')->withDefault();
}
```

If you call your relation like function `$post->hero()` it will return `Illuminate\Database\Eloquent\Builder` class, that also has `delete()` function. But, it will execute sql query. If you call your relation like attribute `$post->hero` it will return model class. `Attachment` model class.

```php
$post->hero()->delete();
```

> **Note.** You should build your relation using `withDefault()` function to avoid the null pointer exception.

Let's generate [observer](https://laravel.com/docs/master/eloquent#observers) for our example model.

```bash
php artisan make:observer PostObserver
```

In PostObserver we create `deleting` function

```php
public function deleting(Post $post)
{
    $post->hero()->delete();
}
```

When you have multiple attachments, you should use `attachment` relation from the `Attachable` trait.

```php
public function deleting(Post $post)
{
    //load attachment as collection and not query attachment()
    $post->attachment->each->delete();
}
```

> **Note.** An experienced Laravel developer will see that there is an `N+1` problem here. It is intentionally done to access the filesystem to delete a file (The database won't do it for us). 


Subscribe example model to the observer in `AppServiceProvider`

```php
public function boot()
{
    ...
    
    Post::observe(PostObserver::class);
}
```


## Default Configuration


By default, each file uploaded follows the strategy described in `config/platform.php`:

```php
/*
|--------------------------------------------------------------------------
| Default configuration for attachments.
|--------------------------------------------------------------------------
|
| Strategy properties for the file and storage used.
|
*/

'attachment' => [
    'disk'      => 'public',
    'generator' => \Orchid\Attachment\Engines\Generator::class,
],
```

- **disk** - The name of the storage used to store files. The entire settings for the storage should be defined in `/config/filesystems.php`.

- **generator** - A class that defines how the uploaded files will be named, in which directories they will be located, and how to avoid duplicating them.


## Event subscription

Different file processing options may require additional processing, such as video compression,
This is possible thanks to an event that you can subscribe to using standard tools and perform a task in the background:

```php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use App\Listener\UploadListener;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event handler mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        UploadFileEvent::class => [
             UploadListener::class,
        ],
    ];

    /**
     * Register any events for your application.
     */
    public function boot()
    {
        parent::boot();
    }
}
```

Each subscription will receive an object `UploadFileEvent`:

```php
namespace App\Listeners;

use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Orchid\Platform\Events\UploadFileEvent;

class UploadListener extends ShouldQueue
{
    use InteractsWithQueue;
    
    /**
     * Handle the event.
     *
     * @param  UploadFileEvent  $event
     * @return void
     */
    public function handle(UploadFileEvent $event)
    {
        //$event->attachment
        //$event->time
    }
}
``` 