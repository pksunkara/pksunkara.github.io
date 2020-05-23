+++
date = 2020-05-17T00:00:00+02:00
draft = false
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
developers. Let us define the baseline using these frameworks and compare the Rust web micro frameworks to
this baseline.

* Rails `v6.0`, Django `v3.0`, Laravel `v7.6`
* Actix `v3.0.0-alpha.2`, Gotham `v0.5.0-rc.1`, Tide `v0.8.1`, Warp `v0.2.2`

## Restricting Methods

Let us start the comparison with trivial stuff. Simplest use case is to restrict some of the routes to
only a few HTTP methods. There might be a shorthand syntax if only one HTTP method is allowed. The route
should return **405** HTTP status code if the none of the allowed HTTP methods are used.

#### Rails

```ruby
match 'orgs', to: 'orgs#create', via: [:post, :put]
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
Route::match(['post', 'put'], 'orgs', 'OrgsController@create');
```

#### Actix

```rust
app.service(
    resource("orgs")
        .route(post().to(orgs::create))
        .route(put().to(orgs::create))
);
```

Cons

* This does not return **405** but instead returns **404**.
* There is no shorter way to specify multiple allowed methods on the same route for the same handler.

#### Tide

```rust
app.at("orgs").post(orgs::create).put(orgs::create);
```

Cons

* There is no shorter way to specify multiple allowed methods on the same route for the same handler.

#### Warp

```rust
path!("orgs").and(post().or(put()).unify()).and_then(orgs::create);
```

#### Gotham

```rust
route.request(vec![POST, PUT], "orgs").to(orgs::create);
```

## Path parameters

One of the first non-simplistic use case any web developer would look for is on how to use path parameters.
The path params can be specified as either *required* or *optional*. They should also be able to be specified
in the form of *regex patterns*. There might be a shorter syntax for *glob (tail match)* path param.

#### Rails

```ruby
# required
get 'orgs/:id', to: 'orgs#show'
# optional
get 'repos(/:id)', to: 'repos#show'
# regex
get 'users/:id', to: 'users#show', id: /[a-z]+/
# optional regex
get 'search(/:id)', to: 'search#show', id: /[a-z]+/
# glob
get 'blob/*path', to: 'repo_blob#show'
# optional glob
get 'tree(/*path)', to: 'repo_tree#show'
# glob in middle
get 'blob/*path/commits', to: 'repo_blob#commits'
# optional glob in middle
get 'tree(/*path)/commits', to: 'repo_tree#commits'
```

#### Django

```python
# required
path('orgs/<id>', views.orgs.show),
# optional
re_path('^repos(/(?P<id>[^/]+))?$', views.repos.show),
# regex
re_path('^users/(?P<id>[a-z]+)$', views.users.show),
# optional regex
re_path('^search(/(?P<id>[a-z]+))?$', views.search.show),
# glob
re_path('^blob/(?P<path>.+)$', views.repo_blob.show),
# optional glob
re_path('^tree(/(?P<path>.+))?$', views.repo_tree.show),
# glob in middle
re_path('^blob/(?P<path>.+)/commits$', views.repo_blob.commits),
# optional glob in middle
re_path('^tree(/(?P<path>.+))?/commits$', views.repo_tree.commits),
```

#### Laravel

```php
<?php
# required
Route::get('orgs/{id}', 'OrgsController@show');
# optional
Route::get('repos/{id?}', 'ReposController@show');
# regex
Route::get('users/{id}', 'UsersController@show')->where('id', '[a-z]+');
# optional regex
Route::get('search/{id?}', 'SearchController@show')->where('id', '[a-z]+');
# glob
Route::get('blob/{path}', 'RepoBlobController@show')->where('path', '.*');
# optional glob
Route::get('tree/{path?}', 'RepoTreeController@show')->where('path', '.*');
# glob in middle
Route::get('blob/{path}/commits', 'RepoBlobController@commits')->where('path', '.*');
# optional glob in middle
Route::get('tree/{path?}/commits', 'RepoTreeController@commits')->where('path', '.*');
```

#### Actix

```rust
// required
app.route("orgs/{id}", get().to(orgs::show));

// optional
app.route("repos", get().to(repos::show))
   .route("repos/{id}", get().to(repos::show));

// regex
app.route("users/{id:[a-z]+}", get().to(users::show));

// optional regex
app.route("search", get().to(search::show))
   .route("search/{id:[a-z]+}", get().to(search::show));

// glob
app.route("blob/{path:.+}", get().to(repo_blob::show));

// optional glob
app.route("tree", get().to(repo_tree::show))
   .route("tree/{path:.+}", get().to(repo_tree::show));

// glob in middle
app.route("blob/{path:.+}/commits", get().to(repo_blob::commits));

// optional glob in middle
app.route("tree/commits", get().to(repo_tree::commits))
   .route("tree/{path:.+}/commits", get().to(repo_tree::commits));
```

Cons

