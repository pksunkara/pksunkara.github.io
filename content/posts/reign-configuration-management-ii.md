+++
date = 2020-03-27T00:00:00+01:00
draft = false
title = "Reign: Configuration Management - II"

[taxonomies]
tags = ["rust", "web", "reign"]
categories = ["programming"]
series = ["Reign"]

+++

We have looked at the basic requirements for a configuration management system in our [previous post](../reign-configuration-management). If you recall, we described one of the requirement as:

> Retrieving a configuration value from the system should be as simple as writing a variable name

Reading an environment variable is especially tricky because they do not have types. A variable whose value is `"true"` can be describing either a `String` or a `bool`. Similarly, a variable with value `"1"` can be describing any of `String`, `int32` or `bool`. If we implement a naive retrieval system, then the user would need to validate the value before using it.

We also describe another requirement as:

> Merge env vars with config in files

One of the issues with this approach is leaking of non application related environment variables into the application code. While doing this in a backend application might not be a big security issue, it would still be better if the configuration management system can allow the users to whitelist all the config values they need.

We haven't really talked about using secrets in configuration variables during development of the application. You can not write them into the config files because they are secret and need to be kept that way. A developer can always create local environment variables before starting development but this becomes cumbersome and repetitive. One way of fixing this would be to store those environment variables in the developer's local shell profile. But, what if two different applications use the same environment vairable name?

We can solve this by using **local config files**. These files are similar to the environment config files but are not stored in the code and exist only locally. Our configuration management system needs to account for these files while processing the config files.

Let us think of a medium sized team of five developers working on an application. They would need to share the secrets with each other for local development. They could pass around these **local config files**, but this is not scalable. They could give people access to the secrets so that the developers can retrieve the secrets directly, but this is also not scalable and more importantly not secure from an organisation perspective.

We can fix this by having an external secret manager (ex: [Vault](https://www.vaultproject.io/)) and a small script which can retrieve the secrets after retrieving code from upstream. Applications with secrets for local development is a rare use case and our system can either support this or not. It can also be argued that this is out of scope for the configuration management system.

Thus, a good configuration manager should also:

* Validate the config value during retrieval
* Allow whitelisting of needed environment variables
* Support local config files

