  What I Learned: Locality of Shadowing

  Key principle: Shadowing should be as local as possible and never leak
  beyond where it's explicitly needed.

  1. Keep Shadowing Contained to Its Namespace

  When you shadow a core function (like in example.rng):
  - :white_check_mark: It's fine to use rand-int inside example.rng namespace
  - :white_check_mark: The :refer-clojure :exclude [rand-int] makes the shadowing explicit
  - :x: Don't let it leak to other namespaces via :refer

  2. Use Qualified Names Outside the Shadowing Namespace

  In other namespaces (like example.systems.production):

  ;; :white_check_mark: GOOD - Makes it explicit this is rng's version
  (ns example.systems.production
    (:require [example.rng :as rng]))

  (rng/rand-int rng-state 100)  ; Clear: this is the deterministic version
  (rng/with-rng world :production ...) ; Clear: using rng module

  ;; :x: BAD - Brings shadowing into this namespace
  (ns example.systems.production
    (:require [example.rng :refer [rand-int]]))

  (rand-int rng-state 100)  ; Confusing: which rand-int?

  Why: When someone reads rng/rand-int, they know it's not core's random
  version. When they see bare rand-int, they expect core's behavior.

  3. For Temporary Overrides, Use Scoped Tools

  If you need to temporarily change behavior (like in tests):

  ;; :white_check_mark: GOOD - Change is explicitly scoped
  (with-redefs [rand-int (fn [n] 42)]
    (test-something))  ; Override only applies here

  ;; :x: BAD - Global change
  (def rand-int (fn [n] 42))  ; Now EVERYTHING uses this

  Why: with-redefs makes it clear the override is local to that block. If
  this becomes a library, the redefinition won't leak to callers.

  4. The Goal: No Surprises at Any Distance

  - Inside example.rng: rand-int is our deterministic version (documented)
  - In example.systems.production: rng/rand-int makes it clear it's not
  core
  - In user code: If they import as [example.rng :as rng], they still see
  rng/rand-int
  - In tests: with-redefs makes temporary overrides explicit

  5. Library-Safe Design

  If example becomes a library:
  - Users who require our namespaces won't get surprised by shadowed
  functions
  - They'll see rng/rand-int and know it's not core
  - Our shadowing stays contained to where we defined it

  ---
  Summary in one sentence:
  Shadow locally, qualify when crossing namespace boundaries, use scoped
  tools for temporary changes, so that redefinitions never surprise anyone
  at any level of the call stack.
