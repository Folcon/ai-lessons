---
name: clojure-repl
description: Use when working with Clojure code, connecting to REPLs, evaluating expressions, debugging, or doing interactive development
---

# Clojure REPL Mastery

## Philosophy

The REPL is a **user interface to your running program**. Core principles:

1. **Come with a plan** - Know what you're testing; aimless exploration leads to distraction
2. **Hypothesis → Test → Refine** - Evaluate small expressions, build understanding incrementally
3. **Preserve your work** - The REPL is ephemeral; capture discoveries in code/tests/comments
4. **Know its limits** - Rapid feedback isn't a substitute for design; step back when stuck

## Connecting

```bash
lein repl :connect              # Connect to existing nREPL
lein repl                       # Start new REPL

# Batch evaluation via heredoc (quote delimiter to prevent shell expansion)
cat << 'EOF' | lein repl :connect
(require '[clojure.string :as str])
(str/join ", " [1 2 3])
EOF
```

## Special Vars (Critical!)

| Var | Purpose |
|-----|---------|
| `*1` | Last result |
| `*2` | Second-to-last result |
| `*3` | Third-to-last result |
| `*e` | Last exception |
| `*ns*` | Current namespace |

```clojure
(+ 1 2 3)         ;; => 6
*1                ;; => 6
(* *1 10)         ;; => 60
(def saved *1)    ;; Capture before it's gone!
```

## Help & Discovery

```clojure
(doc fn-name)              ;; View docstring
(source fn-name)           ;; View source code
(dir clojure.string)       ;; List all vars in namespace
(apropos "join")           ;; Search var names
(find-doc "concatenate")   ;; Search docstrings
```

## Debugging Patterns

### Non-invasive Inspection
```clojure
;; doto prints and returns - insert anywhere without restructuring
(doto (complex-calculation x) prn)

;; Inline def to capture intermediate values (remove when done!)
(defn process [data]
  (let [step1 (transform data)
        _ (def debug-step1 step1)  ;; Capture for REPL inspection
        step2 (filter valid? step1)]
    step2))
```

### Context Recreation
```clojure
;; Reproduce function conditions by defining vars matching locals
(def m 80)
(def r 5e6)
;; Now evaluate function body expressions directly
(* G (/ m (* r r)))
```

### Error Investigation
```clojure
*e                          ;; Last exception object
(clojure.repl/pst)          ;; Print stack trace (last exception)
(clojure.repl/pst *e 30)    ;; With depth limit
(.getMessage *e)            ;; Just the message
(ex-data *e)                ;; Data from ex-info exceptions
```

### Tap-Based Debug Logging

Use `tap>` with a global atom to capture values during debugging. This is especially useful for LLM-assisted debugging - structured data in the REPL beats parsing console output.

**Setup (evaluate once per session):**
```clojure
(defonce debug-log (atom []))
(defonce debug-log-max 100)

(defn- debug-entry [v]
  (let [ts #?(:clj (System/currentTimeMillis) :cljs (.getTime (js/Date.)))
        [label value] (if (and (vector? v) (= 2 (count v)) (keyword? (first v)))
                        v
                        [nil v])]
    {:ts ts :label label :value value}))

(defonce _tap-handler
  (add-tap (fn [v]
             (swap! debug-log
                    (fn [log]
                      (let [log' (conj log (debug-entry v))]
                        (if (> (count log') debug-log-max)
                          (vec (drop (- (count log') debug-log-max) log'))
                          log')))))))
```

**Query helpers:**
```clojure
(defn logs
  "Query debug log: (logs), (logs 5), (logs :label), (logs :label 3)"
  ([] @debug-log)
  ([n-or-label]
   (if (number? n-or-label)
     (vec (take-last n-or-label @debug-log))
     (vec (filter #(= n-or-label (:label %)) @debug-log))))
  ([label n]
   (vec (take-last n (filter #(= label (:label %)) @debug-log)))))

(defn log-values
  "Like logs, but returns just the values"
  ([] (mapv :value @debug-log))
  ([n-or-label] (mapv :value (logs n-or-label)))
  ([label n] (mapv :value (logs label n))))

(defn clear-logs! [] (reset! debug-log []))

(defn last-log
  "Most recent entry, or just value with (last-log :v)"
  ([] (last @debug-log))
  ([_] (:value (last @debug-log))))
```

