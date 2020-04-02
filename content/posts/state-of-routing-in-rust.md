+++
date = 2020-03-30T00:00:00+01:00
draft = true
title = "State of routing in Rust"

[taxonomies]
tags = ["rust", "web"]
categories = ["programming"]
+++

There are many micro frameworks in Rust. Some famous examples are [Actix][actix], [Gotham][gotham], [Tide][], [Warp][warp], etc. We have seen many blog posts comparing their performances and middleware capabilities. But what we haven't seen is an article comparing their routing functionality. I hoped to fill that gap and make this post the most up to date guide for the state of routing in Rust.

**NOTE**: I will not be comparing [Rocket](http://rocket.rs) because it does not run on *Stable* yet.

We have a lot of mature web frameworks like [Rails][rails], [Django][django], [Laravel][laravel], [Phoenix][phoenix], etc.. that have been working on solving the routing problem and dealing with the typical use cases from web developers. Before we start comparing, let's define the benchmark using these frameworks.



[actix]: http://actix.rs
[gotham]: http://gotham.rs
[tide]: http://docs.rs/tide
[warp]: http://docs.rs/warp
[rails]: http://rubyonrails.org
[django]: http://djangoproject.com
[laravel]: http://laravel.com
[phoenix]: http://www.phoenixframework.org/



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