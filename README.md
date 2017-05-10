# Colony Case Study: Isomorphic Apps Without Node

## Background

Colony is a global film streaming platform connecting content owners with passionate fans through exclusive extra content. In a landscape of significant competition, our go-to-market strategy relies heavily on world-class product and user experience, with ambitions to redefine the way film is consumed online. An important differentiation between our video-on-demand model and that of competitors like Netflix is our transactional business model: this means that content on our platform is open to the public to browse, and purchase on a pay-as-you-go basis, and isn’t hidden behind a subscription paywall. We benefit from letting Google crawl and index our entire catalogue as a result. On top of this, the nature of our product requires a dynamic user interface that must frequently update to reflect changes to a complex application state. For example, elements must update to reflect whether the user is signed in or not, or whether the content bundle (or parts of it) is owned, or at a certain point in its rental period. While our initial prototype was built using out of the box ASP.NET MVC, we soon realised that turning our front-end into a “single page app” would vastly improve our ability to deal with these challenges, while also enabling our back-end and front–end teams to work independently of one another by “decoupling” the stack.

We were therefore faced with the dilemma of how to combine a traditional server-rendered and indexable website, with a single page app. At the time in 2014, isomorphic frameworks like Meteor provided this sort of functionality out of the box, but required a Node.js backend. SPA frameworks like Angular and Ember were already ubiquitous but came hand-in-hand with SEO drawbacks that we wanted to avoid.

Our back-end had already been built in ASP.NET MVC, and two thirds of our development team was made up of C# developers. We wondered if there was a way we could achieve something similar to an isomorphic application, but without a Node.js backend, and without having to rebuild our entire application from scratch. How could C# and JavaScript - technologies which are typically viewed as separate and incompatible - be brought together? In other words - could we build an Isomorphic application without Node.js?

Solving this problem would also provide us with numerous performance benefits and the ability to instantly render the application in very specific states - for example, linking to a film page with its trailer open, or rendering the checkout journey at a specific point. As most of this UI takes place within “modals” in our design, these are things which would traditionally be rendered by JavaScript and could be managed only in the front-end.

We wanted the ability to server render any page or view in any particular state based on its URL, and then have a client-side JavaScript application start up in the background to take over the rendering and routing from that point on. For this to happen we would firstly need a common templating language and shared templates. Secondly, we would need a common data structure for expressing the application state.

Up until this point, view-models had been the responsibility of the back-end team, but with an ideal decoupled stack, all view design and templating would be the responsibility of the front-end team, including the structure and naming of the data delivered to the templates.

From the perspective of removing any potential duplication of effort or code, we pondered the following questions:

1. Was there a templating language with implementations in both C# and JavaScript? For this to be conceivable, it would need to be “logicless”, and therefore not allow any application code in the templates in the way that C# Razor or PHP does.
2. If a component was needed by both the front and back-end, could it be expressed in a language-agnostic data format such as JSON?
3. Could we write something (for instance, a view-model) in one language and automatically transpile it to another?

## Templating

We began by looking at templating. We were big fans of Handlebars due to its strict “logicless” nature and its intentionally limited scope. Given a template, and a data object of any structure, Handlebars will output a rendered string. As it is not a full-blown view engine, it can be used to render small snippets, or integrated into larger frameworks to render an entire application. Its logicless philosophy restricts template logic to `#if`, `#unless`, and `#each`. The idea being that if anything more complex is needed, the template is not the place for it.

Although originally written for JavaScript, Handlebars can be used with any language for which an implementation exists and as a result, implementations now exist for almost every popular server-side programming language. Thankfully for us, a well-maintained open source implementation for C# existed in the form of the excellent Handlebars.Net.

Our front-end views had been built in a very modular way, meaning that rather than using monolithic page templates, any particular view was made up from a sequence of reusable “modules” that could in theory function in any order or context.  With our previous server-side view rendering solution in Razor, this concept didn’t translate well from the front-end to the back-end, as our back-end team would need to take the static modules, stitch them together into larger constructs, and insert the necessary logic and template tags as desired. With a shared templating language and data structure, however, there was no reason why we couldn’t now share these individual modules between both sides of the stack and render them in a desired order using either C# or JavaScript, with identical results. The process of designing a module, templating it, and making it dynamic, could then be done entirely by the front-end team.

## Data

