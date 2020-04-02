+++
date = 2019-12-13T00:00:00+05:30
draft = false
title = "Reign: Configuration Management"

[taxonomies]
tags = ["rust", "web", "reign"]
categories = ["programming"]
series = ["Reign"]

+++

Every web application requires configuration management. Providing a configuration manager is one of important tasks of any web framework.

[Rails](https://rubyonrails.org) does it by asking the user to edit yaml files and a few ruby files. Some node.js frameworks achieve this by using JSON files. In this post, I will present an idea on how the configuration management system should ideally work.

Few years ago, [Heroku](https://heroku.com) published a website detailing a concept called [The Twelve-Factor App](http://12factor.net). We can see that configuration is listed third in the list. I would like to quote some of their sentences.

> An application’s config is everything that is likely to vary between deploys (staging, production, developer environments, etc)

Being run in different environments is something that a web application cannot avoid. A normal web application would need at least 3 different environments. One for **production**, one for **development** and the last for **test**. A good configuration system should support multiple environments.

> Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires strict separation of config from code.

This is the driving force behind using a configuration management system. Some might say that this would complicate the overall code by adding yet another module. It would also take more time to develop this system. But, with the rise of module/package managers, it is just a matter of plugging in the system. In order to reduce further complexity, Retrieving a configuration value from the system should be as simple as writing a variable name.

> The twelve-factor app stores config in environment variables

Most of the current configuration managers use files to store the config. This method still doesn’t solve the problem of needing to add the config files to the code. As mentioned above, the ideal method of storing config is using environment variables (also known as **env vars** or **env**). On average, a web application needs at least 10 config variables. It will be difficult and annoying for the developer to use and maintain such high number of env vars which is why most of them opt to add the config files to their code.

Therefore, an ideal configuration system should support both env vars and config files. It would be even better if the system could merge the env vars with the config files. Maybe it can support command line arguments too.

> In a twelve-factor app, env vars are granular controls, each fully orthogonal to other env vars. They are never grouped together as “environments”, but instead are independently managed for each deploy.

Most people don’t realize that there is a hidden problem in this. For example, if a developer wants to use database information config from **staging** environment for their personal **joes-staging** environment, they need to keep monitoring the **staging** environment for any changes and have to update their personal environment config file. One way to mitigate this issue is by providing a way for environment config files to merge with each other while allowing to batch config in named groups.

Then, in the above example, all Joe has to do is create a **joes.staging** environment config file with the required config and leave the database config empty. When the configuration manager needs to retrieve database config for his environment, it should fallback to **staging** environment config.

Let us recap. A good configuration manager should:

 - Support multiple environments in files
 - Be easy to retrieve a config value
 - Merge env vars, command line args with config in files
 - Merge different environment files when required