* There is no shorter way to specify optionals. We could have used `app.service(resource(..))` as mentioned
  in [#1054](https://github.com/actix/actix-web/issues/1054) but `app.service` stops matching anything else
  if the beginning of the URL matches the route given to it. All following routes which uses one of the
  specified routes here as a prefix (with or without `/` after the prefix) doesn't work.

#### Tide

```rust
// required
app.at("orgs/:id").get(orgs::show);

// glob
app.at("blob/*path").get(repo_blob::show);

// glob in middle
app.at("blob/*path/commits").get(repo_blob::commits);
```

Cons

* **MAJOR**: Does not support optionals.
* **MAJOR**: Does not support regex path param.

#### Warp

```rust
// required
path!("orgs" / String).and(get()).and_then(orgs::show);

// optional
path!("repos").and(get()).and_then(|| repos::show(Default::default()));
path!("repos" / String).and(get()).and_then(repos::show);

// regex
#[derive(Default)]
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
path!("search").and(get()).and_then(|| search::show(Id::default()));
path!("search" / Id).and(get()).and_then(search::show);

// optional glob
path("tree").and(tail()).and(end()).and(get()).and_then(repo_tree::show);
```

Cons

* **MINOR**: Does not support required glob path param.
* **MINOR**: Does not support anything after optional glob path param.
* There is no shorter way to specify regex path params. Even though types that implement `FromStr` are able
  to be used directly without the need for regex, it would be nice to have a regex filter.
* There is not shorter way to specify optionals. They can be made a little bit shorter if not full if we
  implement dummy default filters for each param type. I think [Warp][] in general needs a few dummy filters.

#### Gotham

```rust
// required
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct IdExtractor {
    id: String,
}

route.get("orgs/:id").with_path_extractor::<IdExtractor>().to(orgs::show);

// optional
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct OptIdExtractor {
    id: Opt<String>,
}

route.get("repos").with_path_extractor::<OptIdExtractor>().to(repos::show);
route.get("repos/:id").with_path_extractor::<OptIdExtractor>().to(repos::show);

// regex
route.get("users/:id:[a-z]+").with_path_extractor::<IdExtractor>().to(users::show);

// optional regex
route.get("search").with_path_extractor::<OptIdExtractor>().to(users::show);
route.get("search/:id:[a-z]+").with_path_extractor::<OptIdExtractor>().to(users::show);

// glob
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct PathExtractor {
    path: Vec<String>,
}

route.get("blob/*").with_path_extractor::<PathExtractor>().to(repo_blob::show);

// optional glob
#[derive(Deserialize, StateData, StaticResponseExtender)]
struct OptPathExtractor {
    path: Option<Vec<String>>,
}

route.get("tree").with_path_extractor::<OptPathExtractor>().to(repo_tree::show);
route.get("tree/*").with_path_extractor::<OptPathExtractor>().to(repo_tree::show);

// glob in middle
route.get("blob/*/commits").with_path_extractor::<PathExtractor>().to(repo_blob::commits);

// optional glob in middle
route.get("tree/commits").with_path_extractor::<OptPathExtractor>().to(repo_tree::commits);
route.get("tree/*/commits").with_path_extractor::<OptPathExtractor>().to(repo_tree::commits);
```

Cons

* There is no shorter way to specify optionals.
* The usage of path params is a little bit more verbose and is thus not easy to use.

## Scopes

As the developers start to build more features into their web app, they will be wanting to use scopes (also
known as *prefixes*). Most common use case should be supported, a *static scope* where the prefix is just a
string without any path params. In addition to that, scopes with params anywhere in the prefix should be
supported.

**NOTE**: I have reduced the number of code examples shown below because they can be inferred
from the param examples above and the scope examples shown here. But I have manually tested all
the possible scenarios related to this section.

#### Rails

```ruby
# static scope
scope 'pages' do
  get 'about', to: 'pages#about'
end

# optional regex scope
scope 'search(/:id)', id: /[a-z]+/ do
  get 'advanced', to: 'search#advanced'
end

# glob in middle scope
scope 'blob/*path/commits' do
  get 'graph', to: 'repo_blob#graph'
end
```

#### Actix

```rust
// static scope
app.service(scope("pages")
    .route("/about", get().to(pages::about))
);

// optional regex scope
app.service(scope("search")
    .route("/advanced", get().to(search::advanced))
    .service(scope("{id}")
        .route("/advanced", get().to(search::advanced))
    )
);

// glob in middle scope
app.service(scope("blob/{path:.+}/commits")
    .route("/graph", get().to(repo_blob::graph))
);
```

Cons

* There is no shorter way to specify scopes with optionals needing to duplicate the routes nested inside them.
* **MINOR**: Non-empty routes in sccopes with globs at the end do not work.

#### Tide

```rust
// static scope
app.at("pages").nest({
    let mut app = tide::new();
    app.at("about").get(pages::about);
    app
});

// glob in middle scope
app.at("blob/*path/commits").nest({
    let mut app = tide::new();
    app.at("graph").get(repo_blob::graph);
    app
});
```

Cons

* **MAJOR**: Does not support scopes with optionals.
* **MAJOR**: Does not support scopes with regex path param.

#### Warp

```rust
// static scope
let scope = path("pages");

scope.and(path!("about")).and(get()).and_then(pages::about);

// optional regex scope
let scope = path("search");

scope.and(path!("advanced")).and(get()).and_then(|| pages::about(Id::default()));
scope.and(path!(Id / "advanced")).and(get()).and_then(pages::about);
```

Cons

* **MINOR**: Does not support scopes with required glob path param.
* **MINOR**: Does not support scopes with anything after optional glob path param.
* There is no shorter way to specify scopes with optionals needing to duplicate the routes nested inside them.

#### Gotham

```rust
// static scope
route.scope("pages", |route| {
    route.get("about").to(pages::about);
});

// optional regex scope
route.scope("search", |route| {
    route.get("advanced").with_path_extractor::<OptIdExtractor>().to(search::advanced);
});

route.scope("search/:id:[0-9]+", |route| {
    route.get("advanced").with_path_extractor::<OptIdExtractor>().to(search::advanced);
});

// glob in middle scope
route.scope("blob/*/commits", |route| {
    route.get("graph").with_path_extractor::<PathExtractor>().to(repo_blob::graph);
});
```

Cons

* There is no shorter way to specify scopes with optionals needing to duplicate the routes nested inside them.
* The verbosity of defining params we saw in the above sections really starts to get in the way here. Scopes
  do not accept path extractors which means all the routes inside them need to define their own extractors.
  And if the routes themselves have params, you would need to define more custom extractors.

## Nested Scopes

For the next use case we will be talking about is nested scopes. Most web applications would
have complicated relationships between their data resources and end up using nested scopes to convey them.

**NOTE**: We have already seen most of the relevant code for this in the above section.

All the frameworks worked as expected except for the problems already described in the above sections.

## Sibling Scopes

Now, let us talk about the final but important use case, *Sibling Scopes*. When there are two scopes side by
side and they each have a route inside them that resolve to the same uri, the framework should resolve the
ambiguity by giving priority to the routes defined in either ascending or descending order.

**NOTE**: The code example given below might seem nonsensical but I chose it because it conveys the above
mentioned ambiguity in a simpler perspective.

#### Rails

```ruby
scope 'foo/higher' do
  get '', to: 'photos#foo_higher'
end

scope 'foo' do
  get 'higher', to: 'photos#foo_common_higher'
  get 'lower', to: 'photos#foo_common_lower'
end

scope 'foo/lower' do
  get '', to: 'photos#foo_lower'
end
```

When `/foo/higher` and `/foo/lower` is called in the below scenario, [Rails][] responds with the action
`foo_higher` and `foo_common_lower` respectively. As you can see, the routes defined at the top have higher
priority. [Django][] follows the same behaviour while [Laravel][] gives priority to the routes defined at
the bottom.

#### Rust

[Actix][], [Warp][] and [Gotham][] follow the rule by giving priority to the routes defined at the top

#### Tide

```rust
app.at("foo/higher").nest({
    let mut app = tide::new();
    app.at("").get(foo::higher);
    app
});

app.at("foo").nest({
    let mut app = tide::new();
    app.at("higher").get(foo::common_higher);
    app.at("lower").get(foo::common_lower);
    app
});

app.at("foo/lower").nest({
    let mut app = tide::new();
    app.at("").get(foo::lower);
    app
});
```

Cons

* **MINOR**: It seems to always go into the scope which matches the highest number of path segments. It
  responds with `foo::higher` for `/foo/higher` and `foo::lower` for `/foo/lower` here.

## Final Remarks

Based on the comparison results above, we can rank [Gotham][] as the most mature web micro framework in Rust
based on the routing capabilities. Next, we have a tie between [Actix][] and [Warp][] while [Tide][] occupies
the last spot by having the most deficiencies. I initially wanted to rank [Warp][] above [Actix][] but
decided not to. I have tested more complicated use cases but they all fall into one of the above compared
sections.

I did this comparison because I suspected some scoping issues in [Actix][]. I also got disillusioned with
[Gotham][] since it looks like nobody is maintaning it anymore and I wanted to rule it out of the contention
for the best web micro framework race. The results actually reaffirmed my opinion that [Gotham][] is the best
out of them all and I will keep using it and contributing to it.

I will try to keep this blog post up to date if the mentioned issues are fixed in the future. This blog is
open sourced at [github](https://github.com/pksunkara/pksunkara.github.io) which means I am willing to accept
pull requests to keep this up to date.

[Actix]: https://actix.rs
[Gotham]: https://gotham.rs
[Tide]: https://docs.rs/tide
[Warp]: https://docs.rs/warp
[Rails]: https://rubyonrails.org
[Django]: https://djangoproject.com
[Laravel]: https://laravel.com
