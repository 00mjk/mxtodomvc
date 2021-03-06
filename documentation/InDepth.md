# TodoMVC, with Matrix Inside&trade;: the Mechanics
*Matrix dataflow and mxWeb, in depth*

## Preface
This in-depth look at what happens under the hood of mxWeb/Matrix will make more sense if you have already read [the preamble](../README.md) and followed [our implementation](BuildingTodoMVC.md). 

*Warning: Much of what follows duplicates material offered so far, to provide a coherent narrative thread. To find just the new in-depth stuff, look for "Matrix Under the Hood" headers.*

## Introduction
The Matrix dataflow library converts the reads and writes of declarative formulas and handlers and somewhow produces a complete application. In our final episode we will lift the hood on Matrix just far enough to see how a simple write propagates out to interested formulas and finally shapes a Web page. 

We choose *mxWeb* as the vehicle for introducing Matrix because nothing challenges a developer more than managing application state while an intelligent user does their best to use an interface correctly. Then marketing overhauls the U/X.

> "UIs...I am not going to go there. I don't do that part."  
-- Rich Hickey on the high ratio of code to logic in UIs, *Clojure/Conj 2017*

mxWeb is a thin web un-framework built atop Matrix. We say "un-framework" because mxWeb exists only to wire the DOM for dataflow. The API design imperative is that [MDN](https://developer.mozilla.org/en-US/) be the mxWeb reference; mxWeb itself introduces no new architecture.

Matrix achieves this simply by enhancing how we initialize, read, and write individual properties:
* properties can be initialized as a literal value or as a function;
* should some property `A` be initialized with a literal, we can write to it;
* should a functional property `B` read `A`, `A` remembers `B`;
* when we write to `A`, `A` tells `B`; and
* we can supply "on change" callbacks for properties.

## Set-Up
If you have not already executed these steps during the previous walkthrough, let's get you caught up:

````bash
git clone https://github.com/kennytilton/mxtodomvc.git
cd mxtodomvc
lein deps
lein clean
lein fig:build
````
This will auto compile and send all changes to the browser without the need to reload. After the compilation process is complete, you will get a Browser Connected REPL. A web page should appear on a browser near you with a header saying just "hello, Matrix". 

For issues, questions, or comments, ping us at kentilton on gmail, or DM @hiskennyness on Slack in the #Clojurians channel.

### enter-todos
We jump to a point early in our implementation of TodoMVC, when we first display a list of fixed to-dos. This tag adds a bunch more UI structure but no ability to edit or create todos:
* we load a few fixed-todos at start-up;
* we show them in a list;
* one control lets us toggle a to-do between completed or not; and
* another control logically deletes a to-do.
````bash
# Control-D
git checkout enter-todos
lein fig:build
````
<img height="384px" align="center" src="pix/enter-todos.png?raw=true">

Here is the code to make a to-do *model*. `cI` is shorthand for "input cell", one to which we can imperatively assign values in an event handler.
````clojure
(defn make-todo
  "Make a matrix incarnation of a todo item"
  [title]

  (md/make
    :id (util/uuidv4)
    :created (util/now)
    :title (cI title)
    :completed (cI nil)
    :deleted (cI nil)))
````
Apparent booleans like `:completed` will in fact be nil or timestamps, so no "?" suffixes.

Here is the to-do-list container model, set up to take a list of hard-coded initial to-dos to get us rolling:
````clojure
(defn todo-list [seed-todos]
  (md/make ::todo-list
    :items-raw (cFn (for [to-do seed-todos]
                      (make-todo to-do})))
    :items (cF (doall (remove td-deleted (<mget me :items-raw))))))
````
The bulk of the to-do app does not care about deleted to-dos, so we use a clumsy name "items-raw" for the true list of items ever created, and save the name "items" for the ones actually used. `cFn` is short for "formulaic then input", meaning the property is initialized by running the formula and thereafter is set by imperative code.

We can now start our demo matrix off with a few preset to-dos. 

````clojure
(md/make ::md/todoApp
      :todos (todo/todo-list ["Wash car" "Walk dog" "Do laundry" "Mow lawn"])
      :mx-dom (cFonce
                (with-par me
                  (section {:class "todoapp" :style "padding:24px"}
                    (webco/wall-clock :date 60000 0 15)
                    (webco/wall-clock :time 1000 0 8)
                    (header {:class "header"}
                      (h1 "todos"))
                    (todo-items-list)
                    (todo-items-dashboard)
                    (webco/app-credits mxtodo-credits)))))