One issue with our previous solution was the ad-hoc nature of our view-models in our prototype, which evolved organically per template as features were added. There was never a big picture of the overall application data-structure with data often being exposed to some templates and not others, requiring effort and care from both teams when creating new templates.

To avoid this problem early on, we designed a “global” data structure for the whole application, that could represent the site, a resource and the application state in a single object.  On the back-end, this structured data would be delivered from the database, mapped into a view model and delivered to handlebars to render a view. On the front-end, the data would firstly be received over a REST API and then delivered to Handlebars. In each case however, the final data provided to handlebars would be identical, and therefore also the output.

The data structure we decided on was as follows. This object can be thought of a state container for the entire application at any point in time.

```
{
    global: {...},
    config: {...},
    entry: {...},
    user: {...},
    state: {...},
    modules: {...}
}
```
> Our application’s top-level data structure

The `global` object holds content-centric data pertaining to the entire website or application, irrespective of the resource that’s being viewed. Things like static text, open graph and meta content are contained here.

The `config` object holds configuration-centric data pertaining to the entire website or application, irrespective of the resource that’s being viewed. Things like the environment variables and Google Analytics IDs are contained here.

The `entry` object holds data pertaining to the current resource being viewed (e.g. a page, or a particular film). As a user navigates around the site, the requested resource’s data would be pulled in from a specific endpoint, and the entry property would be updated as needed.

``` handlebars
<head>
    ...
    <title>
        {{#if entry.title}}
            {{entry.title}} | 
        {{/if}}

        {{global.title}}
    </title>
</head>
```
> An example of the `global` and `entry` objects in use in a template

The `user` object holds non-sensitive data pertaining to the signed in user, such as their name, avatar and email address.

``` handlebars
<aside>
    <h3>Hello {{user.firstName}}!</h4>
    
    {{#if user.hasPurchases}}
        <h4>Your Recent Purchases</h4>
        
        {{#each user.recentPurchases}}
            ...
        {{/each}}
    {{/if}}
</aside>
```
> An example of the `user` object in use in a template

The `state` object is used to reflect the current rendered state of the application, which would be crucial to our ability to render complex states on the back-end via a URL. For example, a single view could be rendered with a particular tab selected, a modal open, or an accordion expanded, as long as that state had its own URL.

``` handlebars
<nav>
    <a href="{{entry.url}}/extras/" class="tab{{#if state.isExtrasTab}} tab__active{{/if}}">Extras</a>

    <a href="{{entry.url}}/about/" class="tab{{#if state.isAboutTab}} tab__active{{/if}}">About</a>
</nav>
```
> An example of the `state` object in use in a template

The `module` is used when data must be delivered to a specific module without exposing it to other modules. For example, we may need to iterate through a list of items within an entry (for example, a gallery of images), rendering an image module for each one, but with different data. Rather than that module having to pull its data out of the entry using its index as a key, the necessary data can be delivered directly to it via the `module` object.

```
<figure>
    <img src="{{module.image.url}}" alt="{{module.image.altText}}"/>
    
    {{#if module.description}}
        <figcaption>
            <p>{{module.description}}</p>
        </figcaption>
    {{/if}}
</figure>
```
> An example of the `module` object in use in a template

## Transpiled View Models

With our top level data-structure defined, the internal structure of each of these objects would now need to be defined. With C# being the strictly-typed language that it is, arbitrarily passing dynamic object literals around in the typically loose JavaScript style would not cut it. Each view-model would need strictly-defined properties with their respective types. 

As we wanted to keep view-model design the responsibility of the front-end, we decided that these should be written in JavaScript. This would also allow front-end team members to easily test the rendering of templates (for example, using a simple Express development app) without needing to integrate them with the .NET back-end.

JavaScript classes could be used to define the structure of each model, with default values used to infer type.

``` js
class Bundle extends Product {
    constructor() {
        super();

        this.title      = '';
        this.director   = '';
        this.synopsis   = '';
        this.artwork    = new Image();
        this.trailer    = new Video();
        this.film       = new Video();
        this.extras     = [new Extra()];
        this.isNew      = false;
        ...
    }
}
```
> A typical JavaScript view model in our application describing a `Bundle`, and inheriting from another model called `Product`. 

