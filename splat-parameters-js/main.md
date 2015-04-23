Are you coming from a Ruby background, and wondering how to do something like Ruby's `*lastArgs` splat parameter syntax? Read on for a nifty technique to simulate the same functionality with a JavaScript higher-order function.


### Splat Parameters in Ruby

A splat parameter is a parameter which collects all arguments `N`-`M` passed into a function, where `N` is last named parameter in a function definition, and `M` is the last argument actually passed into the function at runtime. This pattern is helpful when providing functions that may accept a variable number of comma-delimited arguments.

Let's see some simple use cases in Ruby before we move on to JavaScript. If this syntax is new to you, [EndOfLine](https://endofline.wordpress.com/) has [a great tutorial](https://endofline.wordpress.com/2011/01/21/the-strange-ruby-splat/) on Ruby splat arguments.

	// Method to set X keys in a hash to the same value
    def multiKeySet hash, value, *keys
    	keys.reduce(hash) do |memo, key|
        	memo[key] = value
            memo
        end
    end
    
    hash = multiKeySet({}, 'X', :foo, :bar, :baz)
    hash #=> {:foo=>"X", :bar=>"X", :baz=>"X"}
    
Note that (1) we supplied 5 arguments, even though the method looks like it should only take 3, and (2) arguments 3-5 are collected and passed in as a single `Array`, `keys`.

Let's make a "merge" function that takes many hashes and produces a single hash, which is the (shallow) merge of all the hashes.

	def mergeHashes *hashes
		hashes.reduce({}, &:merge)
	end
    
    merged = mergeHashes({foo: 1}, {bar: 2}, {baz: 3})
    merged #=> {foo:1, bar:2, baz:3}
    
This time, we used the splat as the only argument, so the method rolls all passed arguments into an `Array` as the single named parameter `hashes`.

Now that we know what we want to be able to do, we can try emulating this Ruby language feature in JavaScript.


### Splatting in JavaScript

Unlike Ruby, JavaScript does not ship with syntax that allows us to collect `N`-`M` arguments. We want the same functionality, so let's write manual versions of the same functionality that we see in Ruby. 

#### Manual Splats
    
First, the `multiKeySet` example, manually:

	// Since no splat, *keys from definition.
    // JavaScript allows variable-length calls, so this is ok.
	function multiKeySet(hash, value) {
    	// This is what we use to produce keys.
        // We are taking all arguments 2-N inclusive.
    	return [].slice.call(arguments, 2)
        	.reduce(function(memo, key) {
        		memo[key] = value;
            	return memo;
        	}, hash);
    }
    
    var hash = multiKeySet({}, 'X', 'foo', 'bar', 'baz');
    hash //=> {foo:"X", bar:"X", baz:"X"}
    
The key pattern here is the `[].slice.call(arguments, N)` step, in which we simulate `*keys` in Ruby by taking advantage of JavaScript's ability to take variable `arguments` length. Note that `arguments` effectively collects *all* arguments, but we only want 2-N in this case. We also take care to use `[].slice.call` notation, because in JavaScript the `arguments` object is *not* of type `Array`, which has the `slice` method, but of type `Arguments`. Using `[].slice.call` tells JavaScript to borrow the `slice` method from `Array` and use it on the `Arguments`-type object anyway.

In this second example, we see much of the same. JavaScript lacks an inbuilt `merge` function for `Object`, so we implement that manually too.

	function merge(hash, other) {
    	return Object.keys(other).reduce(function(memo, key) {
            memo[key] = next[key];
            return memo;
		}, hash);
    }
	function mergeHashes() {
    	return [].reduce.call(arguments, merge, {});
    }
    
This example also needs a manual call to an `Array` method, this time skipping `slice`, because we already want all of `arguments`, and going straight to `reduce`. Still, this is not as clean as we want.


#### Abstracting Splats

While we can still accomplish the same things that Ruby lets us do, because JavaScript both (1) allows access to the entire `arguments` object and (2) allows any function to be passed any number of arguments at runtime, we want to clean this up. 

Ideally, we want to both (1) convey intent and (2) shorten up all subsequent code using splat-like notions of N-M argument collection. We can do both, by doing some tricky functional programming, wherein we make a function `splatted`, which takes function definitions of Ruby-like splat syntax, and then returns a function that implements this logic for us.

Here's what the `splatted` higher-order function looks like:

	// Accept a function, which will have splat-like
    //   arguments defined.
	function splatted(splattable) {
    	// Return the working splat-like function
    	return function() {
        	// Gather all arguments as Array
        	var args = [].slice.call(arguments, 0);
            // Splice out the first arguments
            var newArgs = args.splice(0, splattable.length - 1);
            // Push in remaining arguments as an Array
            newArgs.push(args);
            // Invoke the original function, with gathered
            //   last arguments Array as last element.
            return splattable.apply(this, newArgs);
        };
    }
    
Not the most intuitive code of all time, but it gets the job done, and will significantly improve readability of our two example functions. Before we rewrite the two examples from above, let's take a look at the basic behavior of this `splatted` function.

	// Assuming splatted above is available...
    var gatherAll = splatted(function(args) {
    	return args;
    });
    gatherAll('foo', 'bar'); //=> ['foo', 'bar']

    var gatherAfterTwo = splatted(function(a, b, restOfArgs) {
    	return restOfArgs;
    });
    gatherAfterTwo(1, 2, 3, 4, 5, 6); //=> [3, 4, 5, 6]
    
As we can see, we can now emulate the Ruby splat behavior, while simulataneously reducing code and increasing readability. The intent of the code above is obvious - we are making a "splat" function out of the simpler-looking definitions. No more `[].slice.call`!


#### Using `splatted` Functions

This time around, our `multiKeySet` and `mergeHashes` functions will be much cleaner and clearer. We use `splatted` to convey to the reader that the last `argument`s will be collected, so we should make the function definitions use plurals, like we did in `*keys` and `*hashes` with Ruby.

Rewriting `multiKeySet`, with `splatted`:

	var multiKeySet = splatted(function(hash, value, keys) {
    	return keys.reduce(function(memo, key) {
        	memo[key] = value;
            return memo;
        }, hash);
    });
    
As we can see, the function body's code itself is far cleaner - no strange `[].slice.call`-ing, no reference to `arguments`, and no magical `2` denoting the number of arguments to skip before collection.

Moving on to out next example:

	// Still need this piece.
	function merge() {
    	function merge(hash, other) {
            return Object.keys(other).reduce(function(memo, key) {
                memo[key] = next[key];
                return memo;
            }, hash);
        }
    }
    
    var mergeHashes = splatted(function(hashes) {
    	return hashes.reduce(merge, {});
    });

This time, we can actually name `hashes` like one would in Ruby, and convey to the reader, in English-like syntax, that *`mergeHashes` is a splatted function that passes in (an `Array` of) `hashes`...*

Much better! All the functions work the same way, but we have dramatically improved code readability.


### Wrap Up

As a feature-light language, JavaScript sometimes lacks conveniences that other, more feature-rich languages like Ruby provide. Fortunately, we can take advantage of JavaScript's functional nature to write new features into our own code, by producing higher-order functions that implement said features.

If you find yourself doing the same kind of thing over and over again as setup code within functions, try to see if your repeated pattern is a candidate for abstraction into a higher-order function. 

Have a pattern like this that you would like to see implemented in JavaScript? Request a walkthrough like this one with a tweet [@ayetempleton](https://twitter.com/intent/tweet?text=%40ayetempleton%2520re%3A%2520blog.templeton.host%2Fjavascript-splat-functions)!




    
    