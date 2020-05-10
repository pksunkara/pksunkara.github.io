+++
date = 2020-05-10T00:00:00+02:00
draft = true
title = "State of routing in Rust"

[taxonomies]
tags = ["rust", "web"]
categories = ["programming"]
+++

There are many micro frameworks in Rust. Some famous examples are [Actix][], [Gotham][], [Tide][], [Warp][],
etc. We have seen many blog posts comparing their performances and middleware capabilities.
But what we haven't seen is an article comparing their routing functionality and capabilities. I hoped to
fill that gap and make this post the most up to date guide for the state of routing in Rust.

**NOTE**: I will not be comparing [Rocket](https://rocket.rs) because it does not run on *Stable* yet.

We have a lot of mature web frameworks like [Rails][], [Django][], [Laravel][], etc..
that have been working on solving the routing problem and dealing with the typical use cases from web
developers. Let us define the benchmak using these frameworks and compare the rust micro frameworks to
this benchmark.

* Rails `v6.0`, Django `v3.0`, Laravel `v7.6`
* Actix `v3.0.0-alpha.2`, Gotham `v0.5.0-rc.0`, Tide `v0.8.1`, Warp `v0.2.2`

## Restricting Methods

Let us start the comparison with trivial stuff. Simplest use case is to restrict some of the routes to
only a few HTTP methods. There might be a shorthand syntax if only one HTTP method is allowed. The route
should return **405** HTTP status code if the none of the allowed HTTP methods are used.

#### Rails

```ruby
match '/orgs', to: 'orgs#create', via: [:post, :put]
```

<!-- returns 404 instead of 405 -->

#### Django

```python
path('orgs', views.orgs.create),
```

```python
@require_http_methods(["POST", "PUT"])
def create(request):
    pass
```

#### Laravel

```php
<?php
Route::match(['post', 'put'], '/orgs', 'OrgsController@create');
```

#### Actix

```rust
app.service(
    resource("/orgs")
        .route(post().to(orgs::create))
        .route(put().to(orgs::create))
);
```

Cons

* This does not return **405** but instead returns **404**.
* There is no shorter way to specify multiple allowed methods on the same route for the same handler.

#### Tide

```rust
app.at("/orgs").post(orgs::create).put(orgs::create);
```

Cons

* There is no shorter way to specify multiple allowed methods on the same route for the same handler.

#### Warp

```rust
path!("orgs").and(post().or(put()).unify()).and_then(orgs::create);
```

#### Gotham

```rust
route.request(vec![POST, PUT], "/orgs").to(orgs::create);
```

## Path parameters

One of the first non-simplistic use case any web developer would look for is on how to use path parameters.
The path params can be specified as either *required* or *optional*. They should also be able to be specified
in the form of *regex patterns*. There might be a shorter syntax for *glob (tail match)* path param.

#### Rails

```ruby
# required
get '/orgs/:id', to: 'orgs#show'
# optional
get '/repos(/:id)', to: 'repos#show'
# regex
get '/users/:id', to: 'users#show', id: /[a-z]+/
# glob
get '/blob/*path', to: 'repo_blob#show'
# optional regex
get '/search(/:id)', to: 'search#show', id: /[a-z]+/
# optional glob
get '/tree(/*path)', to: 'repo_tree#show'
```

#### Django

```python
# required
path('orgs/<id>', views.orgs.show),
# optional
path('repos', views.repos.show),
path('repos/<id>', views.repos.show),
# regex
re_path('^users/(?P<id>[a-z]+)$', views.users.show),
# glob
re_path('^blob/(?P<path>.+)$', views.repo_blob.show),
# optional regex
re_path('^search(/(?P<id>[a-z]+))?$', views.search.show),
# optional glob
re_path('^tree(/(?P<path>.+))?$', views.repo_tree.show),
```

#### Laravel

```php
<?php
# required
Route::get('/orgs/{id}', 'OrgsController@show');
# optional
Route::get('/repos/{id?}', 'ReposController@show');
# regex
Route::get('/users/{id}', 'UsersController@show')->where('id', '[a-z]+');
# glob
Route::get('/blob/{path}', 'RepoBlobController@show')->where('path', '.*');
# optional regex
Route::get('/search/{id?}', 'SearchController@show')->where('id', '[a-z]+');
# optional glob
Route::get('/tree/{path?}', 'RepoTreeController@show')->where('path', '.*');
```

#### Actix

```rust
// required
app.route("/orgs/{id}", get().to(orgs::show));

// optional
app.route("/repos/{id}", get().to(repos::show))
   .route("/repos", get().to(repos::show));

// regex
app.route("/users/{id:[a-z]+}", get().to(users::show));

// glob
app.route("/blob/{path:.+}", get().to(repo_blob::show));

// optional regex
app.route("/search/{id:[a-z]+}", get().to(search::show))
   .route("/search", get().to(search::show));

// optional glob
app.route("/tree/{path:.+}", get().to(repo_tree::show))
   .route("/tree", get().to(repo_tree::show));
```

Cons

* There is no shorter way to specify optionals. We could have used `app.service(resource(..))` as mentioned
  in [#1054](https://github.com/actix/actix-web/issues/1054) to make it a little bit shorter if not full, but
  there seems to be a bug where all following routes which uses one of the specified routes here as a prefix
  (with or without `/` after the prefix) doesn't work.

#### Tide

```rust
// required
app.at("/orgs/:id").get(orgs::show);

// optional
app.at("/repos").get(repos::show);
app.at("/repos/:id").get(repos::show);

// glob
app.at("/blob/*path").get(repo_blob::show);

// optional glob
app.at("/tree").get(repo_tree::show);
app.at("/tree/*path").get(repo_tree::show);
```

Cons

* **BUG**: Panics when the optional param is not present in the path instead of forwarding the error.
* **MAJOR**: Does not support regex path paramerter.
* There is no shorter way to specify optionals.

#### Warp

```rust
// required
path!("orgs" / String).and(get()).and_then(orgs::show);

// optional
path!("repos").and(get()).and_then(|| repos::show(Default::default()));
path!("repos" / String).and(get()).and_then(repos::show);

// regex
struct Id(String);

impl FromStr for Id {
    type Err = ();
    fn from_str(s: &str) -> Result<Id, ()> {
        match Regex::new("^[a-z]+$").unwrap().is_match(s) {
            true => Ok(Id(s.to_string())),
            _ => Err(())
        }
    }
}

path!("users" / Id).and(get()).and_then(users::show);

// optional regex
path!("search").and(get()).and_then(|| search::show(Id(Default::default())));
path!("search" / Id).and(get()).and_then(search::show);

// optional glob
path("tree").and(tail()).and(end()).and(get()).and_then(repo_tree::show);
```

Cons

* **MINOR**: Does not support required glob scenario.
* There is no shorter way to specify regex path params. Even though types that implement `FromStr` are able
  to be used directly without the need for regex, it would be nice to have a regex filter.
* There is not shorter way to specify optionals. They can be made a little bit shorter if not full if we
  implement dummy default filters for each param type. I think [warp][] in general needs a few dummy filters.

#### Gotham

```rust
// required
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct IdExtractor {
    id: String,
}

route.get("/orgs/:id").with_path_extractor::<IdExtractor>().to(orgs::show);

// optional
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct OptIdExtractor {
    id: Opt<String>,
}

route.get("/repos").with_path_extractor::<OptIdExtractor>().to(repos::show);
route.get("/repos/:id").with_path_extractor::<OptIdExtractor>().to(repos::show);

// regex
route.get("/users/:id:[a-z]+").with_path_extractor::<IdExtractor>().to(users::show);

// glob
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct PathExtractor {
    path: Vec<String>,
}

route.get("/blob/*").with_path_extractor::<PathExtractor>().to(repo_blob::show);

// optional regex
route.get("/search").with_path_extractor::<OptIdExtractor>().to(users::show);
route.get("/search/:id:[a-z]+").with_path_extractor::<OptIdExtractor>().to(users::show);

// optional glob
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct OptPathExtractor {
    path: Option<Vec<String>>,
}

route.get("/tree").with_path_extractor::<OptPathExtractor>().to(repo_tree::show);
route.get("/tree/*").with_path_extractor::<OptPathExtractor>().to(repo_tree::show);
```

Cons

* There is no shorter way to specify optionals.
* The usage of path params is a little bit more verbose and is thus not easy to use.

[Actix]: https://actix.rs
[Gotham]: https://gotham.rs
[Tide]: https://docs.rs/tide
[Warp]: https://docs.rs/warp
[Rails]: https://rubyonrails.org
[Django]: https://djangoproject.com
[Laravel]: https://laravel.com

> Supported and ETU | Not Supported | Not ETU
>
> Associations
>
> Trailing Slashes
>
> ​	Resolve same by default
>
> ​		Laravel, Django | |
>
> ​	Redirect Middleware for SEO
>
> ​		Django | |
>
> ​	Can be separated
>
> ​		Django | |
>
> Normalizing Slashes
>
> ​	Laravel | Django |
