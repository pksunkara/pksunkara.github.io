+++
date = 2020-03-30T00:00:00+01:00
draft = true
title = "State of routing in Rust"

[taxonomies]
tags = ["rust", "web"]
categories = ["programming"]
+++

There are many micro frameworks in Rust. Some famous examples are [Actix][actix], [Gotham][gotham], [Tide][],
[Warp][warp], etc. We have seen many blog posts comparing their performances and middleware capabilities.
But what we haven't seen is an article comparing their routing functionality. I hoped to fill that gap and
make this post the most up to date guide for the state of routing in Rust.

**NOTE**: I will not be comparing [Rocket](https://rocket.rs) because it does not run on *Stable* yet.

We have a lot of mature web frameworks like [Rails][rails], [Django][django], [Laravel][laravel],
[Phoenix][phoenix], etc.. that have been working on solving the routing problem and dealing with the
typical use cases from web developers. Before we start comparing, let's define the benchmark using these
frameworks.

### Versions

| Rails | Django | Laravel | Phoenix |
|:-:|:-:|:-:|:-:|
| 6.0.2 | 3.0.4 |

### Restricting Methods

Let us start the comparison with trivial stuff. Simplest use case is to restrict some of the routes to
only a few HTTP methods. There might be a shorthand syntax if only one HTTP method is allowed. The route
should return **405** HTTP status code if the none of the allowed HTTP methods are used.

###### Rails

```ruby
match 'photos', to: 'photos#create', via: [:post, :put]
```

###### Django

```python
path('photos', views.photos.create)
```

```python
@require_http_methods(["POST", "PUT"])
def create(request):
    pass
```

###### Laravel

```php
<?php
Route::match(['post', 'put'], '/photos', 'PhotosController@create')
```

###### Phoenix

```elixir
match :post, "/photos", PhotosController, :create
match :put, "/photos", PhotosController, :create
```

Or you can use [mux](https://github.com/christopheradams/multiplex)

```elixir
mux [:post, :put], "/photos", PhotosController, :create
```

<!-- returns 404 -->

### Path parameters

One of the first non-simplistic use case any web developer would look for is on how to use path parameters.

###### Rails

```ruby
get 'photos/:id', to: 'photos#show'
```

###### Django

```python
path('photos/<id>', PhotoResource.as_view())
```

###### Laravel

```php
<?php
Route::get('/photos/{id}', 'PhotosController@show')
```



[actix]: https://actix.rs
[gotham]: https://gotham.rs
[tide]: https://docs.rs/tide
[warp]: https://docs.rs/warp
[rails]: https://rubyonrails.org
[django]: https://djangoproject.com
[laravel]: https://laravel.com
[phoenix]: https://www.phoenixframework.org/



Supported and ETU | Not Supported | Not ETU

Path Params

​	Required

​		Laravel, Django | |

​	Optional

​		Laravel | | Django

​	Regex

​			Laravel, Django | |

​	Globs (tails)

​		Laravel, Django | |

Associations

Trailing Slashes

​	Resolve same by default

​		Laravel, Django | |

​	Redirect Middleware for SEO

​		Django | |

​	Can be separated

​		Django | |

Multi Methods

​	Django | |

Normalizing Slashes

​	Laravel | Django |

Method not allowed

​	Django | |