**Usage:**
```clojure
;; Labeled logging - vector with keyword first
(tap> [:fetch response])
(tap> [:error {:fn 'process :ex (ex-message e)}])

;; Unlabeled - any other value
(tap> intermediate-result)

;; Query in REPL
(logs 5)                  ;; Last 5 entries
(logs :fetch)             ;; All :fetch entries
(log-values :error 3)     ;; Last 3 error values only
(last-log :v)             ;; Most recent value

;; In threading macros
(->> data
     (map transform)
     (doto #(tap> [:after-map %]))
     (filter valid?))
```

**Why this beats devtools:** Structured data stays in the REPL where you can `def`, transform, and test against captured values directly.

## Data Visualization

### Pretty Printing
```clojure
(require '[clojure.pprint :as pp])
(pp/pprint big-nested-map)
(pp/pp)                     ;; Pretty-print *1 (shortcut!)
(pp/print-table [:a :b] [{:a 1 :b 2} {:a 3 :b 4}])
```

### Controlling Output Size
```clojure
(set! *print-level* 3)      ;; Truncate nesting (shows # for hidden)
(set! *print-length* 10)    ;; Limit collection items
(set! *print-level* nil)    ;; Reset to unlimited
```

Example:
```clojure
(set! *print-level* 2)
{:a {:b {:c {:d "deep"}}}}
;; => {:a {:b #}}           ;; Deeper levels hidden
```

## Namespace Management

```clojure
(require '[myproject.core :as core])       ;; Load with alias
(require '[myproject.core] :reload)        ;; Reload after file changes
(require '[myproject.core] :reload-all)    ;; Reload with dependencies
(in-ns 'myproject.core)                    ;; Switch to namespace
(ns-publics 'myproject.core)               ;; List public vars
```

**Warning**: Always `require` a namespace before `in-ns` - switching to unloaded namespace causes confusing errors.

## Hot Reloading Pattern

For code that should update when redefined:
```clojure
;; Reference the Var, not its value
(defn handler [request]
  (#'actual-handler request))  ;; Var indirection via #'

;; Now redefining actual-handler takes effect immediately
```

## Collection Gotchas

```clojure
;; conj adds to efficient end
(conj '(1 2 3) 0)    ;; => (0 1 2 3) - front for lists
(conj [1 2 3] 4)     ;; => [1 2 3 4] - end for vectors

;; pop vs rest on empty
(rest '())           ;; => () - returns empty
(pop '())            ;; => Exception!

;; Type differences
(= \c "c")           ;; => false (char vs string)
(= 2 2.0)            ;; => false (use == for numeric equality)
(== 2 2.0)           ;; => true

;; Records vs Types
(:key (MyRecord. 1)) ;; => works (records are maps)
(:key (MyType. 1))   ;; => nil (types are not maps)
```

## Threading Macros

```clojure
;; -> thread-first: insert as FIRST arg (maps, strings, objects)
(-> {}
    (assoc :a 1)        ;; (assoc {} :a 1)
    (update :a inc))    ;; (update {...} :a inc)

;; ->> thread-last: insert as LAST arg (sequences)
(->> (range 10)
     (filter odd?)      ;; (filter odd? (range 10))
     (map inc)          ;; (map inc (...))
     (reduce +))        ;; (reduce + (...))
```

## REPL State Management

The REPL accumulates state. When things get confusing:

```clojure
;; Check what's defined
(ns-publics *ns*)

;; Remove a def
(ns-unmap *ns* 'problematic-var)

;; Nuclear option: restart REPL or
(require '[myns] :reload-all)
```

## Workflow Summary

```
1. Form hypothesis about code behavior
2. Write minimal expression to test it
3. Evaluate and observe result
4. Use *1/*2/*3 to build on results
5. (def name *1) to preserve important values
6. When errors occur: *e, pst, ex-data
7. Move working code to source files
8. Repeat
```