Templates often require the checking of multiple pieces of data before showing or hiding a particular element. To keep templates clean however, `#if` statements in Handlebars may only evaluate a single property, and comparisons are not allowed without custom helper functions which in our case would have to be duplicated in both JavaScript and C#. While more complex logic can be achieved by nesting logical statements, this creates unreadable and unmaintainable templates, and is against the philosophy of logicless templating. We needed to decide where the additional logic needed in these situations would live.

Thanks to the addition of “getters” in ES5 JavaScript, we were able to easily add dynamically evaluated properties to our constructors that could be used in our templates, which proved to be the perfect place to perform more complex evaluations and comparisons.

``` js
class Bundle extends Product {
    constructor() {
        ...
        this.trailer = new Video();
        this.isNew   = false;
        ...
        Object.defineProperty(this, 'hasWatchTrailerBadge', {
            get() {
                return this.isNew && this.trailer !== null;
            }
        });
    }
}
```
> An example of a dynamic ES5 “getter” property on a view model, evaluating two other properties from the same model.

We now had all of our view models defined with typed properties and getters, but the problem remained that they existed only in JavaScript. Our first approach was to manually rewrite each of them in C#, but we soon realised that this was a duplication of effort, unscaleable and highly error-prone. We felt like we could automate this process if only we had the right tools. Could we somehow “transpile” our JavaScript constructors into C# classes?

We decided to try our hand at creating a simple Node.js app to do just this. Using enumeration, type-checking and prototypal inheritance we were able to create descriptive “manifests” for each of our constructors. These manifests contained information such as the name of the class, and what classes, if any, did it inherit from. At the property level, they contained the names and types of all properties, whether or not they were getters, and if so, what the getter should evaluate and what type should it return.

With each view model parsed into a manifest, the data could now be fed into (ironically) a Handlebars template of C# class, resulting in a collection of production-ready .cs files for the back-end, each describing a specific view model.

``` cs
namespace Colony.Website.Models
{
    public class Bundle : Product
    {
        public string title { get; set; }

        public string director { get; set; }

        public string synopsis { get; set; }

        public Image artwork { get; set }

        public Video trailer { get; set; }

        public Video film { get; set; }

        public List<Extra> extras { get; set; }

        public Boolean isNew { get; set; }

        public bool hasWatchTrailerBadge
        {
            get
            {
                return this.isNew && this.trailer != null;
            }
        }
    }
}
```

> The same “Bundle” view-model, transpiled into C#

As our the requirements of the transpiler and the needs of the project evolved, our transpiler evolved into a full Javascript-based AST parser which enabled us to deal with the transpilation of getter function logic more accurately and robustly.

## Layouts

We now needed a way of defining which modules should be rendered for a particular view, and in what order.

As a highly generic and changeable part of the architecture, we wondered if we could express each view as a JSON file that again could be shared between the front-end and back-end, thus avoiding duplication of these lists in C# and JavaScript.


``` json
[
    "head",
    "header",
    "welcome",
    "film-collection",
    "sign-up-cta",
    "footer",
    "foot"
]
```
> A simple JSON file describing a possible layout of a home page

While listing the names of modules in a particular order was simple, we wanted the ability to conditionally render modules only when specific conditions were met. For example, a user sidebar should only be rendered if a user is signed in.

