---
layout: post
title:  "Task Bundles"
date: 2016-06-13 00:00:00
categories: jekyll update
author: Gardner Vickers
---

Onyx's strength has always been it's data-driven nature thanks to
its data based API. This API can be thought of as a low-level language for your
distributed computation.

A data structure based API forces you to understand and decide all-the-things
upfront. A common approach to conquer this problem is to include a sort of
meta-language in the data structure. In all but the simplest cases, this leads
to recreating a new programming language in your data structures.
It's much better to use what we already have.

We have wonderful tools for manipulating data structures, they are
built into Clojure! We routinely use these tools to build abstractions for
ourselves. They allow us to move up and down conceptual levels, all the way
from `(start-server ...)` down to `(.readLine (io/reader (ServerSocket. 80)))`

In order for the abstractions we build to be useful for others, or useful in the
in the context of a larger system, they must share a common contract for
composition. By doing this, the overall abstraction does not matter as what it
produces can just snap into place.

In other words, **Legos for Onyx**.

Over the last few months, we've been refining our set of abstractions over Onyx's
data API. The pattern is referred to as "task bundles", and the core of it exists
in the `onyx.job` namespace.

### Task Bundle Map
Task bundles operate on "task bundle maps", a shared contract.
A task bundle map is just a plain clojure map of the shape

```clojure

{:task {:task-map {:onyx/name :inc
                   :onyx/type :function
                   :onyx/fn ::inc-viewcount
                   :onyx/batch-size 1
                   :myproject/special-value 100
                   :onyx/batch-timeout 1000}
        :lifecycles [{...}]
        :flow-conditions [{...}]
        :windows [{...}]
        :triggers [{...}]}

 :schema {:task-map {:myproject/special-value s/Num}
          :lifecycles {:myproject/special-state s/Str}
          :windows {...}
          :triggers {...}}}
```


A "task bundle map" made of two parts. The `:task` key represents Onyx
declarations. The `:schema` key represents constraints on those declarations.
`add-task` will check the constraints against the declarations.

The overarching idea with task bundles, is to take everything required for a
task type, from the task map, to the flow conditions, or windows, and bundle
them together in a way that promotes re-use of task types that you may build.

### Tasks

We package up "task bundles" into "tasks", which are just functions that build the data in the required form.
We do this so that we have an easy entry-point to change the "task bundle map".

See below where we allow changing the key sequence (`ks`) that the `inc-in-segment`
function operates on.

```clojure

(s/defn inc-key
  ([task-name :- s/Keyword task-opts]
   {:task {:task-map (merge {:onyx/name task-name
                             :onyx/type :function
                             :onyx/fn ::inc-in-segment
                             :onyx/params [::inc-key]}
                            task-opts)}
    :schema {:task-map IncKeyTask}})
  ([task-name :- s/Keyword
    ks :- [s/Keyword]
    task-opts]
   (inc-key task-name (merge {::inc-key ks} task-opts))))
```

The two-arity form allows users of the task to see what they *need* to provide
to have a valid task bundle. `::inc-in-segment` will not work unless we specify
a key sequence to the number we want to increment. To signal this, we both
provide constraints on it and include it in the second arity.

### Task Bundle Modifiers
`onyx.job/add-task` provides a second variadic arity, taking a function of the
form:

```clojure
(fn [task-bundle] ...) => {task-bundle-map}
```
This allows us to bundle meta behavior  and apply it
to task bundles before they are schema checked and merged. This is useful for
things like adding logging, triggers, or further restricting schema.
An example of adding a trigger to send window state to MySQL is below.

```clojure

(s/defn with-trigger-to-sql
  [window-id :- s/Keyword connection-uri :- s/Str]
  (fn [task-definition]
    (-> task-definition
        (update-in [:task :triggers] conj
                   {:trigger/window-id window-id
                    :trigger/refinement :onyx.refinements/accumulating
                    :trigger/on :onyx.triggers/segment
                    :trigger/threshold [5 :elements]
                    :trigger/sync :twit.persist.sql/upsert-trending
                    :sql/connection-uri {:connection-uri connection-uri}})
        (update-in [:schema :triggers] conj
                   {:sql/connection-uri {:connection-uri s/Str}}))))
```

`with-trigger-to-sql` add's new Onyx declarations to the `:task` key (triggers),
and new constraints. `add-task` will check these new constraints just the same.
Convention is to name task bundle modifier functions `with-*`.

## Jobs
"tasks" and "task bundle modifiers" are all about encapsulating functionality to
assist with reuse. Right now, most of the Onyx plugins provide a task bundle
interface under `onyx.tasks.<plugin-name>`. This allows us to avoid dealing
directly with the Onyx job map, making our jobs [look like this](https://github.com/onyx-twitter-sample/twit/blob/master/src/twit/jobs/trending.clj).

```clojure
(-> {:workflow [[:in :reshape-segment]
                [:reshape-segment :split-hashtags]
                [:split-hashtags :out]]
                  :catalog []
                  :lifecycles []
                  :windows []
                  :triggers []
                  :flow-conditions []
                  :task-scheduler :onyx.task-scheduler/balanced}

    (add-task (twitter/stream :in [:id :text :createdAt] (merge batch-settings twitter-config)))
    (add-task (reshape/reshape-segment :reshape-segment batch-settings))
    (add-task (tweet/emit-hashtag-ids :split-hashtags [:id] [:text] batch-settings))
    (add-task (tweet/window-trending-hashtags :out :hashtag-window))
    (add-task (core-async-task/output :out (merge batch-settings {:onyx/group-by-key :hashtag
                                                                  :onyx/flux-policy :recover
                                                                  :onyx/min-peers 1
                                                                  :onyx/max-peers 1
                                                                  :onyx/uniqueness-key :id})))
    (add-task (tweet/with-trigger-to-sql :hashtag-window connection-uri)))
```

The [Onyx Twitter Sample](https://github.com/onyx-platform/onyx-twitter-sample)
holds a testing namespace for each job showcasing the benefits of this DSL
approach. It also demonstrates other related concepts like job registration and
submission. We intend this to be a community showcase of Onyx functionality that
will grow over time. Feel free to add extra jobs demonstrating other Onyx
features, or suggestions for job ideas.
