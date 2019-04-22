# Solution

Now is time to stop talking about the issues and focus on the solutions. Let’s come back to our successfully migrated module and the code that consumes it:

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

Our users-module.ts file is perfect now. We can reuse it if we want in a brand new create-xxx-app modules codebase.

## Fix the typings

As we’ve seen admin-workflow.ts is broken now as there is no global namespace with createUser inside like “Application.Administration.Users.createUser”. Typescript is awesome and knows that. So…

> **How do we tell Typescript that we want to expose our refactored module to the global scope?**

I think this is one of the hardest tricks that took me days of readings to get to.  
****Remember how was the code before the refactor?

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This is how the external world “sees” our code and we must still provide the same “signature”.

Let’s create a file named “users-module-shim.ts” \(I sufix it with “shim” to mark it as something special/tricky\). In fact, once we’ve managed to convert all the dependees to modules all the “shim” files could be removed as they would not be necessary anymore.  
Their only purpose is to avoid breaking the code of the dependees which are still in the global scope, in our case “admin-workflow.ts”.

I will be adding some code progressively to this file in order to be understandable:

### Import the module

{% code-tabs %}
{% code-tabs-item title="users-module-shim.ts" %}
```typescript
import * as users from "path/to/file/users-module";
```
{% endcode-tabs-item %}
{% endcode-tabs %}

First of all we import all \(“\* as users”\) the content of our refactored module.  
What does this import mean at the beginning of the file? Well, as we’ve seen, this mean our “shim” file will be also a module!  
Why do we export all? We want to import everything that is exported in our refactored module.

### Declare global

Ok, let’s go on:

{% code-tabs %}
{% code-tabs-item title="users-module-shim.ts" %}
```typescript
import * as users from "path/to/file/users-module";

declare global {
  namespace Application.Administration {
    export import Users = users;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Whaaaat?!  
Yes, don’t underestimate the brevity of those lines. They took me a lot of suffering, try and error, and dead ends before getting to them.  
FIrst of all:  
**declare global**: this is a really powerful command! We are declaring something in the global scope from inside a module! This is the only way to do it! This is awesome as we want to create some new global type from inside a module, which is closed to the external world.  
Remember the initial question of this chapter: _How do we tell Typescript that we want to expose our refactored module to the global scope?_  
Well, this is the key to do it!

There is something else here; **declare**; this means to Typescript that we are providing information just for the compiler to check the types properly, but typescript won’t generate any code at runtime. This is also ok, because we are going to generate the runtime code ourselves later.

### Use an alias

Ok, so we are declaring something in the global scope from inside a module. What is it?  
First of all we have the namespace:

```typescript
namespace Application.Administration
```

You see? Not exactly the same as the original. We are declaring a namespace but removing the last part of it. Why? Because we want to create a public “variable” \(it’s not really a variable; check later\) inside this namespace that will have all the information of the refactored module.

Note: it would have been awesome to be able to do something like this:

```typescript
namespace Application.Administration.Users = users;
```

Unfortunately that’s not possible, although quite intuitive.

So let’s see that “variable” inside the namespace:

```typescript
export import Users = users;
```

Mmmm, that does not look like a variable if you have been using typescript.

The `import Users =`  part is called an “**alias**” in Typescript and it’s another piece of interesting code. Again took me a loooot of time to bump into it. It’s a variable at runtime but it also has typing information inside. And it can be used as a “wrapper”, the same way namespaces and modules are used.

Official doc about the alias:  
[**https://www.typescriptlang.org/docs/handbook/namespaces.html**](https://www.typescriptlang.org/docs/handbook/namespaces.html)

### **Dead ends before the alias**

Maybe it’s better to understand that part seeing a bit of my many tries an errors.`export var Users = users;`  
****This is a valid piece of code and it would work in some cases. In fact it would work for the `createUser` function, and the compiler would not complain. But it would NOT work if we are exporting types inside the original namespace.  
Let’s say we had something like this originally in our users.ts file:

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }

  export interface User {}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

As you see we are also exporting an interface from our original namespace, and we are also using that interface:

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
namespace Application.Administration.Tasks {
  function signIn(info: SignInModel) {
    // Other important steps to signIn

    // create the user
    Application.Administration.Users.createUser(info.user);
  }

  function processUser(user: Application.Administration.Users.User) {
    // do something
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

You see? We have now a function “_processUser_” which uses the exported interface `user: Application.Administration.Users.User`  
Now the compiler would complain as for him this:  
`export var Users = users;`  
Is declaring a variable and cannot be used as a type. I recommend you a further reading about the “value space” and the “type space” for Typescript and realize how it treats them both quite differently having a clear separation of what affects the “value space”, what the “type space” and what both \(example: classes\).

Ok so you might think… Let’s also create a type so we can use the types inside:  
`export type Users = users;`  
Unfortunately this type becomes the “module type”, but cannot be used as a wrapper/container of all the things the module has inside, so it could not be used as we are expecting to:  
`user: Users.User`

### **Alternative to alias: explicit definitions**

Another alternative to the alias is specifying every single exported member. Let’s take the previous example again; imagine we had:

{% code-tabs %}
{% code-tabs-item title="users.ts" %}
```typescript
namespace Application.Administration.Users {
  export function createUser(user: User) {
    // Implementation
  }

