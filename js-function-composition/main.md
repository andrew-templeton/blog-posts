This post will explore a few tricks we can use as JavaScript developers to make code more readable and reusable.

### What is composition?

One of the most powerful aspects of JavaScript is the ability to use functions as first-class citizens. Since functions can both accept and return other functions, we can "compose" functions to build complexity while retaining readability and reusability.

> In computer science, *function composition* ... is an act or mechanism to combine simple functions to build more complicated ones. - [Wikipedia](http://en.wikipedia.org/wiki/Function_composition_%28computer_science%29)

My use of Wikipedia as a source notwithstanding, this statement should sound familiar to those familiar with class-based programming. Functional composition purports to solve the problem of building complexity in the same manner that inheritance does in class-based programming (and more generally, all object oriented programming).

Composition, at face value, looks quite simple - put two or more functions together, and get back a more complex function which does the same thing that the input functions would have done when run in sequence.

### Composition in Action

First, let's take a look at a very simple case in which we compose two functions into one, to do some basic math. Let's lay out the base functions we will compose.

	function addTwo(x) {
    	return x + 2;
    }
    function timesFive(x) {
    	return x * 5;
    }
    
Simple enough arithmetic. What if we want to `addTwo`, then `timesFive`? We will pass the result of `addTwo` into `timesFive`. There are several ways to accomplish this manually. Here are a few ways we can define an `addTwoThenTimesFive` function:

	// Storing the intermediate results, twice:
    function addTwoThenTimesFive(x) {
    	var y = addTwo(x);
        var z = timesFive(y);
        return z;
    }
    
    // Store only one intermediate:
    function addTwoThenTimesFive(x) {
    	var y = addTwo(x);
        return timesFive(y);
    }
    
    // No intermediates, nested calls:
    function addTwoThenTimesFive(x) {
    	return timesFive(addTwo(x));
    }
    
While we have accomplished what we wanted to do, this is not particularly extensible, and we have to keep track of calling order within the function bodies. The code does not seem to lend itself to general reuse anywhere.


### Getting more abstract

Can we make the code look any more generic? How about we use `reduce` on the steps `addTwo` and `timesFive` themselves?
    
    // Using reduce on the simple function steps themselves:
    function addTwoThenTimesFive(x) {
    	var steps = [addTwo, timesFive];
    	return steps.reduce(function(intermediate, step) {
        	return step(intermediate);
        }, x);
    }
    
Well, this works too, but may seem like a head scratcher - the code looks *more* complex! If we look closely, we see that the function passed to `reduce` above is generic - there is no need for this to pertain to math. The code could be doing any number of things, and the only pieces specific to this problem are the function name `addTwoTimesFive` and `[addTwo, timesFive]`.

This tells us that we are getting closer to a general solution, because we have a piece of the code that is usable anywhere - we could just swap out `steps` to be any other functions and rename the function.

Let's pull the function passed to `reduce` out, so the code is clearer:

	function runStep(intermediate, step) {
    	return step(intermediate);
    }
	function addTwoThenTimesFive(initialValue) {
    	var steps = [addTwo, timesFive];
        return steps.reduce(runStep, initialValue);
    }
    
Now, the body of `addTwoThenTimesFive` conveys intent to the reader. It tells me I am going to `reduce` my steps down, using `runStep` in sequence on `initialValue`. 

### Composition & `pipeline`

While we able to convey intent to the user in the last example, we still have not solved the problem of reusability. I will still need to manually define an `Array` of `steps` within the new, composed function definition.

How can we keep conveying intent, but prevent the need for repeated `reduce` code everywhere? It turns out, anyone that has ever worked with Bash has an excellent model for this - piping. 

Let's borrow the notion of piping and build a higher-order function that builds `pipeline` functions for us - that is, functions that are a sequence of steps, as defined by smaller functions. First, we will revisit the last example above, using placeholders for the function name and value of `steps`:

	// Original
    function runStep(intermediate, step) {
    	return step(intermediate);
    }
	function addTwoThenTimesFive(initialValue) {
    	var steps = [addTwo, timesFive];
        return steps.reduce(runStep, initialValue)
    }
    
    // Using placeholders for use-case-specific pieces:
    function runStep(intermediate, step) {
    	return step(intermediate);
    }
    function COMPOSED_NAME(initialValue) {
    	var steps = ARRAY_OF_STEPS;
        return steps.reduce(runStep, initialValue);
    }
    
We can clearly see that the pattern will work on any use case, if only we had an easy way to define the `COMPOSED_NAME` and `ARRAY_OF_STEPS`.

We can do this by making a higher-order `pipeline` function that takes comma-delimited steps and returns the function equivalent to `COMPOSED_NAME`.

	function runStep(intermediate, step) {
    	return step(intermediate);
    }
    function pipeline() {
    	var steps = [].slice.call(arguments, 0);
        return function(initialValue) {
        	return steps.reduce(runStep, initialValue);
        };
    }
    // Using it:
    var addTwoThenTimesFive = pipeline(addTwo, timesFive);
    addTwoThenTimesFive(1); //=> 15
    addTwoThenTimesFive(-1); //=> 5
    addTwoThenTimesFive(100); //=> 510

Now *that's* more like it. All of the code needed to put the two functions together is now abstracted and isolated to `pipeline`. All we have to do to compose functions is call `pipeline` on the functions we want to compose, and store the result in a variable.

### More simple use cases

Now that we have the ability to `pipeline` smaller functions into a larger, reusable function, let's see some use cases.

Because we used `var steps = [].slice.call(arguments, 0);`, `pipeline` will work with any number of steps. The following examples both use three smaller steps, but you can use any number of steps, 0-N.

Finding the magnitude of a vector: 
	
    // building some small functions out
	function square(x) { return x * x; }
	function squareAll(nums) { return nums.map(square); }
    function add(x, y) { return x + y; }
    function addAll(nums) { return nums.reduce(add, 0); }
    
    var magnitude = pipeline(squareAll, addAll, Math.sqrt);
    
    magnitude([3, 4]); //=> 5
    magnitude([2, 2, 2, 2]); //=> 4
    
Doing DOM manipulation (uses jQuery). Suppose we want a function to make `.special` elements in a selection of DOM snippets `.active` and `jQuery.prototype.show()` them:

	function getSpecials(elems) {
    	return elems.find('.special');
    }
	function makeActive(elems) {
    	return elems.addClass('active');
    }
    function show(elems) {
    	return elems.show();
    }
    
    var activateShowSpecials = pipeline(
    	getSpecials,
        makeActive,
        show
	);
    var mySelection = jQuery('anytag.anyClass');
    activateShowSpecials(mySelection);
    //=> Returns the elements, having now performed the steps.

### Complex: `pipeline` of `pipeline`s

Since `pipeline` accepts functions and returns functions, we can pass the result of one pipeline into another! This is particularly helpful if there is a complex set of relationships between frequently used functions.

This is often the case when building up a vocabulary of functions operating on a certain kind of data. Let's build up some pieces of a vernacular of functions operating on `Number`s.

Our two most basic math functions:

	function times(x) {
    	return function(y) {
        	return x * y;
        };
    }
    
    function plus(x) {
    	return function(y) {
        	return x + y;
        };
    }
    
    
Building some simple pipes: 

	var addTwoThenDouble = pipeline(add(2), times(2));
    var minusSixThenRoot = pipeline(add(-6), Math.sqrt);
    var muchoMath = pipeline(addTwoThenDouble, minusSixThenRoot);
    var equivalentMath = pipeline(
    	add(2),
        times(2),
        add(-6),
        Math.sqrt
    );
    
    
Putting it all together: 

	// Many ways to go from 33 => 8

	// Manually in sequence
    add(2)(33); //=> 35
    times(2)(35); //=> 70
    add(-6)(70); //=> 64
    Math.sqrt(64); //=> 8
    
	// Pipelines in sequence:
	//   note that I pass result of one into another
	addTwoThenDouble(33); //=> 70
    minusSixThenRoot(70); //=> 8

    // Doing this in one step
    minusSixThenRoot(addTwoThenDouble(33)); //=> 8
    
    // With our pipeline-of-pipelines
    muchoMath(33); //=> 8
    
    // With our long single pipeline
    equivalentMath(33); //=> 8
    
Information overload! We can see that a `pipeline`-of-`pipeline`s is the same as making one large `pipeline` with all the same steps. One would use this technique when both the final pipeline *and* the intermediate pipelines are useful.

### Wrapping it up

We now have a way to use higher-order functions to dramatically simplify the construction of many steps into a single step. `pipeline` is an excellent way to build complexity, as seen with the examples here and in the wild, with analogous tools like the Bash `|` command.

Have a pattern like this that you would like to see implemented in JavaScript? Request a walkthrough like this one with a tweet [@ayetempleton](https://twitter.com/intent/tweet?text=%40ayetempleton%2520re%3A%2520blog.templeton.host%2Fjavascript-function-composition)!


 --- 
 
##### Appendix: `pipeline` implementations

As created in this post: 

	function runStep(val, step) {
    	return step(val);
    }
    function pipeline() {
    	var steps = [].slice.call(arguments, 0);
        return function(input) {
        	return steps.reduce(runStep, input);
        };
    }
    
Advanced: Using `Function.prototype.bind` to save space:

	function runStep(val, step) {
    	return step(val);
    }
    function pipeline() {
    	return [].reduce.bind(arguments, runStep);
    }

Single-function, without `runStep`:

	function pipeline() {
    	var steps = [].slice.call(arguments, 0);
        return function(input) {
        	return steps.reduce(function(val, step) {
            	return step(val);
            }, input);
        };
    }

Math-style composition, which executes *right-to-left*. This is also very similar to the manual compositions nesting syntax. Simply use `reduceRight` instead of `reduce`:

	function runStep(val, step) {
    	return step(val);
    }
	function compose() {
    	var steps = [].slice.call(arguments, 0);
        return function(input) {
        	return steps.reduceRight(runStep, input);
        };
    }
    
    // How this differs from pipeline...:
    // Manually done
    function multi(x) {
    	return a(b(c(x)));
    }
    // equivalent compose call:
    var multi = compose(a, b, c);
    // equivalent pipeline call (note reverse order):
    var multi = pipeline(c, b, a);

	