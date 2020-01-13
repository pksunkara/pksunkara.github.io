+++
date = 2016-10-01T15:25:32+02:00
draft = false
title = "Cloudant: CouchDB as Backend"

[taxonomies]
tags = ["database", "couchdb", "cloudant"]
categories = ["programming"]
+++

There have been a [lot](http://blog.mattwoodward.com/2012/03/definitive-guide-to-couchdb.html) [of](http://www.staticshin.com/programming/easy-user-accounts-management-with-couchdb/) [guides](https://nolanlawson.com/2013/11/15/couchdb-doesnt-want-to-be-your-database-it-wants-to-be-your-web-site/) on how to use [CouchDB](http://couchdb.apache.org) as a database for the web while restricting users to write only the data they own. I have been recently experimenting with how to use [Cloudant][]'s CouchDB as a backend directly for one of my Single Page Applications [**SPA**]. This blog post gives a little perspective on the limitations of the idea and how to achieve it.

## Limitations

The number one major limitation of CouchDB-as-a-backend is that every user can read all the data. This makes it not suitable for some kind of applications (finance etc..) but really well suited for other kinds of things. One famous example of this would be [NPM][], Node.js package manager.

The other major limitation is that there's no password recovery. You can't just say "*give your email, we’ll send you a new one*". Fortunately, you can bypass this by using [Single sign-on](https://en.wikipedia.org/wiki/Single_sign-on). I have personally decided to use [Auth0](https://auth0.com) in my **SPA**.

## Setting up

I have listed below the steps on how to achieve this [Cloudant][]. I am going to assume that you have an account ready to go with username **name** and password **pass**. Let's use the [NPM][] service as an example.

### Create Users Database

You need to create an **_users** database which will be used for user authentication and authorization.

```bash
curl -X PUT -u name:pass https://name.cloudant.com/_users
```

###### Turn off Cloudant Security

You need to turn off [Cloudant][]'s security for this database so that you can allow anyone to register themselves as user.

Create the security document which needs to be uploaded as **security.json**.

```json
{
  "couchdb_auth_only": true,
  "members": {
    "names": [],
    "roles": []
  },
  "admins": {
    "names": [],
    "roles": []
  }
}
```

Upload it.

```bash
curl -X PUT -u name:pass https://name.cloudant.com/_users/_security \
-H "Content-Type: application/json" \
-d @security.json
```

### Create Application Database

You need to create a database which will be used as backend of your application. Let's create one called **npm**.

```bash
curl -X PUT -u name:pass https://name.cloudant.com/npm
```

###### Make it public

You need to turn off [Cloudant][]'s security for this database so that you can allow all users to read and write.

```bash
curl -X PUT -u name:pass https://name.cloudant.com/npm/_security \
-H "Content-Type: application/json" \
-d @security.json
```

### Secure Application for User

You need to add a validation function which runs whenever someone's trying to write to the application database.

Create the validation function as **validation.json**.

```json
{
  "_id": "_design/_auth",
  "language": "javascript",
  "validate_doc_update": "function(newDoc, oldDoc, userCtx) {\n\n  function require(beTrue, key, message) {\n    var err = {};\n    err[key] = message;\n\n    if (!beTrue) throw(err);\n  }\n\n  require(userCtx.name, 'unauthorized', 'You need to login');\n\n  if (oldDoc) {\n    require((userCtx.roles.indexOf('_admin') != -1 || userCtx.name == oldDoc.user), 'forbidden', 'You are not allowed to update it');\n  }\n}\n"
}
```

Upload it as a design document.

```bash
curl -X POST -u name:pass https://name.cloudant.com/npm
-H "Content-Type: application/json" \
-d @validation.json
```

This validation function is explained [below](#app-explanation)

### Secure User Data

You need to secure the authentication database data for users.

Create the validation function as **authentication.json**.

```json
{
  "_id": "_design/_auth",
  "language": "javascript",
  "validate_doc_update": "function(newDoc, oldDoc, userCtx) {\n  if (newDoc._deleted === true) {\n    if (userCtx.roles.indexOf('_admin') == -1 && userCtx.name != oldDoc.name) {\n      throw({forbidden: 'Only admins may delete other user docs.'});\n    }\n\n    return;\n  }\n\n  if ((oldDoc && oldDoc.type !== 'user') || newDoc.type !== 'user') {\n    throw({forbidden : 'doc.type must be user'});\n  }\n\n  if (!newDoc.name) {\n    throw({forbidden: 'doc.name is required'});\n  }\n\n  if (!newDoc.roles) {\n    throw({forbidden: 'doc.roles must exist'});\n  }\n\n  if (!isArray(newDoc.roles)) {\n    throw({forbidden: 'doc.roles must be an array'});\n  }\n\n  for (var idx = 0; idx < newDoc.roles.length; idx++) {\n    if (typeof newDoc.roles[idx] !== 'string') {\n      throw({forbidden: 'doc.roles can only contain strings'});\n    }\n  }\n\n  if (newDoc._id !== ('org.couchdb.user:' + newDoc.name)) {\n    throw({forbidden: 'Doc ID must be of the form org.couchdb.user:name'});\n  }\n\n  if (oldDoc) {\n    if (oldDoc.name !== newDoc.name) {\n      throw({forbidden: 'Usernames can not be changed.'});\n    }\n  }\n\n  if (newDoc.password_sha && !newDoc.salt) {\n    throw({forbidden: 'Users with password_sha must have a salt. See /_utils/script/couch.js for example code.'});\n  }\n\n  if (newDoc.password_scheme === \"pbkdf2\") {\n    if (typeof(newDoc.iterations) !== \"number\") {\n       throw({forbidden: \"iterations must be a number.\"});\n    }\n\n    if (typeof(newDoc.derived_key) !== \"string\") {\n       throw({forbidden: \"derived_key must be a string.\"});\n    }\n  }\n\n  if (userCtx.roles.indexOf('_admin') == -1) {\n    if (oldDoc) {\n      if (userCtx.name !== newDoc.name) {\n        throw({forbidden: 'You may only update your own user document.'});\n      }\n\n      var oldRoles = oldDoc.roles.sort();\n      var newRoles = newDoc.roles.sort();\n\n      if (oldRoles.length !== newRoles.length) {\n        throw({forbidden: 'Only _admin may edit roles'});\n      }\n\n      for (var i = 0; i < oldRoles.length; i++) {\n        if (oldRoles[i] !== newRoles[i]) {\n          throw({forbidden: 'Only _admin may edit roles'});\n        }\n      }\n    } else if (newDoc.roles.length > 0) {\n      throw({forbidden: 'Only _admin may set roles'});\n    }\n  }\n\n  for (var i = 0; i < newDoc.roles.length; i++) {\n    if (newDoc.roles[i][0] === '_') {\n      throw({forbidden: 'No system roles (starting with underscore) in users db.'});\n    }\n  }\n\n  if (newDoc.name[0] === '_') {\n    throw({forbidden: 'Username may not start with underscore.'});\n  }\n\n  var badUserNameChars = [':'];\n\n  for (var i = 0; i < badUserNameChars.length; i++) {\n    if (newDoc.name.indexOf(badUserNameChars[i]) >= 0) {\n      throw({forbidden: 'Character `' + badUserNameChars[i] + '` is not allowed in usernames.'});\n    }\n  }\n}"
}
```

Upload it as a design document.

```bash
curl -X POST -u name:pass https://name.cloudant.com/_users
-H "Content-Type: application/json" \
-d @authentication.json
```

This validation function is explained [below](#user-explanation).

## Explanations

This section explains how the validation functions work.

<a name="app-explanation"></a>
### Application Database

```js
function(newDoc, oldDoc, userCtx) {

  function require(beTrue, key, message) {
    var err = {};
    err[key] = message;

    if (!beTrue) throw(err);
  }

  // You need the user to be logged in
  require(userCtx.name, 'unauthorized', 'You need to login');

  // If the user is updating a document, you need to make sure he owns it
  if (oldDoc) {
    require((userCtx.roles.indexOf('_admin') != -1 || userCtx.name == oldDoc.user), 'forbidden', 'You are not allowed to update it');
  }
}
```

<a name="user-explanation"></a>
### Users Database

```js
function(newDoc, oldDoc, userCtx) {
  if (newDoc._deleted === true) {
    // Allow deletes by admins and matching users without checking the other fields
    if (userCtx.roles.indexOf('_admin') == -1 && userCtx.name != oldDoc.name) {
      throw({forbidden: 'Only admins may delete other user docs.'});
    }

    return;
  }

  // We only allow user docs for now
  if ((oldDoc && oldDoc.type !== 'user') || newDoc.type !== 'user') {
    throw({forbidden : 'doc.type must be user'});
  }

  if (!newDoc.name) {
    throw({forbidden: 'doc.name is required'});
  }

  if (!newDoc.roles) {
    throw({forbidden: 'doc.roles must exist'});
  }

  if (!isArray(newDoc.roles)) {
    throw({forbidden: 'doc.roles must be an array'});
  }

  for (var idx = 0; idx < newDoc.roles.length; idx++) {
    if (typeof newDoc.roles[idx] !== 'string') {
      throw({forbidden: 'doc.roles can only contain strings'});
    }
  }

  if (newDoc._id !== ('org.couchdb.user:' + newDoc.name)) {
    throw({forbidden: 'Doc ID must be of the form org.couchdb.user:name'});
  }

  // Don't allow usernames to be changed
  if (oldDoc) {
    if (oldDoc.name !== newDoc.name) {
      throw({forbidden: 'Usernames can not be changed.'});
    }
  }

  // Password logic
  if (newDoc.password_sha && !newDoc.salt) {
    throw({forbidden: 'Users with password_sha must have a salt. See /_utils/script/couch.js for example code.'});
  }

  if (newDoc.password_scheme === "pbkdf2") {
    if (typeof(newDoc.iterations) !== "number") {
       throw({forbidden: "iterations must be a number."});
    }

    if (typeof(newDoc.derived_key) !== "string") {
       throw({forbidden: "derived_key must be a string."});
    }
  }

  // If user is not server admin
  if (userCtx.roles.indexOf('_admin') == -1) {
    if (oldDoc) {
      // Allow user to update only their documents
      if (userCtx.name !== newDoc.name) {
        throw({forbidden: 'You may only update your own user document.'});
      }

      // Validate role updates
      var oldRoles = oldDoc.roles.sort();
      var newRoles = newDoc.roles.sort();

      if (oldRoles.length !== newRoles.length) {
        throw({forbidden: 'Only _admin may edit roles'});
      }

      for (var i = 0; i < oldRoles.length; i++) {
        if (oldRoles[i] !== newRoles[i]) {
          throw({forbidden: 'Only _admin may edit roles'});
        }
      }
    } else if (newDoc.roles.length > 0) {
      throw({forbidden: 'Only _admin may set roles'});
    }
  }

  // Character validations
  for (var i = 0; i < newDoc.roles.length; i++) {
    if (newDoc.roles[i][0] === '_') {
      throw({forbidden: 'No system roles (starting with underscore) in users db.'});
    }
  }

  if (newDoc.name[0] === '_') {
    throw({forbidden: 'Username may not start with underscore.'});
  }

  var badUserNameChars = [':'];

  for (var i = 0; i < badUserNameChars.length; i++) {
    if (newDoc.name.indexOf(badUserNameChars[i]) >= 0) {
      throw({forbidden: 'Character `' + badUserNameChars[i] + '` is not allowed in usernames.'});
    }
  }
}
```

## Addendum

You need to make sure that every data document you create in the application database needs to have an **user** field which need to be filled with the username of the user. Otherwise the validation function will keep throwing up forbidden errors.

You are now ready to use CouchDB-as-a-backend on [Cloudant][]. Enjoy!

[Cloudant]: https://cloudant.com
[NPM]: https://npmjs.com
