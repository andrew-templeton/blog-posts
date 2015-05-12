This post dives deep on `Array.prototype`'s ECMA5 methods: `forEach`, `map`, `reduce`/`reduceRight`, `filter`, `every`, and `some`. At the end of this (long) lesson, you should be able to:

* Use all of these methods in the correct context  
* Understand how they are implemented
* Begin to form an intuition for when to use the methods
* Pick up some discrete math knowledge
* Know what `map-reduce` means at a base level

---

ECMA5, the latest version of JavaScript's backing specification that is in common use in the wild, adds many functional methods to the `Array.prototype`. These methods, like `forEach`, `map`, `reduce`/`reduceRight`, `filter`, `every`, and `some`, were previously only usable with custom polyfills or through use of libraries like [Underscore](http://underscorejs.org).

These methods [have been around for a while now](http://kangax.github.io/compat-table/es5/#Array.prototype.map), but I still see engineers at many companies that do not leverage the methods to their full potential, instead preferring `for` and `while` loops in many cases that would benefit from these higher-order methods. 

Part of this lack of adoption seems to extend from difficulties understanding how JavaScript allows functions to be used as arguments for other functions - the *higher-order functions* of which we speak.
<a name="layout"></a>
First, let's take a look at what each method does. We will then implement each of them ourselves[\*](#disclaim_1), and finally see some interesting extensions to their use that are applicable to plenty of problem spaces.

Let's get started!

---

## Method Index
[`Array.prototype.forEach`](#foreach)  
[`Array.prototype.map`](#map)  
[`Array.prototype.reduce`/`reduceRight`](#reduce)  
[`Array.prototype.filter`](#filter)  
[`Array.prototype.every`](#every)  
[`Array.prototype.some`](#some)  

---

<a name="foreach"></a>
### Array.prototype.forEach

`forEach ` helps with simple iteration over the length of an `Array`, from left-to-right. Instead of writing `for` loops all of the time, we can pass a function into `forEach`, anonymous or otherwise, which will be executed for each (get it?) element in the `Array`.

###### Using `forEach` 
    /*
     * Test case: Add the index of the element to each element's foo property,
     *   making it the index if there's no Number property foo.
     * In:  [{foo: 3}, {foo: 10}, {}]
     * Out: undefined
     * *** "In" is mutated: [{foo: 3}, {foo: 11}, {foo: 2}]
     */

Old way with `for`:

    function mutateFoos(arr) {
    	for (var index = 0; index < arr.length; index++) {
        	arr[index].foo = 'number' == typeof arr[index].foo
            	? obj.foo + index
                : index;
        }
    }
    
Hybrid, much like internal implementation of `forEach`:

    function incrementFoo(obj, index) {
    	obj.foo = 'number' == typeof obj.foo
        	? obj.foo + index
            : index;
    }
    function mutateFoos(arr) {
    	for (var index = 0; index < arr.length; index++) {
        	incrementFoo(arr, index);
        }
    }
New way with `forEach`:

    function incrementFoo(obj, index) {
    	obj.foo = 'number' == typeof obj.foo
        	? obj.foo + index
            : index;
    }
    function mutateFoos(arr) {
		// Passing the function; reads like English!
    	arr.forEach(incrementFoo);
    }
	
    
###### A simple implementation of `forEach`

The callback function supplied in the last example above should have `element`, `index` and `array` passed in as arguments, in that order, on each iteration. Most callbacks do not use all three parameters, as seen in the example above, which does not use the third parameter. This implementation looks much like the second, "hybrid" example above.

    Array.prototype.forEach = function(callback) {
    	for (var index = 0; index < this.length; index++) {
        	callback(this[index], index, this);
        }
    };
    
Note that `forEach` does *not* return anything. We passed in `element`, which is just the element of the current `this` at `index`, `index` itself, which is the point in the `for` loop, and `this` itself, which is usually the `Array` being iterated over itself.

###### When to use `forEach`

Much like a `for` loop, there is no return value. This makes 	`forEach` useful in essentially two cases - mutations of an object itself, or mutations of closure values - values accessible to inner function we run many times.

We have already seen a "mutation of objects" use case, above. How about a "mutation of closure variables" example? We will find the magnitude of a vector of any length - simply use the Pythagorean Theorem. We will only use the `element` argument to the `forEach` callback parameter.

    function magnitude(vector) {
    	var closureSumSquares = 0;
        vector.forEach(function(elem) {
        	closureSumSquares += Math.pow(elem, 2);
        });
        return Math.sqrt(closureSumSquares);
    }
    magnitude([3, 4]); // 5
    magnitude([3]); // 3
    magnitude([2, 2, 2, 2]); // 4

---
<a name="map"></a>
### Array.prototype.map

`map` extends the iterative notion of `forEach`, but actually returns a value. `map` takes a callback, and returns another `Array` which represents the return values of the callback as you iterate over the original collection/`Array`. Much like `forEach`, the callback can expect the same inputs, and iterates left-to-right.

###### Using `map`

    /*
     * Test case: Return a new Array, containing all the foo
     *   properties from objects in the input Array. No mutations.
     * In:  [{foo: 1}, {foo: 2}, {bar: 3}, {foo: 'Whee!'}]
     * Out: [1, 2, undefined, 'Whee!']
     */
     
Old way with `for`:

    function getFoos(arr) {
    	var newArr = [];
    	for (var index = 0; index < arr.length; index++) {
        	newArr.push(arr[index].foo);
        }
        return newArr;
    }
    

Hybrid, much like internal implementation of `map`:

    function getFoo(element) {
    	return element.foo;
    }
	function getFoos(arr) {
    	var newArr = [];
        for (var index = 0; index < arr.length; index++) {
        	newArr.push(getFoo(arr[index]));
        }
        return newArr;
    }
    
New way with `map`:

	function getFoo(element) {
    	return element.foo;
    }
    function getFoos(arr) {
    	return arr.map(getFoo);
    }
    
###### A simple implementation of `map`

In the same manner as `forEach`, the callback function supplied in the last example above should have `element`, `index` and `array` passed in as arguments. The first implementation looks much like the second, "hybrid" example above, and the second implementation is a more advanced approach using `forEach`.

	// With a for loop, like the hybrid example in this section
	Array.prototype.map = function(callback) {
    	var array = [];
        for (var index = 0; index < this.length; index++) {
        	array.push(callback(this[index], index, this));
        }
        return array;
    }
    // With forEach, to extend upon what we learned above
    Array.prototype.map = function(callback) {
    	var array = [];
        [].forEach.call(this, function(elem, idx, arr) {
        	array.push(callback(elem, idx, arr));
        });
        return array;
    };
    
Unlike `forEach`, `map` actually returns the collective results of each function - we have "mapped" over the collection, returning a new collection, that is related by the provided callback.

If I lost you in the second example, when I did `[].forEach.call`, that's okay - this is an advanced technique that handles edge cases, when `this` has been bound to be something other than an `Array` (like a `NodeList`). It forces `Array.prototype.forEach` to keep using the provided `this` inside it's own body. I forsee another `this`/`call` post topic!

###### When to use `map`

Unlike `forEach`, `map` has a useful return value. While it may be tempting to use `map` like `forEach` and do mutations since they both have access to the same closure variables and iterate the same way, in the majority of cases, this is a poor choice. `map` usually signifies to a reader that only the return value is being used, since `forEach` is so often used for mutations.

Use `map` when you are trying to create a new `Array` from an existing collection of elements. The three most common cases are when pulling inner values from a collection of objects, which we saw with `foo` above, doing math computations on each elements, and creating more complex structures from an existing collection.

Our example of a math use case is another version of the `magnitude` function we implemented as an example in the `forEach` section:

	function magnitude(vector) {
    	var sum = 0;
    	function addToSum(x) { sum += x; }
        function square(x) { return Math.pow(x, 2) };
        vector.map(square).forEach(addToSum);
        return Math.sqrt(sum);
    }
    magnitude([3, 4]); // 5
    magnitude([3]); // 3
    magnitude([2, 2, 2, 2]); // 4
    
For creation of more complex structures, let's make some script tags based on a list of URLs:

	function appendHead(elem) {
    	document.head.appendChild(elem);
    }
    function toScript(url) {
    	var script = document.createElement('script');
    	script.src = url;
    	return script;
    }
    function appendScriptsToHead(urls) {
        urls.map(toScript).forEach(appendHead);
    }
    appendScriptsToHead(['foo.js', 'bar.js']);
    // All scripts now in <head> tag
    
---
<a name="reduce"></a>
### Array.prototype.reduce

`reduce` is `map`'s sibling function - while `map` returns an `Array` after running each element of another `Array` through a callback, `reduce` returns a single value. `map` can be seen as a translation function, while `reduce` is a "rollup" function.

`reduce` replaces another common use case for the `for`/`while` loop - when we want to produce a single value based on the content of an `Array`, be it a sum, a product, or a rolled-up `Object`.

###### Using `reduce`

    /*
     * Test case: Return the sum of an Array of Number elements.
     * In: [1, 2, 3, 4, 5]
     * Out: 15
     */
     
Old way with `for`:

	function sum(array) {
	    var total = 0
    	for (var index = 0; index < array.length; index++) {
        	total += array[index];
        }
        return total;
    }

Hybrid, much like internal implementation of `reduce`:

	function add(x, y) {
    	return x + y;
    }
	function sum(array) {
    	var total = 0;
        for (var index = 0; index < array.length; index++) {
        	total = add(total, array[index]);
        }
        return total;
    }

New way with `reduce`:

	function add(x, y) {
    	return x + y;
    }
    function sum(array) {
    	return array.reduce(add, 0);
    }
    
###### A simple implementation of `reduce`

Unlike `forEach` and `map`, which only take a callback, `reduce` takes a second argument, often called `seed`, which is a starting value. Much like the `for` loop and hybrid `for` loop examples set `var sum = 0` in their first lines, we `seed` `reduce` with `0`.

Furthermore, the callback to `reduce` is provided four arguments, instead of three. Like `map` and `forEach`, arguments `element`, `index`, and `array` are provided, but we also need to provide the value so far in the progression, most often called `memo`. 

`memo` is either the return value of callback in the last step we made while iterating through the `Array`, or in the case of the first step, the `seed` value, because no steps have been made yet.

Here are two implementations of `reduce`, one using a simple `for` loop, and the other, more advanced approach, using `forEach`.

	// With a for loop, like the hybrid example in this section
    Array.prototype.reduce = function(callback, seed) {
    	var memo = seed;
        for (var index = 0; index < this.length; index++) {
        	memo = callback(memo, this[index], index, this);
        }
        return memo;
    };
    // With forEach, to extend upon what we learned above
    Array.prototype.reduce = function(callback, seed) {
    	var memo = seed;
        [].forEach.call(this, function(elem, index, arr) {
        	memo = callback(memo, elem, index, arr);
        });
        return memo;
    };

Like `map`, `reduce` returns a value. Unlike `map`, `reduce` returns the collective single result of each function - we have "reduced" the collection, returning a single value.

If `call` is still unfamiliar, see the note in the `Array.prototype.map` implementation section.


###### When to use `reduce`

Like `map`, `reduce` returns a value, so the same expectations about avoiding mutations apply to `reduce` as do to `map` - try not to mutate anything external in a callback supplied to `reduce`.

Use `reduce` when you are trying to create a new single value from an existing collection of elements, rolling them up. The three most common cases are when doing math computations like sums and products of elements, which we saw with `sum` above, creating a more complex single `Object` structure from an existing collection, and running several pieces of logical tests in a row to get a single `true`/`false`.

Creating a more complex structure with `reduce`:

	function mergeKeys(memoObj, nextObj) {
    	return Object.keys(nextObj)
            .reduce(function(obj, nextKey) {
            	obj[nextKey] = nextObj[nextKey];
            	return obj;
            }, memoObj);
    }
	function rollupObjects(objects) {
    	return objects.reduce(mergeKeys, {});
    }
    rollupObjects([
    	{foo: 'A'},
        {bar: 'B'},
        {baz: 'C'}
	]); // {foo: 'A', bar: 'B', baz: 'C'}
    
Running several logical tests in a row:

	
	function functionalAND(tests, testValue) {
    	return tests.reduce(function(memo, nextTest) {
    		return memo && nextTest(testValue);
    	}, true);
    }
    // Usage:
    function isNumber(x) { return 'number' == typeof x }
    function isPositive(x) { return x > 0; }
    function isEven(x) { return x % 2 === 0; }
    var isPositiveEvenNumber = [
    	isNumber,
        isPositive,
        isEven
    ];
    functionalAND(isPositiveEvenNumber, 2); // true
    functionalAND(isPositiveEvenNumber, 3); // false
    functionalAND(isPositiveEvenNumber, -1); // false
    functionalAND(isPositiveEvenNumber, 'hehe'); // false
    
###### Noteworthy: Array.prototype.reduceRight

I will not do a full section on this one, because `reduceRight` is functionally identical to `reduce`, except that iteration over elements begins from the right side (the last element), instead of the left side (the first element).

---
<a name="filter"></a>
### Array.prototype.filter

Another value producer, `filter` returns an `Array` much like `map`. However, `filter` returns an `Array` of length less than or equal to the input `Array`, by "filtering" down to elements in the original `Array` that meet the test condition defined by a callback supplied. 

Filter accepts one argument, the callback test function. The callback supplied can expect to receieve the same inputs as in `forEach` and `map` - `element`, `index` and the whole `array`.

###### Using `filter`

    /*
     * Test case: Return a new Array, containing only
     *   elements with even, numeric foo properties.
     * In:  [{foo: 3}, {foo: 4}, {bar: 4}, {foo: true}, {foo: -2}]
     * Out: [{foo: 4}, {foo: -2}]
     */
     
Old way with `for`:

	function onlyEvenFoos(array) {
    	var filtered = [];
        for (var index = 0; index < array.length; index++) {
        	if (array[index] &&
            	typeof array[index].foo == 'number' &&
                array[index].foo % 2 === 0) {
            	filtered.push(array[index]);
            }
        }
        return filtered;
    }
    
Hybrid, much like internal implementation of `filter`:
	
    function hasEvenFoo(elem) {
    	return elem &&
        	'number' == typeof elem.foo &&
            elem.foo % 2 === 0;
    }
	function onlyEvenFoos(array) {
    	var filtered = [];
    	for (var index = 0; index < array.length; index++) {
        	if (hasEvenFoo(array[index], index, array)) {
            	filtered.push(array[index]);
            }
        }
        return filtered;
    }
    
New way with `filter`:

	function hasEvenFoo(elem) {
    	return elem &&
        	'number' == typeof elem.foo &&
            elem.foo % 2 === 0;
    }
	function onlyEvenFoos(array) {
    	return array.filter(hasEvenFoo);
    }
    

###### A simple implementation of `filter`

Just like `forEach` and `map`, callbacks supplied to `filter` should be passed `element`, `index`, and the `array` itself. The first implementation is very similar to the hybrid approach above, the second uses `forEach`, and the third uses `reduce` (with the single returned value being the filtered `Array` instance).

Using a `for` loop:

	Array.prototype.filter = function(testFunc) {
    	var filtered = [];
        for (var index = 0; index < this.length; index++) {
        	if (testFunc(this[index], index, this)) {
            	filtered.push(this[index]);
            }
        }
        return filtered;
    };
    
Using `forEach`:

	Array.prototype.filter = function(testFunc) {
    	var filtered = [];
        [].forEach.call(this, function(elem, index, arr) {
        	if (testFunc(elem, index, arr)) {
            	filtered.push(elem);
            }
        });
        return filtered;
    };
    
With `reduce`:

	Array.prototype.filter = function(testFunc) {
    	return [].reduce.call(this, function(mem, nxt, idx, arr) {
            return mem.concat(testFunc(nxt, idx, arr) ? nxt : []);
        }, []);
    };
    
    
The first example should be quite familiar, given the "hybrid" example in the test usage section. The second approach simply translates the `for` loop to use `forEach`, but is essentially the same approach. The third approach is the most advanced, opting to seed `reduce` with an empty `Array`/`[]`, and reducing by choosing whether or not to concatenate values to `mem` based upon truthiness of the `testFunc`.

Note that in all three approaches, we are *not* testing for ` === true`, but for truthiness. This means that if any call to `testFunc` returns a value other than the JavaScript falsey values (`[null, undefined, 0, false, NaN, '']`), filter will include the element in the returned `Array`.

Also note that `filter` uses left-to-right execution, so except in cases where a user manually manipulates the order of elements (a bad practice), elements will maintain the same relative order in the new returned subset.


###### When to use `filter`

Like `map` and `reduce`, `filter` has a useful return value, and therefore it is best to avoid using it as a mutation function and stick to just using it as an `Array` subset builder.

Filter is often used to filter down a collection of DOM elements after a `querySelectorAll` (or similar) call, or as a way to filter to only entries in a list that the user asks for, like the examples above.

Filtering DOM nodes:

	// ASSUME THIS HTML...
    <ul id="comidas">
        <li>Fish Tacos</li>
        <li>Enchiladas</li>
        <li>Beef Tacos</li>
        <li>Quesadillas</li>
        <li>Tacos al Pastor</li>
    </ul>
    function tacoRelated(node) {
    	return node.textContent.indexOf('Taco') !== -1;
    }
    function addYumClass(node) {
    	node.classList.add('yum');
    }
	function yumTacos(nodes) {
    	[].filter.call(nodes, tacoRelated).forEach(addYumClass);
    }
    yumTacos(document.querySelectorAll('#comidas li'));
    // HTML looks like this...:
    <ul id="comidas">
        <li class="yum">Fish Tacos</li>
        <li>Enchiladas</li>
        <li class="yum">Beef Tacos</li>
        <li>Quesadillas</li>
        <li class="yum">Tacos al Pastor</li>
    </ul>
    
<a name="every"></a>
### Array.prototype.every

Now we begin the foray into "quantifier" methods. The `every` method is identical in functionality to the `for all <element> in <set>` standard clause in discrete mathematics and logic. 

`every` replaces a common use of `for`, in which one iterates over an `Array` of objects and runs the same test on every element. One interesting property of `every`: an empty `Array` (empty set) will return true, no matter what test case is provided, because there were not any test failures. This may seem strange, but [the reasoning holds in math](http://math.stackexchange.com/questions/202452/why-is-predicate-all-as-in-allset-true-if-the-set-is-empty).

###### Using `every`

	/*
     * Test cases: Check if all elements have an even,
     *   numeric foo property value.
     * In:  [{foo: 2}, {foo: 4}]
     * Out: true
     * In:  [{foo: 2}, {foo: 3}]
     * Out: false
     * ** The special empty set case
     * In:  []
     * Out: true
     
Old way using `for`:

	function allEvenFoos(array) {
        for (var index = 0; index < array.length; index++) {
        	if (!(array[index] &&
            	'number' == array[index].foo &&
                array[index].foo % 2 === 0)) {
                    return false;
            }
        }
        return true;
    }
    
Hybrid, much like internal implementation of `every`:
	
    function evenFoo(elem) {
    	return elem
        	&& 'number' == typeof elem.foo
            && elem.foo % 2 === 0;
    }
	function allEvenFoos(array) {
    	for (var index = 0; index < array.length; index++) {
        	if (!evenFoo(array[index], index, array)) {
            	return false;
            }
		}
        return true;
    }

New way with `every`:

	function evenFoo(elem) {
    	return elem
        	&& 'number' == typeof elem.foo
            && elem.foo % 2 === 0;
    }
    function allEvenFoos(array) {
    	return array.every(evenFoo);
    }
    
###### A simple implementation of `every`

Just like `forEach`, `map`, and `filter`, `every` expects a single callback function, and that callback function is provided `element`, `index`, and the `array` itself. We can see from the "hybrid" example above how the special empty set behavior arises from an implementation with a `for` loop quite easily.

Three implementations this time - one using `for`, another using `forEach`, and the last using `reduce`. The `for` implementation is slightly more efficient, due to early exits from the loop on `false` cases, but I provide all three here to help practive use of `forEach` and `reduce`. All three are functionally identical.

With just a `for` loop:

	Array.prototype.every = function(callback) {
    	for (var index = 0; index < this.length; index++) {
        	if (!callback(this[index], index, this)) {
            	return false;
            }
        }
        return true;
    };
    
With `forEach`:

	Array.prototype.every = function(callback) {
    	var passing = true;
        [].forEach.call(this, function(elem, index, arr) {
        	passing = passing && callback(elem, index, arr);
        });
    	return !!passing;
    };
    
With `reduce`:

	Array.prototype.every = function(callback) {
    	return !![].reduce.call(this, function(mem, nxt, idx, arr) {
        	return mem && callback(nxt, idx, arr);
        }, true);
    };

Note that we seeded `reduce` with true, so the empty set case will still pass. The `!!` idiom in JavaScript that I use in the `forEach` and `reduce` implementations forces any value to be `true` or `false`, and thus coerces any truthy/falsey values to `true`/`false`, respectively.

###### When to use `every`

Like `map`, `reduce`, and `filter`, when using `every`, mutation should be avoided. Try to use `every` when you have an `Array` to check a condition against - when you need to know "do all elements in this set return true".

There are two main cases for `every`. One, which we have seen, is to ensure that a single test passes for all data structures in a set. The other, more advanced use, is the opposite: when checking if "every" test condition passes on a single data structure.

In this case, we take a shot at the second case. We want to make a function that checks if a passed value is all of: Numeric, positive, divisible by 4.

	function isNumeric(x) { return 'number' == typeof x; }
    function isPositive(x) { return x > 0; }
    function isDivisibleByFour(x) { return x % 4 === 0; }
    var positiveNumDiv4 = [
    	isNumeric,
        isPositive,
        isDivisibleByFour
    ];
    function isPositiveNumDiv4(x) {
    	return positiveNumDiv4.every(function(test) {
        	return test(x);
        });
    }
    isPositiveNumDiv4(2); // true
    isPositiveNumDiv4(1); // false
    isPositiveNumDiv4(-2); // false
	isPositiveNumDiv4('foobar'); // false
    [2, 1, -2, 'foobar'].every(isPositiveNumDiv4); // false

---
<a name="some"></a>
### Array.prototype.some

Like `every`, `some` returns a boolean value based on a test callback and an array. However, unlike `every`, which checks for non-failure of every element in a set, `some` checks for at least one pass of the callback condition. 

`some`: "At least one passes"/"Not all failures"  
`every`: "All of them pass"/"None of them fail"

Other than this differentiation, they function very similarly, accepting a single test condition callback, which in turn expects the same arguments to be passed: `element`, `index` and the `array` itself.

Like `every`'s special case for empty sets, `some` has a special case - empty sets always return `false`, because no element in the `Array` can pass the test.

###### Using `some`

	/*
     * Test case: See if any object have a numeric, 
     *   positive, even foo property value.
     * In:  [{foo: 3}, {foo: 2}]
     * Out: true
     * In:  [{}, {foo: 3}]
     * Out: false
     * ** The special empty set case
     * In:  []
     * Out: false
     */
     
Old way with `for`:

	function anyEvenFoos(array) {
    	for (var index = 0; index < array.length; index++) {
        	if (array[index] &&
                'number' == typeof array[index].foo &&
                array[index].foo % 2 === 0) {
                    return true;
            }
        }
        return false;
    }
    
Hybrid, much like internal implementation of `some`:

	function evenPosFoo(elem) {
    	return elem &&
        	'number' == typeof elem.foo &&
            elem.foo % 2 === 0;
    }
    function anyEvenFoos(array) {
    	for (var index = 0; index < array.length; index++) {
        	if (evenPosFoo(array[index], index, array)) {
            	return true;
            }
        }
        return false;
    }
    
New way with `some`:

	function evenPosFoo(elem) {
    	return elem &&
        	'number' == typeof elem.foo &&
            elem.foo % 2 === 0;
    }
    function anyEvenFoos(array) {
    	return array.some(evenPosFoo);
    }
    
###### A simple implementation of `some`

Just like `every` loops over cases and returns `true`/`false`, `some` does the same. There are very few differences in the code between `every` and `some`.

As with `every`, in the "hybrid" example above, the special case for empty set reveals itself clearly - in an `Array` of no length, the `for` loop is skipped, and the final `return false;` statement is hit.

The first implementation generalizes the same approach as "hybrid" above, the second uses `forEach`, the third uses `reduce`, and the fourth uses `every` to show logical relations. The first implementation is slightly more efficient than the rest, but all examples are valueable in understanding the relationships between these functions.

With a `for` loop:

	Array.prototype.some = function(callback) {
    	for (var index = 0; index < this.length; index++) {
        	if (callback(this[index], index, this)) {
            	return true;
            }
        }
        return false;
    };
    
With `forEach`: 

	Array.prototype.some = function(callback) {
		var anyPassed = false;
        [].forEach.call(this, function(elem, idx, arr) {
        	anyPassed = anyPassed || callback(elem, idx, arr);
        });
        return !!anyPassed;
    };
    
With `reduce`: 

	Array.prototype.some = function(callback) {
    	return !![].reduce.call(this, function(mem, nxt, idx, arr) {
        	return mem || callback(nxt, idx, arr);
        }, false);
    };

With `every`:

	Array.prototype.some = function(callback) {
    	return ![].every.call(this, function(elem, idx, arr) {
        	return !callback(elem, idx, arr);
        });
    };

Note that we seeded `reduce` with true, so the empty set case will still return `false`. The `!!` idiom in JavaScript that I use in the `forEach` and `reduce` implementations forces any value to be `true` or `false`, and thus coerces any truthy/falsey values to `true`/`false`, respectively.

For my math-inclined readers, the definition of `some` in terms of `every` in the last example follows one of discrete math's "[De Morgan's Laws](http://en.wikipedia.org/wiki/De_Morgan%27s_laws)", that is...:  
`(x1 || x2 ... || xn) <=> !(!x1 && !x2 ... && !xn)`  
Or, "`some` is `true` in the set means the same as not `every` is `false` in the set."

###### When to use `some`

Like `map`, `reduce`, `filter`, and `every`, when using `some`, mutation should be avoided. Try to use `some` when you have an `Array` to check a condition against - when you need to know "do any elements in this set return true".

There are two main cases for `some`. One, which we have seen, is to ensure that a single test passes for any data structures in a set. The other, more advanced use, is the opposite: when checking if "some" test condition in a set of conditions passes on a single data structure.

In this example, we take a shot at the second case. We want to make a function that checks if a passed value is Numeric, positive, and divisible by any of: `[2, 3, 5, 7]`. We will use both `every` and `some` together, as well as `map`, to get an idea of how to build complex logical tests.

	function divisibilityCheck(divisor) {
    	return function(x) {
        	return x % divisor === 0;
        };
    }
    function numeric(x) {
    	return 'number' == typeof x;
    }
    function positive(x) {
    	return x > 0;
    }
    function tests(x) {
    	return function(test) {
        	return test(x);
        };
    }
    var divisors = [2, 3, 5, 7];
    var divisionChecks = divisors.map(divisibilityCheck);
    function someDivisibleCheck(x) {
    	return divisionChecks.some(tests(x));
    }
    var numPosDivisible = [
    	numeric,
        positive,
        someDivisibleCheck
    ]
    function isDivisiblePositiveNumber(x) {
    	return numPosDivisible.every(tests(x));
    }

	isDivisiblePositiveNumber(-1); // false
    isDivisiblePositiveNumber(2); // true
    isDivisiblePositiveNumber(0); // false
    isDivisiblePositiveNumber(5); // true
    isDivisiblePositiveNumber('not num'); // false
    isDivisiblePositiveNumber(3); // true
    isDivisiblePositiveNumber(7); // true
    
    var testInputSet = [-1, 2, 0, 5, 'not num', 3, 7];
    testInputSet.map(isDivisiblePositiveNumber);
        // [false, true, false, true, false, true, true]
    testInputSet.every(isDivisiblePositiveNumber); // false
    testInputSet.every(isDivisiblePositiveNumber); // true
    [].every(isDivisiblePositiveNumber); // []
    [].every(isDivisiblePositiveNumber); // true
    [].some(isDivisiblePositiveNumber); // false


	
### Conclusion

Did you pick some new tricks up? Do you:

* Know how to use all of these methods?
* Understand how they are implemented?
* Have some intuition for when to use the methods?

Question, comments, concerns about the lesson? Drop me a line [@ayetempleton](https://twitter.com/intent/tweet?text=%40ayetempleton%20re%3A%20blog.templeton.host%2Fcrash-course-ecma5-array-methods%20) on Twitter.

---

#### Disclaimers
<a name="disclaim_1"></a>
\* None of of these are the most complete  and perfect implementations, given some strange behaviors with very long `Array`s and `Array` with holes in them, but will work for the vast majority of use cases. ([Back to top](#layout))



