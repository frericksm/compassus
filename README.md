# Compassus [![CircleCI](https://circleci.com/gh/compassus/compassus.svg?style=svg)](https://circleci.com/gh/compassus/compassus)

A routing library for Om Next.

Read the [announcement blog post](https://anmonteiro.com/2016/06/the-quest-for-a-unified-routing-solution-in-om-next/).

## Contents

- [Installation](#installation)
- [Guide](#guide)
  - [Declaring routes](#declaring-routes)
  - [Assembling a Compassus application](#assembling-a-compassus-application)
    - [Implementing the parser](#implementing-the-parser)
    - [Mixins](#mixins)
    - [Utility functions](#utility-functions)
  - [Changing routes](#changing-routes)
  - [Integrating with browser history](#integrating-with-browser-history)
    - [Bidi + Pushy example](#bidi--pushy-example)
    - [Secretary + `goog.History` example](#secretary--googhistory-example)
- [Documentation](#documentation)
- [Companies using Compassus](#companies-using-compassus)
- [Copyright & License](#copyright--license)

## Installation

### Stable

Leiningen dependency information:

```clojure
[compassus "0.2.1"]
```

Maven dependency information:

```xml
<dependency>
  <groupId>compassus</groupId>
  <artifactId>compassus</artifactId>
  <version>0.2.1</version>
</dependency>
```

### Pre-release

Grab a pre-release version of the latest features by depending on the following:
```clojure
[compassus "0.3.0-SNAPSHOT"]
```

## Guide

To get started, require Compassus somewhere in your project.

```clojure
(ns my-app.core
  (:require [om.next :as om :refer-macros [defui]]
            [compassus.core :as compassus]))
```

### Declaring routes

Your application's routes are represented by a map of keywords (the route handlers of your application) to the respective Om Next component classes. The following example shows the routes for a simple application that has 2 routes, `:index` and `:about`:

```clojure
(defui Index
  ...)

(defui About
  ...)

(def routes
  {:index Index
   :about About})
```

To specify the initial route of the application, wrap its component class in a `index-route` call as shown below.

```clojure
(def routes
  ;; :index is the initial route of the application
  {:index (compassus/index-route Index)
   :about About})
```

Routes can also be idents. Below is an example route definition that uses an ident as the route key.

``` clojure
(defui Item
  ...)

(defui ItemList
  ...)

{:items (c/index-route ItemList)
 [:item/by-id 0] Item}
```

### Assembling a Compassus application

Creating a Compassus application is done by calling the `application` function. This function accepts a configuration map that should contain your routes and the options to pass to the Om Next reconciler. Compassus will create the reconciler for you. Here's an example:

``` clojure
(def app
  (compassus/application
    {:routes {:index (compassus/index-route Index)
              :about About}
     :reconciler-opts {:state {}
                       :parser (om/parser {:read read))}}))
```

#### Implementing the parser

The parser is a required parameter to an Om Next reconciler. As such, a Compassus application also needs to have one, the advantage being that most of the plumbing has been done for you. The parser in a Compassus application will dispatch on the current route. Therefore, all that is required of a parser implementation is that it knows how to handle the routes that your application will transition to. An example is shown below with routes we have previously declared.

``` clojure
;; we declared routes for `:index` and `:about`.
;; our parser should dispatch on those keys:

(defmulti read om/dispatch)

(defmethod read :index
  [env k params]
  {:value ...
   :remote ...})

(defmethod read :about
  [env k params]
  {:value ...
   :remote ...})
```

For convenience, the parser's `env` argument contains a `:route` key with the current route of the Compassus application.

#### Mixins

The configuration map you pass to `compassus.core/application` can also contain an
optional `:mixins` key. Its value should be a vector of mixins. Mixins hook into
the generated Compassus root component's functionality in order to extend its capabilities
or change its behavior. The currently built-in mixin constructors are:

##### **`compassus.core/wrap-render`**:

Constructs a mixin that will wrap all the routes in the application. It becomes
useful to specify this mixin whenever you want to define common presentation logic
for all the routes in your Compassus application. `wrap-render` takes a function
or an Om Next component **class**. The component class may or may not implement
`om.next/IQuery` — a query that refers to data that you want to have available to
the wrapper, e.g. the current logged in user and such.

The wrapper will be passed a map with the keys below. If the wrapper is a simple
function, the map will be passed as argument. If it is a component class, the map
will be in the component's props (accessible through `om.next/props`). If the wrapper
implements `om.next/IQuery`, its props will be the data that the query asks for.
You'll find the map with the keys below in the component's computed props, accessible
through `om.next/get-computed`.

  - :owner   - the parent component instance
  - :factory - the component factory for the current route
  - :props   - the props for the current route.

*Note*: There is only supposed to be one `wrap-render` mixin under the `:mixins`
key. If there are more than one, Compassus will only use the first it finds.

Example:

``` clojure
;; Wrapper that doesn't implement `om.next/IQuery`
(defui Wrapper
  Object
  (render [this]
    ;; the wrapper doesn't implement `om.next/IQuery`, so the map is accessible
    ;; through `om.next/props`:
    (let [{:keys [owner factory props]} (om/props this)]
      ;; implement common presentation logic for all routes
      ;; call the given factory with props in the end
      (factory props))))

(def wrapper (om/factory Wrapper))

(def app
  (compassus/application
    {:routes ...
     :reconciler-opts ...
     :mixins [(compassus/wrap-render wrapper)]}))


;; Example of a wrapper that implements `om.next/IQuery`
(defui Wrapper
  static om/IQuery
  (query [this]
    [:current-user])
  Object
  (render [this]
    ;; `:current-user` will be in props, the other keys will be in computed props:
    (let [{:keys [current-user]} (om/props this)
          {:keys [owner factory props]} (om/get-computed this)]
      (dom/div nil
        ;; implement common presentation logic for all routes
        (dom/p nil (str "Current logged in user: " current-user))
        ;; call the given factory with props in the end
        (factory props)))))
```

##### **`compassus.core/will-mount`**:

Constructs a mixin that will hook into the `componentWillMount` lifecycle method
of the generated root component. Takes a function which will receive the component
as argument. Useful to perform any setup before the Compassus application mounts.

Example:

``` clojure
(compassus.core/will-mount
  (fn [self]
    ;; sets a property in the state of the root component
    (om/set-state! self {:foo 42})))
```

##### **`compassus.core/did-mount`**:

Constructs a mixin that will hook into the `componentDidMount` lifecycle method
of the generated root component. Takes a function which will receive the component
as argument. Useful to perform any setup after the Compassus application mounts.

Example:

```clojure`
(compassus.core/did-mount
  (fn [self]
    (start-analytics!)))
 ```

##### **`compassus.core/will-unmount`**:

Constructs a mixin that will hook into the `componentWillUnmount` lifecycle method
of the generated root component. Takes a function which will receive the component
as argument. Useful to perform any cleanup after the Compassus application unmounts.

Example:

``` clojure
(compassus.core/will-unmount
  (fn [self]
    (stop-analytics!)))
```

**A note on mixins**

Mixins are just data. Compassus built-in mixin constructors are just helpers around
assembling this data. For example, building a mixin to hook into the Compassus root
component's query could also be done as shown below:

``` clojure
(def app
  (compassus/application
    {:routes ...
     :reconciler-opts ...
     :mixins [{:render MyWrapper}]}))
```

#### Utility functions

There are a few utility functions in `compassus.core`. Below is a description of these functions along with simple examples of their usage.

##### **`root-class`**

Return the Compassus application's root class.

``` clojure
(compasssus/root-class app)
```

##### **`mount!`**

Mount a compassus application in the DOM.

``` clojure
(compassus/mount! app (js/document.getElementById "app"))
```

##### **`get-reconciler`**

Get the reconciler for the Compassus application.

``` clojure
(compassus/get-reconciler app)
```

##### **`application?`**

Returns true if the argument is a Compassus application.

``` clojure
(compassus/application? app)
;; true
```

##### **`current-route`**

Returns the current application route.

``` clojure
(compassus/current-route app)
```

##### **`compassus-merge`**

By default, Compassus uses Om's `default-merge` function to merge remote responses into the application state. If your server responses are keyed by the current route, use `compassus-merge` as the `:merge` in the reconciler. It will look for the current route in the remote response and merge that into the application state instead.

``` clojure
(compassus/application
  {:routes ...
   :reconciler-opts {:state ...
                     :parser ...
                     :merge compassus/compassus-merge}})
```

### Changing routes

To change the current route of a Compassus application, call the function `set-route!`. An example follows:

``` clojure
;; the argument to `set-route!` can be one of: a Compassus application, an
;; Om Next component or an Om Next reconciler

(compassus/set-route! app :about)

(compassus/set-route! reconciler :about)

(compassus/set-route! this :about)
```

The `set-route!` function can take an additional argument, a map with the following
supported options:

- `queue?`: boolean indicating if the application root should be queued for re-render.
Defaults to true.
- `params`: map of parameters that will be merged into the application state.
- `tx`: transaction(s) (e.g.: `'(do/it!)` or `'[(do/this!) (do/that!)]`) that will
be run after the mutation that changes the route. Can be used to perform additional
setup for a given route (such as setting the route's parameters).

Examples:

``` clojure
;; don't re-render from root
(compassus/set-route! app :about {:queue? false})

;; don't re-render from root, but re-read `:friends/list`
(compassus/set-route! app :about {:queue? false
                                  :tx [:friends/list]})

;; merge `{:route-params {:id 42}}` into the app-state
(compassus/set-route! app :about {:params {:route-params {:id 42}}})

;; run the `set-route-params!` tx after changing route.
(compassus/set-route! app :about {:tx '[(set-route-params! {:id 42})]})
```

### Integrating with browser history

URL (or path) navigation is an orthogonal concern to routing in Om Next components,
which is mainly about swapping components in and out according to the selected route.
However, it might be desirable for applications to setup history navigation only
when the application mounts. In addition, applications might also want to teardown
history if the application unmounts from the DOM. This can easily be achieved in
Compassus through the use of the `did-mount` and the `will-unmount` mixins.

Below are two examples, one using [Bidi](https://github.com/juxt/bidi) and
[Pushy](https://github.com/kibu-australia/pushy), and another using
[Secretary](https://github.com/gf3/secretary) and [`goog.History`](http://google.github.io/closure-library/api/class_goog_History.html).

#### Bidi + Pushy example

``` clojure
(ns my-ns
  (:require [om.next :as om :refer-macros [defui]]
            [compassus.core :as compassus]
            [bidi.bidi :as bidi]
            [pushy.core :as pushy]))

(def bidi-routes
  ["/" {""      :index
        "about" :about}])

(declare app)

(def history
  (pushy/pushy #(compassus/set-route! app (:handler %))
    (partial bidi/match-route bidi-routes)))

(def app
  (compassus/application
    {:routes  {:index (compassus/index-route Index)
               :about About}
     :mixins [(compassus/did-mount (fn [_]
                                     (pushy/start! history)))
              (compassus/will-unmount (fn [_]
                                        (pushy/stop! history)))]}))
```

#### Secretary + `goog.History` example

``` clojure
(ns my-ns
  (:require [om.next :as om :refer-macros [defui]]
            [compassus.core :as compassus]
            [secretary.core :as secretary]
            [goog.history.EventType :as EventType]
            [goog.events :as evt]])
  (:import goog.History))

(declare app)

(defroute index "/" []
  (compassus/set-route! app :index))

(defroute about "/about" []
  (compassus/set-route! app :about))

(def event-key (atom nil))
(def history
  (History.))

(def app
  (compassus/application
    {:routes  {:index (compassus/index-route Index)
               :about About}
     :mixins [(compassus/did-mount (fn [_]
                                     (reset! event-key
                                       (evt/listen history EventType/NAVIGATE
                                         #(secretary/dispatch! (.-token %))))
                                     (.setEnabled history true)))
              (compassus/will-unmount (fn [_]
                                        (evt/unlistenByKey @event-key)))]}))
```

## Documentation

There's API documentation [here](https://compassus.github.io/compassus/doc/0.2.1/).

There are also some devcards examples [here](https://compassus.github.io/compassus/devcards/). Refer to their [source](./src/devcards/compassus/devcards/core.cljs) for more information.

## Companies using Compassus

- [CircleCI](https://circleci.com/), source [here](https://github.com/circleci/frontend)

## Copyright & License

Copyright © 2016 António Nuno Monteiro

Distributed under the Eclipse Public License (see [LICENSE](./LICENSE)).
