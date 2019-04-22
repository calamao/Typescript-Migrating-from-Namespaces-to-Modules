---
description: >-
  I have added this section to give you some extra explanations here that would
  otherwise drift me away from my main workflow.
---

# Extras

## **Generate jQuery module from global scope**

It’s pretty common to have this dependency in legacy projects, even if you work in a modern one you might still need it.

The thing is that, it will be pretty common to refactor a module, trying to convert all it’s dependencies to a module in order to have a “pure module”, and realize that one of those dependencies is jQuery. Of course that dependency is in the global scope like the rest and is loaded by a “script tag”.

Now you need jQuery to be a module like the rest of your dependencies, but also you need it still globally for the rest of your code.  
****So how can we convert the same jQuery that it's loaded in the global scope into a module to be loaded by the modules that require it?

Also jQuery is normally loaded like a named module. so let's come back to our refactored module and put jQuery as a dependency:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
import * as jQuery from 'jquery';

export function createUser(user: User) {
  // Implementation

  jQuery.ajax(/* save user to API */)
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

But if we don’t do anything else ‘jquery’ is not going to be found by the module loader.

So now comes the tricky part again. Let’s come back to our _immediate-module-loader.ts_ and add some more code to it.

{% code-tabs %}
{% code-tabs-item title="immediate-module-loader.ts" %}
```typescript
var ModulesLoaded = loadModule("global-module-loader");


// create a "fake" module of Jquery that loads it from the window global object. This is needed for the module dependencies on jQuery:
// import * as jQuery from "jquery";
System.register("jquery", [], function ($__export, $__moduleContext) {

    return {
        // setters: [], // (setters can be optional when empty)
        execute: function () {

            // this is done so it can be imported as:
            // import * as jQuery from "jquery";
            for (const prop in jQuery) {
                $__export(prop, jQuery[prop]);
            }
        }
    };
});
```
{% endcode-tabs-item %}
{% endcode-tabs %}

I’m not going to explain into detail that code but basically: it’s a very dependent code on the module loader we are using \(SystemJS in this case\). We are creating a module named “jquery” and we are exporting every single member of the jQuery global object into a “exported” property of the module.  
Now our `import * as jQuery from 'jquery';` will import the “generated” module as expected.

## Dependency injection

This is probably a quite specific use case that it was necessary for me. I think anyway it is a very interesting topic to comment as having a module loader now leverages the usage of this fancy technique known as dependency injection.

One of my dependencies was a “notification” component that would show notifications in my legacy application. This component used behind the scenes a Bootstrap plugin to show fancy notifications.  
You can imagine this was a very common component used across all the codebase, including the files we were converting to modules.

As I said I was converting many of those files to be used in a brand-new “create-xxx-app” application. What of course I would not do is bring the legacy notification component into the new codebase. Meaning I would be adding a dependency to Bootstrap and then to the “notification” plugin. But I wanted to use other kind of notification in the new architecture, more aligned with the new framework used.  
So my refactored component would have to show notifications with the Bootstrap plugin when running into the legacy app but use another kind of notification system when running in the modern app.  
How we do this?  
Dependency injection to the rescue!

So let’s considered we have refactored our notification component into a module as we’ve learned.  
Something like this:

{% code-tabs %}
{% code-tabs-item title="notification.ts" %}
```typescript
///<amd-module name="notification-implementation"/>

export function showNotification(message: string, type: string) {
  // Implementation: using a Bootstrap plugin for example
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This will not be a “pure module” as it will use some plugins from the global scope, but we don’t mind for the moment as this is just the implementation for the legacy app which I’m not interested to use in the modern app.  
Notice also that we have named the module to "**notification-implementation**", we will be using it.

Coming back to our “already famous” refactored component, we will use the notifications like this:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
import showNotification from "notification"

export function createUser(user: User) {
  // Implementation

  showNotification("User created!" , "info");
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This named “notification” module does not really exist yet so Typescript cannot find it and therefore cannot provide us with Intellisense. This is the first thing we are going to do. Tell TS what this module looks like:

{% code-tabs %}
{% code-tabs-item title="notification.d.ts" %}
```typescript
declare module "notification" {
  export function showNotificationStandard(message: string, type: string): void;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

For the record: I would never type a “type” property as a string, but with an enum, but just for the sake of simplicity.

You see we’ve created a declaration file \(we’ve seen already that before\) and provided the typing for our module. Now Typescript knows what it looks like when we do something like before:  
`import showNotification from "notification"`  
We have the types in action.

But, at runtime this module does not exist anywhere yet and our module loader is going to complain about that. Let’s fix it.Come back to our _immediate-module-loader.ts_ and register another module:



{% code-tabs %}
{% code-tabs-item title="immediate-module-loader.ts" %}
```typescript
var ModulesLoaded = loadModule("global-module-loader");

// load module from the named-export 'notification' so we can change it at runtime in other App (IoC)
// import * as Notification from "notification";
System.register("notification", ["notification-implementation"], function(
  $__export,
  $__moduleContext
) {
  var Notification;
  return {
    setters: [
      function(notificationImplementation) {
        Notification = notificationImplementation;
      }
    ],
    execute: function() {
      $__export(Notification);
    }
  };
});
```
{% endcode-tabs-item %}
{% endcode-tabs %}

With this we are registering a new module called “notification”. This module has a “module dependency” named “**notification-implementation**”. Remember our refactored module for the legacy notifications? We named it that way.

Now, the rest is very specific for SystemJS module loader, but basically we have the “setters” functions which will be called once every dependency has been loaded. This is where we will take the “notification-implementation” module, provided via setter, and it will be later the one which is going to be exported by the module.

So at the end our “notification” module is going to be exactly the same as our “notification-implementation” module.  
Dependency injection implemented.

## **Dependency injection with Webpack**

If you are interested in how do I manage to inject another dependency for the “notification” module in the new architecture using webpack you might read this.  
****Basically edit your webpack config file and add the following:

```typescript
module.exports = {
 resolve: {
    // other properties...

     // Implement IoC for "notification" component
     // with this we can do:
     // import * as Notification from 'notification';
     // thanks to this alias we can redirect to the New implementation of the "notification" component
     notification: resolve(__dirname, 'path/to/new/component/Notification'),
   }
 },
}
```

You see? The same we did via SystemJS in the legacy app with Webpack is just a matter of “configuration”.

## **Refactor module dependencies**

This chapter is only useful for you if you want to reuse your refactored module in a “pure brand-new” modules codebase \(as it was my case\). Otherwise feel free to skip this chapter.

In the “convert files to module chapter” I said our refactored module is perfect. Well that’s true for that oversimplified example but wouldn’t be that true for a more complex one.  
****Let’s make first some clear statement that you have probably figure it out already:

{% hint style="info" %}
You can access all the code in the global scope from the module files but not the other way round.
{% endhint %}

This means that once we convert a file to a module we don’t need to care about it’s dependencies which are still in the global scope as they are still accessible from inside the module. What we are forced to fix are all the dependees which use our refactored module.

Still, if we are thinking of using our refactored module in a brand new modules code base we don’t want any kind of dependency of our module in the global scope. We want all the dependencies **explicitly imported** by the module. This is what I call a “pure module”.

If we want to accomplish this, it will finally mean that we have to also convert all our dependencies to modules. That way our module will be a “pure module” with no dependency in the global scope, ready to be used in a “pure brand new module codebase”.

So, let’s put an example. Let’s say our initial users.ts file had this look:

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation...

    // notify the user
    Application.Email.send(user.email, "User created successfully");
  }

  /*
  Some more private and public classes, functions, variables here...
  */
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Let’s see the refactored version of it:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
export function createUser(user: User) {
  // Implementation...

  // notify the user
  Application.Email.send(user.email, "User created successfully");
}

/*
Some more private and public classes, functions, variables here...
*/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

As we said, this is a perfectly fine module which will work as long as we have the “_Application.Email.send_” global dependency in our codebase. And it is exactly that way. Anyway, if you would like to reuse exactly that file in a brand new modules codebase you would be forced to also create that dependency in the global scope. This would mean our “brand new codebase” would start to be looking a bit legacy again… This is bad.  
If this is your case \(reusability in a new modules code base\), as it was mine, then you need to also refactor all it’s dependencies to modules. And you cannot scape that as you don’t want to use any special tricks in your new code base.

So let’s say that our email functionality was:

{% code-tabs %}
{% code-tabs-item title="notify-email.ts" %}
```typescript
namespace Application.Email {
  export function send(email: string, message: string) {
    // Implementation...
    // maybe even some further dependencies that need to be refactored as well
  }

  /*
  Some more private and public classes, functions, variables here...
  */
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

It would be refactored similarly as before to:

{% code-tabs %}
{% code-tabs-item title="notify-email-module.ts" %}
```typescript
export function send(email: string, message: string) {
  // Implementation...
  // maybe even some further dependencies that need to be refactored as well
}

/*
Some more private and public classes, functions, variables here...
*/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

And now we can explicitly import it in our initial module:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
import { send as sendEmail } from "/path/to/file/notify-email";

export function createUser(user: User) {
  // Implementation...

  // notify the user
  sendEmail(user.email, "User created successfully");
}

/*
Some more private and public classes, functions, variables here...
*/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Now we have converted our global dependency into a module as well and we can explicitly import it as with any other module.

If you do this for all the dependencies and of course the dependencies of those dependencies you will be able to reuse that code as it is in a pure module codebase.  
****Otherwise \(and as I say I don’t like that\) you will need to use a technique \(which I’m not explaining\) to take all your global dependencies and also create them as global into your new module codebase.

## **Reusability in a brand new code base**

I’ve been talking several times about reusability of legacy code into a brand new create-xxx-app codebase. Let me just clarify this scenario, as it was my case, for you to understand further all the references to it.  
****I had a big part of my legacy app that I wanted to reuse in a independent brand new modern app. To give some extra details it was all our typescript client code responsible for communicating with the the API.  
Also my legacy app is not a dead app at all. That app was the main core of our project and is meant to be there for much longer time with constant changes and enhancements into the files migrated that had to be visible both in the legacy and the modern app.  
Both projects where in the same GIT repository, that is how “sharing” was accomplished in this case. \(a more sophisticated approach would have been to publish the target modules into a private npm repository to be reused across many projects\).

So, once a file and it’s dependencies where successfully migrated to modules I could simply reuse it in my new codebase going some folders “up” when “importing” from my new app to my legacy “converted” module.

Imagine this example scenario:

![](.gitbook/assets/unnamed.png)

We want to reuse the _users-module.ts_ into our _user-administration.ts_ component.Then we would just add this _import_ at the beginning of users-administration.ts

{% code-tabs %}
{% code-tabs-item title="users-administration.ts" %}
```typescript
import * as users from "../../OldWebApp/Administration/users-module";

// example usage
users.getUser("userId");
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This is just a way of how to reuse an already migrated module into the new architecture. As I said there could be more sophisticated ones but this is not the focus of this article.

## **Aliasing namespaces**

This part is only useful for you if you had created some alias for your refactored namespaces.  
Let’s say you had something like this is your code before refactoring it.

```typescript
import Users = Application.Administration.Users;
```

This is pretty handy when you are using a lot this namespace `Application.Administration.Users` across your codebase. With that alias where you would write something like this:  
`Application.Administration.Users.createUser(info.user);`  
Now you will be able to write something like this:  
`Users.createUser(info.user);`  
  
If you were using an alias like that one, as I did, then you will need to add some extra trick.  
The obvious modification to our “shim” file to add the typing would be:

```typescript
declare global {
  namespace Application.Administration {
    export import Users = users;
  }
  export import Users = users;
}
```

This should be a valid piece of code, unfortunately TS complains about that saying something like “_Imports are not permitted in module augmentations_”.  
From my investigation this seems to be considered a bug in the TS compiler, but they are not going to fix that for the moment.

So what can we do instead?Create a new declaration file “**.d.ts**” and add next content:

{% code-tabs %}
{% code-tabs-item title="users-module-shim-global.d.ts" %}
```typescript
import Users = Application.Administration.Users;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This is exactly the same line you had at the beginning. The only difference is that this line is now in a “global declaration file”. If you are not familiar with declaration files you can check TS documentation.  
The important thing here is that we have our alias back again ready to be used. Also, as it is in a declaration file, TS is not going to generate any code for it, but everything compiles again.  
So now we need to add the implementation for that.  
Let’s come back to our “shim” file and add the implementation of the alias:

{% code-tabs %}
{% code-tabs-item title="users-module-shim.ts" %}
```typescript
// ....

Application.Administration.Users = {
  ...Application.Administration.Users,
  ...users
};
window["Users"] = window["Users"] || Application.Administration.Users;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

You see the last part? This is it. We have added a global variable “Users” and now code like this:  
`Users.createUser(info.user);`  
will work again.

## **Namespace generator**

As you are going to create a lot of “shim” files in the migration of your code you will have to generate a lot of namespaces manually. This is how it look likes that code:

```typescript
window.Application = window.Application || {};
window.Application.Administration = window.Application.Administration || {};
window.Application.Administration.Users = window.Application.Administration.Users || {};
```

This if for me a lot of boilerplate so I created a helper function that can be used like this:

```typescript
GenerateNamespace("Application.Administration.Users");
```

And this is the implementation of that function.

```typescript
function GenerateNamespace(path: string) {
  path = "window." + path;

  const namespaceProps = path.split(".");

  const prependPreviousPath = namespaceProps.map((curr, index) => {
    return namespaceProps.slice(0, index + 1).join(".");
  });

  const createObject = objectName => {
    return `${objectName} = ${objectName} || {};`;
  };

  const pathGenerator = prependPreviousPath.reduce(
    (prev, currentNamespaceObject) => {
      return prev + createObject(currentNamespaceObject);
    },
    ""
  );

  // script string
  const script = pathGenerator;
  eval(script);
}
```

I’m not going to explain that code but I hope you can understand it. Otherwise debug it and you will see it pretty clear what is doing.  


  
  
  