Taking inspiration from Handlebars’ limited set of available logic (#if, #unless, #each), we realised we could express everything we needed to in JSON by referring to properties within the aforementioned global data structure data:

```
[
    ...
    {
        "name": "user-sidebar",
        "if": ["user.isSignedIn"]
    },
    {
        "name": "modal",
        "if": ["state.isTrailerOpen"],
        "import": "entry.trailer"
    }
]
```

By restructuring the format of the layout, we now had the ability to express simple logic, and import arbitrary data into modules. Note that the `if` "directives" take the form of arrays to allow the evaluation of multiple properties.

Taking things further, we began to use this format to describe more complex view structures where modules could be nested within other modules using a tree structure which would correlate with the resulting DOM:


```
[
    ...
    {
        "name": "tabs-nav",
        "unless": ["entry.userAccess.isLocked"]
    },
    {
        "name": "tab-container",
        "unless": ["entry.userAccess.isLocked"]
        "layout": [
            {
                "name": "explore-tab",
                "if": ["state.isExploreTabActive"],
                "layout": [
                    {
                        "name": "extra-slider",
                        "each": "entry.extrasSliders"
                    }
                ]
            },
            {
                "name": "about-tab",
                "if": ["state.isAboutTabActive"]
            }
        ]
    }
    ...
]
```

The ability to express the nesting of layouts allowed for complete freedom in the structuring of markup, and a huge reduction of logic within template files.

We had now arrived at a powerful format for describing the layout and structure of our views, with a considerable amount of logic available. Had this logic been written in either JavaScript or C#, it would have required tedious and error-prone manual duplication.

## Renderer Implementation

To arrive at our final rendered HTML on either side of the stack, we now needed to take our templates, layouts and data, and combine them all together to produce a view.

This was a piece of functionality that we all agreed would need to be duplicated, and would need to exist in both a C# and JavaScript form. We designed a specification for a collection of classes which would build up data, parse the layout tree, and recursively iterate through it rendering out each module with its respective data as per the conditions in the layout file, finally returning a rendered HTML string.

As we needed to pay close attention to making sure that both implementations would return identical output, we wrote both the C# and JavaScript renderers as group programming exercises involving the whole development team, which also provided a great learning opportunity for front-end and back-end team members to learn about each other’s technology.

## Front-end Single Page App

Our development team culture had always been to build as much as we could in-house, and own our technology as a result. When it came to developing the first iteration of our single page app in 2014, we had the choice of using an off-the-shelf framework like Angular, Backbone or Ember, or building our own. With our templating taken care of, and our top-level data structure over the API forming our application state, we still needed a few more components to manage routing, data-binding and user interface components. We strongly believed, and still do, in the concept of “unobtrusive” JavaScript, so the blurring of HTML and JavaScript in Angular (and now React's JSX) was something we wanted to avoid. We also realised that trying to retrofit an opinionated framework on to our now quite unique architecture, would result in significant hacking and a fair amount of redundancy in the framework. We therefore decided to build our own solution, with a few distinct principles in mind. Firstly, UI behaviour should not be tightly coupled to specific markup. Secondly, the combination of changes to the application state and the handlebars logic already defined in our templates and layouts should be enough to enable dynamic re-rendering of any element on the page at any time. 

The solution we arrived at is not only extremely lightweight, but also extremely modular. At the lowest level we have our state and fully rendered views delivered from the server, where we can denote arbitrary pieces of markup to “subscribe” to the creation of a new immutable state. These changes are then diffed at a data-level and signalled using DOM events which (while slightly more "hands-on") we found to be much more performant and lighweight than an Angular-like digest loop, experimental “Observable” implementations, or virtual DOM diffing. When a change happens, that section of the DOM is re-rendered and replaced.

A level up, our user interface “behaviors” are entirely separate from this process, effectively progressively enhancing arbitrary pieces of markup. For example, we could apply the same “slider” UI behavior to a variety of different components, each with entirely different markup - in one case a list of films, in another case a list of press quotes.

At the highest level, a History API enabled router was built to intercept all link clicks, and determine which layout file was needed to render the resulting view.

## Final Architecture

[ILLUSTRATION PENDING]

This schematic illustrates the full lifecycle of both the initial server-side request and all subsequent client-side requests, with the components used for each.

## Next Steps

While several solutions have now started to arise allowing the rendering of Angular applications and React components on non-JavaScript engines (such as ReactJS.NET and React-PHP-V8JS), our solution is still unique in the ASP.NET ecosystem due to its lack of dependency on a particular framework. While our back-end team did have to write some additional code (such as the Renderer) during the development of the solution, their data services, controllers, and application logic were left more or less untouched. The amount of front-end UI code they were able to remove from the back-end as a result of decoupling has lead to a leaner back-end application with a clearer purpose. While our application is in one sense “isomorphic”, it is also cleanly divided, with a clear separation of concerns.

We imagine that there are many other development teams out there, who like us, desire the benefits of an isomorphic application, but who feel restrained by their choice of back-end technology - whether that be C#, Java, Ruby, or PHP. In these cases, decisions made during the infancy of a company, are now most likely deeply ingrained in the product architecture, and have influenced the structure of the development team. Our solution shows that isomorphic principles can be applied to any stack, without having to introduce a dependency on a front-end framework that may be obsolete in a year’s time, and without having to rebuild an application from scratch. We hope other teams take inspiration from the lessons we’ve learned and we look forward to seeing isomorphic applications across a range of technologies in the future.

---
© 2016 Colony Ltd / Patrick Kunka