  export interface User {}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Then the proper creation of the typing without using an alias is:

{% code-tabs %}
{% code-tabs-item title="users-module-shim.ts" %}
```typescript
import * as users from "path/to/file/users-module";

declare global {
  namespace Application.Administration.Users {
    export var createUser = users.createUser;
    export type User = users.User;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

As you see we must create a type manually for every single export. Notice also that we are keeping the original namespace now `Application.Administration.Users` so the type we are declaring now is exactly the same as we had originally.  
****The alias works but is kind of a trick which allows us to use the code as if it were a namespace.

### **Alias vs Explicit definitions**

I’ve shown you two different ways to fake the original namespace typing. Both solutions are similar but have subtle differences:

#### **Alias**

The alias solution is not really the same as the original namespace `Application.Administration.Users` even that it works from the point of view of the usage from the dependees. But there is a case when the alias technique would not work as it is. That case is when you use namespace “merging” which is very common by the way. Let’s say you had 2 files where the previous namespace was declared: _users.ts_ and _user-settings.ts_. In both files you are using the same namespace.So, when you convert users.ts to users-module.ts and create the _users-module-shim.ts_ you would have:

* _users-module-shim.ts_: declaring a **namespace** `Application.Administration` + alias `Users`
* _user-settings.ts_: declaring a **namespace** `Application.Administration.Users`

**I**n this case Typescript would complain saying there is a duplicate identifier, one defining an alias and the other a namespace.  
Even if there wasn’t a compilation error you would have the problem that the types inside the alias would not merge inside of the namespace existing in _user-settings.ts_.

In such an scenario you would be forced to convert to modules all the files where that namespace was appearing before getting rid of the compiling errors.

#### Explicit definitions

The more natural option is the one of explicitly declaring all the export members from the typing inside the “shim” file. I suppose this would be also ok in most of the scenarios.  
****This has the advantage that the declaration of the original namespace `Application.Administration.Users` would make the contents of the shim merge as they where merging before with any other external file declaring that namespace.  
The clear disadvantage is that you have to manually declare each exported member in the “declare global” section inside the namespace. This is fine when you have, say 5 exports, but it’s not an elegant option when you have tens of exports.  
Still the compiler would tell you when some export is missing, so it might not be a big deal.  
  
Note: in my specific scenario I was using alias. The reason is that I could not define manually all the exports. This is probably a quite special scenario, but the file I was trying to migrate was generated by a software \(NSwag\) based in our API specification. That means a lot of exports with a loooot of type information \(interfaces and enums mainly\) that were not really under my control and I didn’t want to replicate manually in a “shim” file.  
So in that case the “alias” option worked like a charm for me.  
Also because the namespace of the API client was not used in any other file and had no merging implications, as I explained.

## Add namespaces runtime code

Where we were?... Oh yes, now the typings are fixed so Typescript is not complaining anymore. Unfortunately this was just part of the problem, and not a simple one, but we finally managed. Now when we try to run our app several runtime errors will appear.  
Let’s focus on them one by one. I’m not explaining them in the order you would encounter them but in an order which makes sense to follow the explanations.

In the previous chapter we have declared the definitions of the initial namespace. But as we said that was only a declaration for Typescript. TS will believe that code will be available at runtime and won’t generate any code for that.  
So now it’s the time to generate the code manually that we’ve promised TS it would be there.  
We will add that code in the same “shim” module file as it’s “part of the trick” that we are using and that file with this code could be removed once we have refactored all the dependees.

{% code-tabs %}
{% code-tabs-item title="users-module-shim.ts" %}
```typescript
import * as users from "path/to/file/users-module";

declare global {
  namespace Application.Administration {
    export import Users = users;
  }
}

// Runtime global code!: Needed for legacy global access to the module
window.Application = window.Application || {};
window.Application.Administration = window.Application.Administration || {};
window.Application.Administration.Users =
  window.Application.Administration.Users || {};

// You can create helper function to do the previous lines
// GenerateNamespace("Application.Administration.Users");

Application.Administration.Users = {
  ...Application.Administration.Users,
  ...users
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Ok, I hope this part is a bit more self explanatory. The first part of the code that we’ve added `window.Application = window.Application || {};` is kind of the similar thing that Typescript generates when we create a namespace in the global scope. It checks in every step if the object is already created so not to override it, if it’s not we just create an empty object.  
We do that until we have the whole namespace.

Ok, so we have the namespace created in the global scope. Now we have to add all the functionality which is exported from the refactored module:

```text
Application.Administration.Users = {
  ...Application.Administration.Users,
  ...users
};
```

You see? We are combining \(with ES6 destructuring\) the existing `Application.Administration.Users` namespace \(think that this object might not be empty as this namespace could be already existing in another file with some members that we don’t want to remove\) and then we merge it with all the members existing in our module `...users`.  
This is the manual implementation of “merging namespaces” which Typescript does for us.  
In fact you can create a helper function that does that for you. Check the extras section to see my implementation of it.

And that’s it! We have now a global namespace with all the members \(functions and variables\) which were initially there before the refactor. So the rest of the code will behave exactly the same when doing something like this:  
`Application.Administration.Users.createUser(info.user);`  
Remember our users-workflow.ts file right? That code will not know that all the members are really coming from inside a module, it will not feel the difference with the previous global namespace implementation.

I don’t know what you think, but from my point of view and my experience, creating the “runtime code” was a piece of cake compared with “generating the typings for TS”.  
So this is the end of the “shim” file and we should **do that for every file that we are converting to modules**.

## **Add a module loader**

So far we have added all the necessary typings and all the “obvious” runtime code. If you have followed me so far and make those changes in your code you will notice that your “shim” code and therefore our refactored module are not being executed when you load the webapp.  
No debugger or “alert” check is going to be triggered at any time.

The reason is because we need to load our code in our legacy app explicitly if we are using modules. You might think it should be trivial to load a module, but it isn’t, you need a special library for that, and it’s called **module loader**.

Now, as I said in the introduction, I’m assuming you load your files via “script” tags one by one or a single generated one \(as it was my case and I think the general case\) with the “out” property in the _tsconfig.json_ file.  
If the former case is your case you have only 2 possibilities for module loaders in a browser: **SystemJS or AMD.**  
  
You could use any of them but I chose to use SystemJS, because seemed more modern, long term maintained and had many features \(that I would never use; but who knows...\).

So we must tell TS now that we want to use SystemJS as a module loader. We can do that through our _tsconfig.json_ file:

{% code-tabs %}
{% code-tabs-item title="tsconfig.json" %}
```typescript
{
  "compilerOptions": {
    // ...
    "out": "Scripts/GeneratedTs.All.js",
    "target": "es5",
    "module": "System",
  },
  // ...
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

I’m also targeting “es5” as you see because I want my generated code to be compatible with older browsers like IE.

Now you might think that this is it. But we have just told TS that we want our modules to be loaded with SystemJS, what TS is not doing is adding the SystemJS library to your runtime.  
If you check the generated code you will see that TS have generated some code which uses a global variable called “System” and it is registering the modules like:

```typescript
System.register(
//...
)
```

If we had chosen AMD as a module loader you would see TS generating code like “define\(...\)” and of course expecting to exist a variable called “define” at runtime.

Good, so that “System” global variable is not existing yet. Let’s add SystemJS to our “script tags” dependencies.  
You can find the github project here:  
[https://github.com/systemjs/systemjs](https://github.com/systemjs/systemjs)

You will have to add 2 scripts. One is of course the system.js library:  
[https://github.com/systemjs/systemjs/blob/master/dist/system.min.js](https://github.com/systemjs/systemjs/blob/master/dist/system.min.js)  
You can use that file and add it to your codebase or use a CDN for that, up to you.

We will also add an “extra” of SystemJS called “**named register**” and it’s coming as a separate script.  
[https://github.com/systemjs/systemjs/blob/master/dist/extras/named-register.js](https://github.com/systemjs/systemjs/blob/master/dist/extras/named-register.js)  
You can check the doc but basically it’s going to allow us to load modules by a specific name that we declare, we’ll see that.

So we would have these 2 scripts added to our runtime like:

```markup
<script src="path/to/script/system.min.js"></script>
<script src="path/to/script/named-register.min.js"></script>
```

## **Loading the modules**

With the previous code now we have a global variable called System that will allow us to “register” and “import” \(load\) modules.  
****The “register” part is done by TS in the generated code as you can check by yourself. But there is no specific “import” of a module anywhere.  
This part would be done transparently by a “webpack” from the “entry point” of the application and we would never have to explicitly load a module.

As we are in a legacy app and we don’t have such fancy tools we will have to dig deep into the module loader \(SystemJS in our case\) API.

This will be accomplish by a System.import\(...\) syntax.

But before we start importing all the “shims” we’ve been created so far… let’s think for a minute what do we want to import. Well in our legacy all the code is loaded when we start the web app. So we will want just the same thing, import automatically everything when the application starts, importing all the “shim” files.

But I don’t want to pollute my code with a lot of System.import because of 2 reasons:

* System.import is very implementation-dependent and I would like to use the less the better. In case some day we decide other module loader \(like AMD\) we would have to change less code from a single place.
* Also the System.import requires naming a module with a specific name. So we would have to name every “shim” module to be able to load it.

That’s why we are going to create a module loader helper file like this one:

{% code-tabs %}
{% code-tabs-item title="global-module-loader.ts" %}
```typescript
///<amd-module name="global-module-loader"/>

import "./path/to/file/users-module-shim"
import "./path/to/file/another-module-shim"
import "./path/to/file/just-another-module-shim"
import "./path/to/file/refactor-another-module-shim"
// keep loading all the shims in this file
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Those are normal ES6 imports referencing relatively the path to the files. By the way this kind of imports without any name are called **side effects import**.  
This is called this way as we are just loading the modules expecting something to happen without taking care of the exports. This is precisely what our “shims” are doing, polluting the global scope with all our refactored modules without exporting anything.

I’m sure you also notice the first line of the loader file right?  
`///<amd-module name="global-module-loader"/>`  
Note: Don’t get confused about the amd-module syntax, even if it seems it’s something related with AMD modules that also works when using SystemJS.

This line is telling Typescript to generate the System.register runtime code with a specific name, in this case “**global-module-loader**”. This is the name we are going to be using to load this module.  
Remember the SystemJS named-register library we were loading earlier? This is where is going to be useful, otherwise our named System.import would not work at runtime even if we had a name for it.

And now we just have to import one single module “_global-module-loader.ts_” with the specified name. For that we will create a new file:

{% code-tabs %}
{% code-tabs-item title="immediate-module-loader.ts" %}
```typescript
/// <reference path="global-module-loader.ts" />

var ModulesLoaded = System.import("global-module-loader");
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Notice this file is not a module so it will be executed at “parse time” as the rest of our legacy code.  
So when the JS parser goes through that line is going to load our “global-module-loader” module, and therefore all the modules which are inside \(our “shims”\).

Have you noticed I save the result of System.import in a global scope variable named **ModulesLoaded**? This is also important and we will use it.

## **Waiting for the modules to load**

So we are almost finished! Only the very last part is left.

This issue might not happen to you, but I would say, sooner or later, after you increase your refactor it will happen. So I consider it part of the main workflow of fixes.  
****To understand the next issue I better will create an example and explain what would happen.

Remember the main dependee of our module? _admin-workflow.ts_  
Let’s add another call, also referencing another function of our refactored module.

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
namespace Application.Administration.Tasks {
  Application.Administration.Users.loadCache();

  function signIn(info: SignInModel) {
    // Other important steps to signIn

    // create the user
    Application.Administration.Users.createUser(info.user);
  }

  function processUser(user: Application.Administration.Users.User) {
    // do something
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Do you see that **loadCache** call? Imagine we are loading something in memory when the app starts.Have you noticed this call is not inside of a function right? This means that as soon as the JS parser reads “_admin-workflow.ts_” file it will execute the loadCache function inside `Application.Administration.Users` namespace.

You will see that when you load your app, that line is going to crash because neither the “namespace” nor the “loadCache” function inside are created yet.  
You might think, ok, we just have to make sure our “shim”, which will create the namespace for us, is executed before the _admin-workflow.ts_ file. That means executing our _immediate-module-loader.ts_ before the admin-workflow.ts.  
And for that we have already a TS “directive” called “_reference_” which will make sure of that right? So let’s add that to our _admin-workflow.ts_.

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
/// <reference path="/path/to/file/immediate-module-loader.ts" />

namespace Application.Administration.Tasks {
  Application.Administration.Users.loadCache();

  // ...
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Well done, now we are sure TS has generated the contents of “immediate-module-loader.ts” **before** “admin-workflow.ts”. This means we have already imported all the “shims” right? Remember the contents of our loader?

{% code-tabs %}
{% code-tabs-item title="immediate-module-loader.ts" %}
```typescript
var ModulesLoaded = System.import("global-module-loader");
```
{% endcode-tabs-item %}
{% endcode-tabs %}

System.import has been executed before our call to `Application.Administration.Users.loadCache();` so we are ready to go.  
But… wait a minute, I still have the same runtime error. The namespace is still not created even that System.import has been executed before.

And there it is when the variable **ModulesLoaded**  comes into play.  
You see, System.import returns a promise. That promise is executed asynchronously by definition by the browser \(it goes to the “event loop” of the browser even if the promise is already resolved; I was not aware of that to be honest when I encountered that issue\).  
So there is no order of generated code that can change that.

> **Every single piece of script tag is going to be executed BEFORE our modules have been imported**

So, when all the parsing has been finished then the browser will take care of the event loop and will see that it has some promises to resolve \(among them, our shim modules loading\).

So there is only one solution to this problem, let’s wait for the promise to be resolved for **all the code which is executed at “parse time”** and it’s dependent on our modules.  
Notice that this kind of error can only happen at “parse time”, for any other deferred execution based on user actions, ajax calls, etc, the shims will have already been loaded.

So this is what we will do for this kind of issues:

{% code-tabs %}
{% code-tabs-item title="admin-workflow.ts" %}
```typescript
/// <reference path="/path/to/file/immediate-module-loader.ts" />

namespace Application.Administration.Tasks {
  ModulesLoaded.then(() => {
    Application.Administration.Users.loadCache();
  });

  // ...
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Notice also that we keep the “_reference_” directive to the _immediate-module-loader.ts_  
This is because we want to be sure that ModulesLoaded global var is created before we use it.

