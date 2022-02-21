# Clojure Style Sheets

<img src="logo.png" width="155" height="68" alt="cljss logo" />

[CSS-in-JS](https://speakerdeck.com/vjeux/react-css-in-js) for ClojureScript

[![Clojars](https://img.shields.io/clojars/v/clj-commons/cljss.svg)](https://clojars.org/clj-commons/cljss)
[![cljdoc badge](https://cljdoc.org/badge/clj-commons/cljss)](https://cljdoc.org/d/clj-commons/cljss/CURRENT)
[![CircleCI](https://circleci.com/gh/clj-commons/cljss.svg?style=svg)](https://circleci.com/gh/clj-commons/cljss)

Ask questions on #cljss chat at [Clojuarians Slack](http://clojurians.net/)

<a href="https://www.patreon.com/bePatron?c=1239559">
  <img src="https://c5.patreon.com/external/logo/become_a_patron_button.png" height="40px" />
</a>

`[clj-commons/cljss "1.6.4"]`

## Table of Contents

- [Why write CSS in ClojureScript?](#why-write-css-in-clojurescript)
- [Features](#features)
- [How it works](#how-it-works)
- [Usage](#usage)
- [Composing styles](#composing-styles)
- [Development workflow](#development-workflow)
- [Production build](#production-build)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Supporters](#supporters)
- [License](#license)

## Why write CSS in ClojureScript?

Writing styles this way has the same benefits as writing components that keep together view logic and presentation. It all comes to developer efficiency and maintainability.

Thease are some resources that can give you more context:

- [“A Unified Styling Language”](https://medium.com/seek-blog/a-unified-styling-language-d0c208de2660) by Mark Dalgleish
- [“A Unified Styling Language”](https://www.youtube.com/watch?v=X_uTCnaRe94) (talk) by Mark Dalgleish
- [“The road to styled components: CSS in component-based systems”](https://www.youtube.com/watch?v=MT4D_DioYC8) by Glen Maddern

## Features

- Automatic scoped styles by generating unique names
- CSS pseudo-classes and pseudo-elements
- CSS animations via `@keyframes` at-rule
- CSS Media Queries
- Nested CSS selectors
- Injects styles into `<style>` tag at run-time
- Debuggable styles in development (set via `goog.DEBUG`)
- Fast, 1000 insertions in under 100ms

## How it works

### `defstyles`

`defstyles` macro expands into a function which accepts an arbitrary number of arguments and returns a string of auto-generated class names that references both static and dynamic styles.

```clojure
(defstyles button [bg]
  {:font-size "14px"
   :background-color bg})

(button "#000")
;; "-css-43696 -vars-43696"
```

Dynamic styles are updated via CSS Variables (see [browser support](http://caniuse.com/#feat=css-variables)).

### `defstyled`

`defstyled` macro accepts var name, HTML element tag name as a keyword and a hash of styles.

The macro expands into a function which accepts an optional hash of attributes and child components, and returns a React element. It is available for the Om, Rum and Reagent libraries. Each of them are in corresponding namespaces: `cljss.om/defstyled`, `cljss.rum/defstyled` and `cljss.reagent/defstyled`.

A hash of attributes with dynamic CSS values as well as normal HTML attributes can be passed into the underlying React element. Reading from the attributes hash map can be done via anything that satisfies `cljs.core/ifn?` predicate (`Fn` and `IFn` protocols, and normal functions).

_NOTE: Dynamic props that are used only to compute styles are also passed onto React element and thus result in adding an unknown attribute on a DOM node. To prevent this it is recommended to use keyword or a function marked with `with-meta` so the library can remove those props (see example below)._

- keyword value — reads the value from props map and removes matching attribute
- `with-meta` with a single keyword — passes a value of a specified attribute from props map into a function and removes matching attribute
- `with-meta` with a collection of keywords — passes values of specified attributes from props map into a function in the order of these attributes and removes matching attributes

```clojure
(defstyled h1 :h1
  {:font-family "sans-serif"
   :font-size :size ;; reads the value and removes custom `:size` attribute
   :color (with-meta #(get {:light "#fff" :dark "#000"} %) :color)} ;; gets `:color` value and removes this attribute
   :padding (with-meta #(str %1 " " %2) [:padding-v :padding-h])) ;; gets values of specified attrs as arguments and remove those attrs

(h1 {:size "32px" ;; custom attr
     :color :dark ;; custom attr
     :padding-v "8px" ;; custom attr
     :padding-h "4px" ;; custom attr
     :margin "8px 16px"} ;; normal CSS rule
    "Hello, world!")
;; (js/React.createElement "h1" #js {:className "css-43697 vars-43697"} "Hello, world!")
```

#### predicate attributes in `defstyled`

Sometimes you want toggle between two values. In this example a menu item can switch between active and non-active styles using `:active?` attribute.

```clojure
(defstyled MenuItem :li
  {:color (with-meta #(if % "black" "grey") :active?)})

(MenuItem {:active? true})
```

Because this pattern is so common there's a special treatment for predicate attributes (keywords ending with `?`) in styles definition.

```clojure
(defstyled MenuItem :li
  {:color "grey"
   :active? {:color "black"}})

(MenuItem {:active? true})
```

### pseudo-classes

CSS pseudo classes can be expressed as a keyword using parent selector syntax `&` which is popular in CSS pre-processors or as a string if selector is not a valid keyword e.g. `&:nth-child(4)`.

```clojure
(defstyles button [bg]
  {:font-size "14px"
   :background-color blue
   :&:hover {:background-color light-blue}
   "&:nth-child(3)" {:color "blue"}})
```

### Nested selectors

Sometimes when you want to override library styles you may want to refer to DOM node via its class name, tag or whatever. For this purpose you can use nested CSS selectors via string key.

```clojure
(defstyles error-form []
  {:border "1px solid red"
   ".material-ui--input" {:color "red"}})

[:form {:class (error-form)} ;; .css-817253 {border: 1px solid red}
 (mui/Input)] ;; .css-817253 .material-ui--input {color: red}
```

### `:css` attribute

`:css` attribute allows to define styles inline and still benefit from CSS-in-JS approach.

_NOTE: This feature is supported only for Rum/Sablono elements_

```clojure
(def color "#000")

[:button {:css {:color color}} "Button"]
;; (js/React.createElement "button" #js {:className "css-43697 vars-43697"} "Button")
```

### `defkeyframes`

`defkeyframes` macro expands into a function which accepts arbitrary number of arguments, injects @keyframes declaration and returns a string that is an animation name.

```clojure
(defkeyframes spin [from to]
  {:from {:transform (str "rotate(" from "deg)")
   :to   {:transform (str "rotate(" to "deg)")}})

[:div {:style {:animation (str (spin 0 180) " 500ms ease infinite")}}]
;; (js/React.createElement "div" #js {:style #js {:animation "animation-43697 500ms ease infinite"}})
```

### `font-face`

`font-face` macro allows to define custom fonts via `@font-face` CSS at-rule. The macro generates CSS string and injects it at runtime. The syntax is defined in example below.

_The macro supports referring to styles declaration in a separate \*.clj namespace._

```clojure
(require '[cljss.core :refer [font-face]])

(def path "https://fonts.gstatic.com/s/patrickhandsc/v4/OYFWCgfCR-7uHIovjUZXsZ71Uis0Qeb9Gqo8IZV7ckE")

(font-face
  {:font-family "Patrick Hand SC"
   :font-style "normal"
   :font-weight 400
   :src [{:local "Patrick Hand SC"}
         {:local "PatrickHandSC-Regular"}
         {:url (str path ".woff2")
          :format "woff2"}
         {:url (str path ".otf")
          :format "opentype"}]
   :unicode-range ["U+0100-024F" "U+1E00-1EFF"]})
```

### `inject-global`

`inject-global` macro allows to defined global styles, such as to reset user agent default styles. The macro generates CSS string and injects it at runtime. The syntax is defined in example below.

_The macro supports referring to styles declaration in a separate \*.clj namespace._

```clojure
(require '[cljss.core :refer [inject-global]])

(def v-margin 4)

(inject-global
  {:body     {:margin 0}
   :ul       {:list-style "none"}
   "ul > li" {:margin (str v-margin "px 0")}})
```

### CSS Media Queries

The syntax is specified as of [CSS Media Queries Level 4 spec](https://www.w3.org/TR/mediaqueries-4/).

```clojure
(require [cljss.core :as css])

(defstyles header [height]
  {:height     height
   ::css/media {[:only :screen :and [:max-width "460px"]]
                {:height (/ height 2)}}})
```

- Supported media types: `#{:all :print :screen :speech}`
- Modifiers: `#{:not :only}`

More examples of a query:

#### Boolean query

```clojure
[[:monochrome]]
```

#### Simple conditional query

```clojure
[[:max-width "460px"]]
```

#### Multiple (comma separated) queries

```clojure
[[:screen :and [:max-width "460px"]]
 [:print :and [:color]]]
```

#### Basic range query

_Supported operators for range queries_ `#{'= '< '<= '> '>=}`

```clojure
'[[:max-width > "400px"]]
```

#### Complex range query

```clojure
'[["1200px" >= :max-width > "400px"]]
```

## Usage

`(defstyles name [args] styles)`

- `name` name of a var
- `[args]` arguments
- `styles` a hash map of styles definition

```clojure
(ns example.core
  (:require [cljss.core :refer-macros [defstyles]]))

(defstyles button [bg]
  {:font-size "14px"
   :background-color bg})

[:div {:class (button "#fafafa")}]
```

`(defstyled name tag-name styles)`

- `name` name of a var
- `tag-name` HTML tag name as a keyword
- `styles` a hash map of styles definition

Using [Sablono](https://github.com/r0man/sablono) templating for [React](https://facebook.github.io/react/)

```clojure
(ns example.core
  (:require [sablono.core :refer [html]]
            [cljss.rum :refer [defstyled]]))

(defstyled Button :button
  {:padding "16px"
   :margin-top :v-margin
   :margin-bottom :v-margin})

(html
  (Button {:v-margin "8px"
           :on-click #(console.log "Click!")}))
```

Dynamically injected CSS:

```css
.css-43697 {
  padding: 16px;
  margin-top: var(--css-43697-0);
  margin-bottom: var(--css-43697-1);
}
.vars-43697 {
  --css-43697-0: 8px;
  --css-43697-1: 8px;
}
```

## Composing styles

Because CSS is generated at compile-time it's not possible to compose styles as data, as you would normally do it in Clojure. At run-time, in ClojureScript, you'll get functions that inject generated CSS and give back a class name. Hence composition is possibly by combining together those class names.

```clojure
(defstyles margin [& {:keys [x y]}]
  {:margin-top    y
   :margin-bottom y
   :margin-left   x
   :margin-right  x})

(defstyles button [bg]
  {:padding "8px 24px"
   :background-color bg})

(clojure.string/join " " [(button "blue") (margin :y "16px")]) ;; ".css-817263 .css-912834"
```

## Development workflow

If you see that styles are not being reloaded, add `(:require-macros [cljss.core])`, this should do the job.

When developing with Figwheel in order to deduplicate styles between reloads it is recommended to use Figwheel's `:on-jsload` hook to clean injected styles.

```clojure
:figwheel {:on-jsload example.core/on-reload}
```

```clojure
(ns example.core
  (:require [cljss.core :as css]))

(defn on-reload []
  (css/remove-styles!)
  (render-app))
```

_NOTE: don't forget that once styles were removed you have to re-inject `defkeyframes`, `font-face` and `inject-global` declarations_

## Production build

Set `goog.DEBUG` to `false` to enable fast path styles injection.

```clojure
{:compiler
 {:closure-defines {"goog.DEBUG" false}}}
```

_NOTE: production build enables fast pass styles injection which makes those styles invisible in `<style>` tag on a page._

## Roadmap

- Server-side rendering

## Contributing

- Pick an issue with `help wanted` label (make sure no one is working on it)
- Stick to project's code style as much as possible
- Make small commits with descriptive commit messages
- Submit a PR with detailed description of what was done

## Development

A repl for the example project is provided via [lein-figwheel](https://github.com/bhauman/lein-figwheel).

```
$ cd example
$ lein figwheel
```

If using emacs [cider](https://github.com/clojure-emacs/cider) - you can also launch the repl using `M-x cider-jack-in-clojurescript`.

## Testing

cljss uses a combination of Clojure and ClojureScript tests. Clojure tests are run via `lein test` and ClojureScript tests are run via [doo](https://github.com/bensu/doo). ClojureScript tests require a valid environment in order to run - [PhantomJS](http://phantomjs.org/) being the easiest to install.

Once a valid environment is setup, ClojureScript tests can be run like so:

```
$ lein doo phantom test once
```

Or with file watching:

```
$ lein doo phantom test
```

To run Clojure and ClojureScript tests at once use the `test-all` task:

```
$ lein test-all
```

## Supporters

[<img src="https://avatars3.githubusercontent.com/u/636651?s=460&v=4" width="100px;" />](https://github.com/brianium)
[<img src="https://avatars1.githubusercontent.com/u/193187?s=400&v=4" width="100px;" />](https://github.com/sherbondy)

## License

Copyright © 2017 Roman Liutikov

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
