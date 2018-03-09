# Cursor Pagination for Laravel
[![Latest Version](https://img.shields.io/github/release/juampi92/cursor-pagination.svg?style=flat-square)](https://github.com/juampi92/cursor-pagination/releases)
[![Build Status](https://img.shields.io/travis/juampi92/cursor-pagination/master.svg?style=flat-square)](https://travis-ci.org/juampi92/cursor-pagination)
[![StyleCI](https://styleci.io/repos/122583097/shield?branch=master)](https://styleci.io/repos/122583097)
[![Total Downloads](https://img.shields.io/packagist/dt/juampi92/cursor-pagination.svg?style=flat-square)](https://packagist.org/packages/juampi92/cursor-pagination)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)

This package provides a cursor based pagination already integrated with Laravel's [query builder](https://laravel.com/docs/master/queries) and [Eloquent ORM](https://laravel.com/docs/master/eloquent).
It calculates the SQL query limits automatically by checking the requests GET parameters, and automatically builds
the next and previous urls for you.

## Installation

You can install this package via composer using:

```bash
composer require juampi92/cursor-pagination
```

The package will automatically register itself.

### Config

To publish the config file to `config/cursor_pagination.php` run:

````bash
php artisan vendor:publish --provider="Juampi92\CursorPagination\CursorPaginationServiceProvider" --tag="config"
````

This will publish the following [file](config/cursor_pagination.php). 
You can customize the name of the GET parameters, as well as the default items per page. 

## How does it work

The main idea behind a cursor pagination is that it needs a context to know what results to show next.
So instead of saying `page=2`, you say `next_cursor=10`. The result is the same as the old fashioned page pagination,
but now you have more control on the output, cause `next_cursor=10` should always return the same (unless some records are deleted).
 
This kind of pagination is useful when you sort using the latest first, so you can keep an infinite scroll using `next_cursor` whenever you get to the bottom,
or do a `previous_cursor` whenever you need to refresh the list at the top, and if you send the first cursor,
the results will only fetch the new ones.

#### Pros
 - New rows don't affect the result, so no duplicated results when paginating.
 - Filtering by an indexed cursor is way faster that using database offset.
 - Using previous cursor to avoid duplicating the first elements.
 
#### Cons
 - No previous page, although the browser still has them.
 - No navigating to arbitrary pages (you must know the previous result to know the next ones).

Basically, **it's perfect for infinite scrolling!**

## Basic Usage

### Paginating Query Builder Results
    
There are several ways to paginate items. 
The simplest is by using the `cursorPaginate` method on the **query builder** or an **Eloquent query**. 
The `cursorPaginate` method automatically takes care of setting the proper limit and fetching the next or 
previous elements based on the cursor being viewed by the user.
By default, the `cursor` is detected by the value of the page query string argument on the HTTP request.
This value is automatically detected by the package taking your custom config into account, and is also automatically inserted into links and meta generated by the paginator.

````php
public function index()
{
    $users = DB::table('users')->cursorPaginate();
    return $users;
}
````

### Paginating Eloquent Results

You may also paginate **Eloquent queries**. In this example, we will paginate the `User` model 
with `15` items per page. As you can see, the syntax is identical to paginating query builder results:

````php
$users = User::cursorPaginate(15);
````

Of course, you may call paginate after setting other constraints on the query, such as where clauses:

````php
$users = User::where('votes', '>', 100)->cursorPaginate(15);
````

Or sorting your results:

````php
$users = User::orderBy('id', 'desc')->cursorPaginate(15);
````

Don't worry, the package will detect if the model's primary key is being used for sorting, and it will adapt
itself so the next and previous work as expected.

### Identifier

The paginator identifier is basically the cursor attribute. The model's attribute that will be used for paginating.
It's really important that this identifier is **unique** on the results. Duplicated identifiers could cause that some
records are not showed, so be careful.

*If the query is not sorted by the identifier, the cursor pagination might not work as expected.*

####  Auto-detecting Identifier

If no identifier is defined, the cursor will try to figure it out by itself. First, it will check if there's any orderBy clause.
If there is, it will take the **first column** that is sorted and will use that.
If there is not any orderBy clause, it will check if it's an Eloquent model, and will use it's `primaryKey` (by default it's 'id').
And if it's not an Eloquent Model, it'll use `id`.

##### Example

````php
// Will use Booking's primaryKey
Bookings::cursorPaginate(10);
````

````php
// Will use hardcoded 'id'
DB::table('bookings')->cursorPaginate(10);
````

````php
// Will use 'created_by'
Bookings::orderBy('created_by', 'asc')
    ->cursorPaginate(10);
````

#### Customizing Identifier

Just define the `identifier` option

````php
// Will use _id, ignoring everything else.
Bookings::cursorPaginate(10, ['*'], [
    'identifier'      => '_id'
]);
````

### Date cursors

By default, the identifier is the model's primaryKey (or `id` in case it's not an Eloquent model) so there are no duplicates,
but you can adjust this by passing a custom `identifier` option. In case that specified identifier is casted to date or datetime
on Eloquent's `$casts` property, the paginator will convert it into **unix timestamp** for you.

You can also specify it manually by adding a `[ 'date_identifier' => true ]` option.

##### Example

Using Eloquent (make sure Booking Model has ` protected $casts = ['datetime' => 'datetime']; `)

````php
// It will autodetect 'datetime' as identifier,
//   and will detect it's casted to datetime.
Bookings::orderBy('datetime', 'asc')
    ->cursorPaginate(10);
````

Using Query
````php
// It will autodetect 'datetime' as identifier,
//   but since there is no model, you'll have to
//   specify the 'date_identifier' option to `true`
DB::table('bookings')
    ->orderBy('datetime', 'asc')
    ->cursorPaginate(10, ['*'], [
        'date_identifier' => true
    ]);
````

### Inherits Laravel's Pagination

You should know that CursorPaginator inherits from Laravel's AbstractPaginator, so methods like
`withPath`, `appends`, `fragment` are available, so you can use those to build the url,
but you should know that the default url creation of this package includes query params that you
might already have.

## Displaying Pagination Results

### Converting to JSON

A basic return will transform the paginator to JSON and will have a result like this:

````php
Route::get('api/v1', function () {
    return App\User::cursorPaginate();
});
````

Calling `api/v1` will output:

````json
{
   "path": "api/v1?",
   "previous_cursor": "10",
   "next_cursor": "3",
   "per_page": 3,
   "next_page_url": "api/v1?next_cursor=3",
   "prev_page_url": "api/v1?previous_cursor=1",
   "data": [
        {}
   ]
}
````

### Using a Resource Collection

By default, Laravel's API Resources when using them as collections, they will output a paginator's metadata
into `links` and `meta`.

````json
{
   "data":[
        {}
   ],
   "links": {
       "first": null,
       "last": null,
       "prev": "api/v1?previous_cursor=1",
       "next": "api/v1?next_cursor=3"
   },
   "meta": {
       "path": "api/v1?",
       "previous_cursor": "1",
       "next_cursor": "3",
       "per_page": 3
   }
}
````
 
## Understanding the previous cursor

It's important to clarify that the previous cursor does NOT work like the next one.
The next cursor makes the paginator return the elements that follow that cursor.

So in the case of the next cursor being `10`, that pagination should return `next + 1`, `next + 2`, `next + 3`.
So that works as expected.

The previous cursor though it's not `prev - 1`, `prev - 2`, `prev - 3`. No. It does not return the adjacent elements, or the 'context' for that cursor.
What it does instead is pretty interesting:

Let's assume we do a simple fetch and it returns elements `[10, 9, 8]`. If we do a `prev=10`, it will return empty, cause
those are the first elements. So if we now add 5 more to that: from the 15 to the 11, and we do `prev=10` again, the
result will be `[15, 14, 13]`. **NOT** `[13, 12 ,11]`.

That's because previous works like a filter. It shows the first 'per_page' items that are before that cursor.
The order is the same as in the next cursor, so it fetches the latest ones.

The same logic can be used when combining cursors: `next=13&prev=10` will output the missing two elements: `[12, 11]`, so you can complete the list without fetching any extra items.

## Customizing per page results

With this plugin you can specify the perPage number by many ways.

You can set the `cursor_pagination.per_page` config, so if you don't send any parameters, it will use that as the default.

You can also use an array. The purpose of the array declaration is to separate whenever it's a previous cursor, or a next / default cursor.
This way, when you do a *'refresh'*, you can fetch more results than if you are simply scrolling.
 
To configure that, set `$perPage = [15, 5]`. That way it'll fetch 15 when you do a `previous_cursor`, and 5 when you do a normal fetch or a `next_cursor`. 

## API Docs

### Paginator custom options

````php
new CursorPaginator(array|collection $items, array|int $perPage, array options = [
    // Attribute used for choosing the cursor. Used primaryKey on Eloquent Models as default.
    'identifier'      => 'id',
    'date_identifier' => false,
    'path'            => request()->path(),
]);
````

(*The items must have a $item->{$identifier} property.*)

Eloquent Builder and Query Builder's macro:

````php
cursorPaginate(array|int $perPage, array $cols = ['*'], array options = []): CursorPaginator;
````

### Paginator methods

 ````php
 $resutls->hasMorePages(): bool;
 $results->nextCursor(): string|null;
 $results->prevCursor(): string|null;
 $results->previousPageUrl(): string|null;
 $results->nextPageUrl(): string|null;
 $results->url(['next' => 1]): string;
 $results->count(): int;
 ````
  
 Note: all cursors are casted as strings.

## Testing

Run the tests with:
```bash
vendor/bin/phpunit
```

## Credits

- [Juan Pablo Barreto](https://github.com/juampi92)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