````
#### Matrix Under the Hood
* the optional first "type" parameter ::todoApp is supplied; we will use that in various Matrix searches to identify the root of the TodoMVC matrix.
* building the matrix DOM is now wrapped in `(cFonce (with-par me ...)`;
  * `cFonce` effectively defers the enclosed form until the right lifecycle point in the matrix's initial construction.
  * `with-par me` is how the matrix DOM knows where it is in the matrix tree. All matrix nodes know their parents so they can navigate the tree freely to gather information.
* the app credits are now provided by a new "web component", and that along with the "wall-clock" reusable are off in their own namespace.

And now the to-do item view itself.
````clojure
(defn todo-list-item [todo]
  (li
    {:class (cF (when (td-completed todo)
                  "completed"))}
    {:todo todo}
    (div {:class "view"}
      (input {:class       "toggle"
              ::mxweb/type "checkbox"
              :checked     (cF (not (nil? (td-completed todo))))
              :onclick     #(td-toggle-completed! todo)})

      (label (td-title todo))

      (button {:class   "destroy"
               :onclick #(td-delete! todo)}))))
````
Elsewhere we find the "change" dataflow initiators:
````clojure
(defn td-delete! [td]
  (mset!> td :deleted (util/now)))

(defn td-toggle-completed! [td]
  (mswap!> td :completed
    #(when-not % (util/now))))
````
We execute a simple dashboard as well, without the filters just yet:
````clojure
(defn todo-items-dashboard []
  (footer {:class  "footer"
           :hidden (cF (<mget (mx-todos me) :empty?))}
    (span {:class   "todo-count"
           :content (cF (pp/cl-format nil "<strong>~a</strong>  item~:P remaining"
                          (count (remove td-completed (mx-todo-items me)))))})
    (button {:class   "clear-completed"
             :hidden  (cF (empty? (<mget (mx-todos me) :items-completed)))
             :onclick #(doseq [td (filter td-completed (mx-todo-items))]
                         (td-delete! td))}
      "Clear completed")))
````
Both `hidden` properties address spec requirements, and are implemented precisely by just adding or removing the DOM hidden attribute. The `content` is adjusted with no more than setting `innerHTML`.

In support of the above we extend the model of the to-do list with more dataflow properties:
````clojure
(defn todo-list [seed-todos]
  (md/make ::todo-list
    :items-raw (cFn (for [td seed-todos]
                      (make-todo td)))
    :items (cF (doall (remove td-deleted (<mget me :items-raw))))
    :items-completed (cF (doall (filter td-completed (<mget me :items))))
    :empty? (cF (empty? (<mget me :items)))))
````
Other things the reader might notice:
* `mx-todos` and `mx-todo-items` wrap the complexity of navigating the Matrix to find desired data;
* `doall` in various formulas may soon be baked in to Matrix, because lazy evaluation breaks dependency tracking.  
Recall that Matrix works by changing what happens when we read properties. The internal mechanism is to bind a formula to `*depender*` when kicking off its rule. With lazy evaluation, that binding is gone by the time the read occurs.

#### Matrix Under the Hood
We can now play with toggling the completion state of to-dos, deleting them directly, or deleting them with the "clear completed" button, keeping an eye on "items remaining". Consider specifically what happens when we click the completion toggle of the view displaying an uncompleted todo. This code executes:
````clojure
(mset!> td :completed (now))
````
What then does the Matrix engine do for us?
* the completed property of the associated model gets set to the current JS epoch (duh);
* the class of the todo list item view gets recomputed because it read the :completed property. It changes to "completed";
* the LI DOM element classList gets set to "completed" by an mxWeb observer;
* the :content property formula of the "Items remaining" span recounts the list of todos filtering out the completed and comes up with one less;
* an mxWeb observer updates the span innerHTML to the new "remaining" count;
* the :items-completed property on the todos list model container gets recalculated because it reads the :completed property of *all* todo item models. It grows by one;
* the :hidden property of the "Clear completed" button/map gets recalculated because it reads the :items-completed property of the list of todos. If the length changes either way between zero and one, the :hidden property becomes true or false...
* an mxWeb observer adds or removes the hidden attribute of the "Clear completed" DOM button.

We will spare the reader the detailed analysis of what happens when we click the "delete" button (the red "X" that appears on hover), but they may want to work out for themselves the dataflow from the :deleted property to these behaviors:
* the item disappears;
* if the item was incomplete when deleted, the "Items remaining" drops by one;
* if the item was the only completed item, "Clear completed" disappears;
* if the item was the last of any kind, the dashboard disappears.

### lifting-xhr
Next we re-visit an especially interesting example of lifting: XHR, affectionately known as Callback Hell. We do so exceeding the official TodoMVC spec to alert our user of any to-do item that returns results from a search of the FDA [Adverse Events database](https://open.fda.gov/data/faers/).

If you enter a new to-do, it will appear with a gray alert icon, gray signifying undecided. If no adverse events are found, the alert disappears. If any are found, it turns red. (You will also observe excessive such lookups, to be addressed next.)
````bash
# Control-D
git checkout lifting-xhr
lein fig:build
````
Our treatment to date of [the XHR lift](https://github.com/kennytilton/matrix/tree/master/cljs/mxxhr) is technically minimal but the test suite includes clean dataflow solutions to several Hellish use cases. Our use case here is trivial, just a simple XHR query to the FDA API and one response bound to success or error information.
````clojure
(defn adverse-event-checker [todo]
  (i
    {:class   "aes material-icons"
     :title "Click to see some AE counts"
     :onclick #(js/alert "Feature to display AEs not yet implemented")
     :style   (cF (str "font-size:36px"
                    ";display:" (case (<mget me :aes?)
                                  :no "none"
                                  "block")
                    ";color:" (case (<mget me :aes?)
                                :undecided "gray"
                                :yes "red"
                                ;; should not get here
                                "white")))}

    {:lookup   (cF+ [:obs (fn-obs (xhr-scavenge old))]
                 (make-xhr (pp/cl-format nil ae-by-brand
                             (js/encodeURIComponent
                               (de-whitespace (td-title todo))))
                   {:name       name
                    :send? true
                    :fake-delay (+ 500 (rand-int 2000))}))
     :response (cF (when-let [xhr (<mget me :lookup)]
                     (xhr-response xhr)))
     :aes?     (cF (if-let [r (<mget me :response)]
                     (if (= 200 (:status r)) :yes :no)
                     :undecided))}
    "warning"))
````
    
That is the application code, a powerful example of the SSB *single source* principle. Add or remove that block of code to completely swap in/out all concerns. 

To see where the response dataflow starts we must look at mxXHR libary internals. (Look for the `mset!>`; as for `with-cc`, we touch on that below):
````clojure
(defn xhr-send [xhr]
  (go
    (let [response (<! (client/get (<mget xhr :uri) {:with-credentials? false}))]
       (with-cc :xhr-handler-sets-responded
          (mset!> xhr :response    ;; <------- DATAFLOW BEGINS HERE
            {:status (:status response)
             :body   (if (:success response)
                       ((:body-parser @xhr) (:body response))
                       [(:error-code response)
                        (:error-text response)])}))))))
````
Hellish async XHR responses are now just ordinary Matrix inputs. 
> If you play with new to-dos, do *not* be alarmed by red warnings: all drugs have adverse events, and the FDA search is aggressive: cats have adverse events. Dogs are fine.
Hellish async XHR responses are now just ordinary Matrix inputs. 

#### Matrix Under the Hood
One formula generates an XHR, another use the response, success or failure. Some notes on what happens in between:
* we fake variable response latency;
* `with-cc`, an advanced trick, enqueues its body for execution at the right time in the datafow lifecycle.
* `lookup` functionally returns an mxXHR incarnation of an actual XHR, but...
* ...we specify that the XHR be *sent* immediately! This is where `with-cc` comes in...
* ...getting into the weeds, `with-cc` enqueues its body for execution at the right time in the datafow lifecycle...
* ...and should the user change the title and kick off a new lookup, an observer will GC the old one;
* `response` runs immediately, reads the nil lookup `response` but establishing the dependency, and returns nil;
* the `aes?` predicate runs immediately and does not see a `response`, so it returns `:undecided`;
* the color and display style properties decide on "gray" and "block";
* an mxWeb observer updates the DOM so we see a gray "warning" icon;
* when the actual XHR gets a response, good or bad, it is `<mset!` *with dataflow integrity* into the `response` property of the mxXHR;
* our AE checker `response` formula runs and captures the response;
* `aes?` runs, sees the response, and decides on :yes :or :no;
* the color and display style properties decide on new values;
* mxWeb does its thing and the warning disappears or turns red.

## Summary
In this write-up we have detailed all the things that must happen when users make simple changes, or when a Web page needs remote data and emits an XHR. Without dataflow, the programmer must arrange all that. But if one reviews the mxTodoMVC implementation, that complexity is nowhere to be found. 

Where did it go?

With ReactJS, Facebook engineers popularized the power of declarative, functional programming. In mxWeb we see that power extended beyond the view to the model and to any other system for which we care to write sufficient glue, such as XHR.

User interfaces strike Mr. Hickey as messy beasts. The way we normally code them, they are. But after extending the functional view decomposition Facebook introduced across the application, we discover the simple logic of user interfaces.  






