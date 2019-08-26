---
author: "Grant Peltier"
date: 2019-08-25
title: ClojureScript Context Windows
tags: [CLJS]
---

# Background
If you are familiar with ClojureScript development, you are likely familiar with
[Reagent](https://github.com/reagent-project/reagent), the CLJS wrapper for
React. Furthermore, if your Reagent application has grown and become more
complex you have probably looked into state management solutions like 
[re-frame](https://github.com/Day8/re-frame). Further still, if you use
Re-Frame, you have probably also tried out a debugging tool like 
[re-frisk](https://github.com/flexsurfer/re-frisk). Re-frisk is a wonderful
debugging tool that I whole-heartedly recommend to analyze your re-frame events
and subscriptions. This article is primarly concerned with one of the features
of re-frisk: it's popout window.

# Re-Frisk Implementation
The re-frisk popout window implementation can be seen [here]
(https://github.com/flexsurfer/re-frisk/blob/master/src/re_frisk/devtool.cljs).
For reference, see an abridged version with the key details below:

```clojure
(defn open-window []
  (let [w (js/window.open "" "Debugger" "width=800,height=800,toolbar=no,menubar=no")
        d (.-document w)]
    (.open d)
    (.write d html-doc)
    (goog.object/set w "onload" #(mount w d))
    (.close d)))
```

This snippet opens a new browser window that is 800px by 800px with the title
"Debugger" with no address or toolbars. Think of it simply as modal that you can
move around. `html-doc` is a string representation of a simple HTML document and
`mount` is a function that mounts a reagent component onto the DOM of the newly
opened window. Now, keep in mind that the CLJS context of the newly opened
window is that same as that of the parent window so you can reference the
re-frame DB of the parent window with typical events and subscriptions.

# My Implementation
Now, below is my implementation that includes some logic to append the
stylesheets of the parent to the newly opened window along with the mount
function and a dummy data subscription.

```clojure
(defn dummy-component []
  (let [my-data @(rf/subscribe [:dummy-data])]
    [:div [:pre (with-out-str (pprint my-data))]]))

(defn mount [doc]
  (let [app (.getElementById doc "win-app")]
    (r/render [dummy-component] app)))

(defn html-doc [css-urls]
  (let [links (map #(str "<link rel=stylesheet href=\"" % "\">") css-urls)]
    (str "<!doctype html>"
      "<head>"
      (apply str links)
      "</head>"
      "<body><div id=\"win-app\"></div></body>")))

(defn open-window []
  (let [w (js/window.open "" "Debugger" "width=800,height=800,toolbar=no,menubar=no")
        d (.-document w)
        ;; Beware this will break if you have non-stylesheet links in your
        ;; parent page
        css-urls (-> (js/document.getElementsByTagName "link")
                     array-seq
                     (->> (map #(.-href %))))]
    (.open d)
    (.write d (html-doc css-urls))
    (goog.object/set w "onload" #(mount w d))
    (.close d)))
```

Now, one minor caveat to make note of: the root component that is mounted in
the newly opened window will not re-render based on props (argument) changes.
In order to get the windowed component to re-render as you expect, it will need
to subscribe to data independently.
