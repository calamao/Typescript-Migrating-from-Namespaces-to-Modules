# Introduction

## Lost in migration

You might already have figured it out what I said before all by yourself and started a try on converting your codebase to modules. You might even thought it could not be that hard and would be a matter of changing a few lines of code.

At some point you would reach to several dead ends not knowing exactly why. Then, as everybody else in your situation, you would have reached to Google to solve your problems. You might have even expected to find a whole walkthrough like this one on “how to migrate from namespaces to modules”, because, common, you cannot be the first one facing this issue… right? Well, let me tell you that you are not the first one, neither myself, but it pretty well seems I’m the first one deciding to detail this “walkthrough”.

After several days of browsing the web for a solution I was only able to find scattered clues, pieces of the puzzle that I had to assemble after really understanding what I was doing. Even I had to be creative myself to overpass some dead ends which had not easy “terms” even to google for.

After many days of frustration and hard work I decided to help my future-self, i.e. you, in this hard journey and create the walkthrough I was longing for.

But I’m getting off track… let’s start from the beginning.

## **Convert file to module**

Ok, so let’s convert one of our TS files to a module right? You have read a bit about it and cannot be that hard. You will see that you are completely right! Converting a file to a module is easy!  
****We have users.ts with a “_createUser_” function inside a namespace.

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }

  /*
  Some more private and public classes, functions, variables here...
  */
}  
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This function is used in another place of the application; _admin-workflow.ts_

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
namespace Application.Administration.Tasks {
  function signIn(info: SignInModel) {
    // Other important steps to signIn

    // create the user
    Application.Administration.Users.createUser(info.user);
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Let’s convert “_users.ts_” into a module**:**

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
export namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }

  /*
  Some more private and public classes, functions, variables here...
  */
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Good job! That was it! It wasn’t that hard right? Such a big deal for this?!!  
Yeah, noticed that “_export_” at the beginning? This is it! Your file is a module now.

> By definition a ES6 module is “kind of” any file starting with an “import/export”.

Ok, so let’s close this article as we have accomplish our goal here…   
Hey, wait a minute! _admin-workflow.ts_ does not compile any more!!! Even if we would use pure Javascript \(without TS compilation step\) that code crashes at runtime.  
What’s happening?!  
You might think there must be some small detail we are missing to fix the code again with such a tiny change...

You are wrong! It’s something it took me some time to realise with a lot of readings. That small change in code is a BIG change at runtime:

By converting your file into a module you have “closed” it to the external world. Now nobody can see your file unless explicitly importing it. Moreover, at runtime your file must be imported with a **module loader** \(like SystemJS, AMD, webpack; yes, webpack is much more than a module loader, but it also implements that functionality\). None of your code will be executed until you explicitly load it, so forget about putting a _debugger_ in your brand new module or any _console.log_ as it won’t be executed.

Moreover, previous namespace “_Application.Administration.Users_” is no longer affecting the global scope and won’t be merged with the rest of the “global namespaces” as it was happening before, so that code is not visible by the rest of the code in the global scope and won’t be merged with a file with the same namespace as it would have happened before. In fact that namespace is not useful anymore when we migrate to modules.  
So, even if we have not entered yet into the solution of this problem let me refactor the previous module as this would be the final stage of that file:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
export function createUser(user: User) {
  // Implementation
}

/*
Some more private and public classes, functions, variables here...
*/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

You see? We remove completely the namespace wrapper as we don’t need it anymore. It was useful to provide encapsulation in the global scope, but now our file is a module and is encapsulated “by definition”, only “exports” will be visible in the outside world.

## Obvious solution

You might be thinking… ok Jorge, now we have a module, and you said modules must be explicitly imported to be used right? So let’s import it where we use it.

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
import * as users from "./users-module";

namespace Application.Administration.Tasks {
  function signIn(info: SignInModel) {
    // Other important steps to signIn

    // create the user
    users.createUser(info.user);
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Mmm, that wasn’t that hard. You’ve seen code like this all over the web. Every nowadays-JS-file has it’s imports specified at the beginning of the file.  
Well, this refactor is completely correct, and with some runtime changes it might even work. But something has changed… Do you remember what I said about “_By definition a ES6 module is “kind of” any file starting with an “import/export”._”  
Well, this is exactly what you have done, converting your _admin-workflow.ts_ into a module! Congratulations again, you are one step closer to migrating your whole code base to modules!

But, wait a moment, now our _admin-workflow.ts_ file is not visible any more for the rest of my code base. So I must now import it in every file which depends on it… But this, as we’ve seen, means converting the dependees into modules… and so on and on.  
We end up converting the whole codebase to modules in a single row.  
If previous scenario is feasible for you, DO IT! It’s the easiest way to migrate to modules. This seems to be the only possible solution when you google it. Migrate your whole codebase or stay as it is.

Well, this is not enough when you have hundreds of TS files with thousands of lines of code. Not even necessary if your only necessity is to migrate some files that you would like to reuse in a modern web project with modules and leave the rest of the code base as it is.   
I wanted a **progressive migration.**

If you don’t want that you might well have finished with everything you needed from this article and you can already start your migration. But something tells me you would have figured that out all by yourself and you wouldn’t be here.

{% hint style="info" %}
Note: the progressive change is something that seems to be missing when you google on “how to migrate to modules”. It seems that the only possible solution is to close all the team in a room for 1 month until every piece of code have been migrated to modules. Good luck if you manage to sell that to your stakeholders!
{% endhint %}

## Simple goal

Ok, so let’s clarify the goal again oversimplifying it. We want to go from here:

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }

  /*
  Some more private and public classes, functions, variables here...
  */
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

To here:

{% code-tabs %}
{% code-tabs-item title="users-module.ts" %}
```typescript
export function createUser(user: User) {
  // Implementation
}

/*
Some more private and public classes, functions, variables here...
*/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

With some “small” detail: not breaking any other code in the process and without affecting the whole codebase \(which is still working in the global scope with namespaces\). This means:

* The runtime code works the same
* The typings \(yes, that makes it harder\) for the rest of our legacy code that still uses our code are the same. So the rest of our code base “believes” our migrated code is still a bunch of namespaces in the global scope.
* We want **to progressively** change our code base.

If you want some expanded version of “converting a file to a module” you can check the “Extra” section of this article.

