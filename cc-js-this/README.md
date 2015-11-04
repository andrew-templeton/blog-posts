Today's JavaScript Crash Course covers one of the most ill-understood features of JavaScript - `this` and context binding. We will list all of the rules for `this` at once, then deep-dive into each rule in its own section. After this crash course, you will be equipped with the tools to: 

* Understand the behavior of `this` in code you read. 
* Prevent `this` contextual-binding bugs and ambiguities.
* Understand the nuances of `this` in ECMA5 `'use strict';` mode.

---

###### Why does `this` hurt us so?!

For many intrepid JavaScript engineers coming from other languages, `this` is the source of headaches and lost hours of debugging, because `this` defies all logic relative to, say, Java.

> "A function's `this` keyword behaves a little differently in JavaScript compared to other languages." -- [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

To more seasoned readers, the above quote might cause a chuckle. Indeed, JavaScript uses `this` in a significantly different manner from most languages. Fortunately, there *is* a method to the madness. We need to learn 8 base rules and their implications to fully grasp and understand `this`. Brace yourself, the list may be intimidating - we will give each item a thorough treatment.

---

## The 8 Rules of `this`

[1.](#rule1) When referenced in global scope, `this` is the global object.  
[2.](#rule2) In functions called with dot or bracket notation, the immediate object that the function is a member of is `this`.  
[3.](#rule3) In a standalone-called function, `this` is either the global object or `undefined` when in `'use strict;` mode.  
[4.](#rule4) Inside invocations with `new`, `this` is the `new` object being created.  
[5.](#rule5) In a function invoked with `call` or `apply`, `this` is the first argument of `call`/`apply`.  
[6.](#rule6) In a function created with `bind`, the bound `this` argument *always* takes precedence.  
[7.](#rule7) For event handlers inlined in HTML, `this` is the element with the inlined handler.  
[8.](#rule8) For other event handlers, `this` is the element that fired the event.

(click/tap a numbered bullet above to jump to its respective section)

---


#### Rule 1: In Global Scope

> When referenced in global scope, `this` is the global object.

In JavaScript, code executing outside of any/all functions is within the global scope. For browsers, the global object is `window`, while in Node.js and most other environments, it is `GLOBAL`. Thus, when outside of all functions, `this === window` for browsers, and `this === GLOBAL` in Node.js.

###### When does the rule apply?

Why, global versus non-global scope, of course. What is global scope?
    
    // Global scope, outside any functions.
    // This rule, Rule 1 applies
    var foo = 'bar';

    (function() {
      // NOT global scope, within a function.
      // Does NOT apply here (Rule 3 does)
      var foo = 'bar';
    })();
    
    function namedFunc() {
      // NOT global scope, within a function.
      // Does NOT apply here (Rule 3 does)
      var foo = 'bar'
    }


###### See how it works

First, to make the example environment agnostic, 

    // Storing either GLOBAL or window...:
    var STORED_GLOBAL = typeof window !== 'undefined'
      ? window
      : GLOBAL;
    
Now that we have an environment-agnostic `STORED_GLOBAL` that represents whatever global object is available in the environment:
    
    // Global scope, outside any functions.
    // Rule (1) applies here.
    this === STORED_GLOBAL; // true
    
    // Setting up a stored reference to global this
    var thisAlias = this;
  
    // Setting up some non-global scopes
    var foo = {
      bar: function() {
          // Checking within this non-global scope
            return this === STORED_GLOBAL;
        },
        baz: function() {
          // Using the stored value
          return thisAlias === STORED_GLOBAL;
        }
    };
    // Running the code above..
    foo.bar(); // false
    foo.baz(); // true
    
In the `foo.bar` function, we are no longer in a global scope, but rather inside a function member of the `foo` object, so the test returns `false` this time.   
Note that the `foo.baz` function does *not* use `this` in its body, but uses the stored alias. In this case, the value for `testAlias`, which was set in the global scope, is still bound, so the test is `true`.


#### Rule 2: Dot or Bracket Calls

> In functions called with dot or bracket notation, the immediate object that the function is a member of is `this`. 

When functions are called with dot or bracket notation, like `foo.bar()`, we say that `bar` is a member function of `foo`. Thus, `foo` is the "immediate object that [`bar`] is a member of", and `foo` would be `this` within the function body of `bar`.

The "immediate"-ness of this relationship is important, as it helps us make sense of `this` within longer dot or bracketed calls. For instance, in the call `parent.object.func()`, within the body of `func`, `this` will be `object`, *not* `parent`. This is because `func` is an immediate member of `object`, not `parent` - even though `func` indirectly belongs to `parent`.

####### When does this rule apply?

Dot or bracket notation, where the rule applies, looks like any of the following example calls:

    // While I omit arguments in the calls for brevity,
    //   arguments may be supplied in any of the calls.
    
    // A function to add to our object.
    function borrowedFunc(<params...>) { <code...> }
    // The object we use:
    var someObject = {
      someFunc: function(<params...>) { <code...> },
        // Note that all rules apply this function too,
        //   since it is now an immediate member of someObject.
        anotherFunc: borrowedFunc
    };
    
    // In all of the following cases, this === someObject.
    
    // Dot notation
    someObject.someFunc();
     
    // Bracket notation, with a string literal:
    someObject['someFunc']();
     
    // Bracket notation, with a variable string:
    var someKey = 'someFunc';
    someObject[someKey]();
    
    // Bracket notation, more complex & spacing:
    someObject[(Math.random() > 0.5
      ? 'some'
      : 'another') +
    'Func']();
    
###### See how it works

First, let's set an object up to test against, with subobjects, and a couple functions:

    function adopted() {
      console.log(this === parent);
      console.log(this === child);
    }
    var child = {
      innerInline: function() {
        console.log(this === parent);
        console.log(this === child);
      },
      adopted: adopted
    };
    var parent = {
      child: child, 
      outerInline: function() {
        console.log(this === parent);
        console.log(this === child);
      },
      adopted: adopted
    };

Whew - we now have a nested object, and three functions anchored at four points within the larget `parent` object. All three functions check against our two `Object` literals (`parent`/ `child`), and `child` is a member of `parent`.

Let's see what is logged by our `console.log` calls when invoked. Remember, all of the invocation styles (dot, bracketed...) above produce equivalent results - so we only need use the succinct dot style to explore all behaviors.

First, the most straightforward calls, with the `innerInline` and `outerInline` functions we declared inside the object definition:
  
    child.innerInline();
    // this === parent => false
    // this === child => true
    parent.child.innerInline();
    // this === parent => false
    // this === child => true
    // innerInline is an immediate member of child only.
    
    parent.outerInline();
    // this === parent => true
    // this === child => false
    // outerInline belongs to parent only.
    
Great - we can clearly see the immediate member relationships, because we declared them as such, and called them how one would expect. But what happens when we store the `innerInline` and `outerInline` elsewhere, *then* call them?

    var orphanedInner = child.innerInline;
    var orphanedOuter = parent.outerInline;

    orphanedInner === child.innerInline; // true, okay...
    orphanedInner();
    // this === parent => false, same as before.
    // this === child => false... huh? No more?
    // orphanedInner is *not* a member of child anymore!
    // We called it without dot/bracket notation,
    //     or "out of context"/"without context".

    // Similarly,
    orphanedOuter === parent.outerInline; // true, yep.
    orphanedOuter();
    // this === parent => false, *no context*
    // this === child => false, no change.
    
Yikes - but, if we remember that Rule 2 only applies when called with the dot/bracket notation, this all makes sense. Both of the `orphaned` functions were called "without context", or without said notation. How about the opposite? Recall that we added the same `adopted` function, which was originally created without context in the global scope, directly to both the `parent` and `child` objects.

  


  


    