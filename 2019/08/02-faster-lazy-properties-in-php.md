# Faster "Lazy" Properties In PHP #

"Lazy" property is a common technique of calculating property value only when (and if) it is actually needed.

This article describes [a technique](#faster-approach) improving performance of lazy properties 4-5 times.

Contents:

{{ toc }}

## Lazy Property Getters Are Not Free ##

Typically, property getter method is used to implement "lazy" behavior:

	<?php
	
	namespace Osmianski\Tests;
	
	class LazyProperties
	{
	    protected $foo;
	
	    public function getFoo() {
	        if ($this->foo === null) {
	            $this->foo = 'foo';
	        }
	
	        return $this->foo;
	    }
	}

However, there is a cost. Let's measure how much time does it take access such a property:

	$object = new LazyProperties();
	$count = 10000;
	
	$startedAt = microtime(true);
	for ($i = 0; $i < $count; $i++) {
	    $value = $object->getFoo();
	}
	$elapsed = sprintf("%.2f", (microtime(true) - $startedAt) * 1000);
	echo "'foo' property accessed {$count} times in {$elapsed}ms\n";

The result:

	'foo' property accessed 10000 times in 8.78ms

## Actually, Any Property Getter Is Not Free ##

What about trivial property getter?

    protected $bar = 'bar';

    public function getBar() {
        return $this->bar;
    }

The performance of calling `$object->getBar()` method is similar to `getFoo()`:

	'bar' property accessed 10000 times in 8.39ms

## Faster Approach ##

The alternative approach is to use ["magic" `__get()` method](https://www.php.net/manual/en/language.oop5.overloading.php#object.get):  

	/**
	 * @property $magic
	 */
	class LazyProperties
	{
	    public function __get($property) {
	        switch ($property) {
	            case 'magic': return $this->magic = 'magic';
	        }
	    }
	}    

Here is how it works:

* After object is just created, it has no `magic` property at all.
* On first access to `magic` property, since there is no requested property, `__get()` method is called. `__get()` method:
	* calculates the value;
	* creates the property;
	* assigns calculated value to it;
	* returns the value.
* On subsequent access to `magic` property, since there the property already exists, the value of the property is returned, without calling any method at all.

The performance of accessing `$object->magic` property is much faster:

	'magic' property accessed 10000 times in 1.47ms

## Performance Comparison ##

On Windows, PHP 7.2.17:

	'foo' property accessed 10000 times in 8.78ms
	'bar' property accessed 10000 times in 8.39ms
	'magic' property accessed 10000 times in 1.47ms

On the same computer, Linux virtual machine, PHP 7.3.8:

	'foo' property accessed 10000 times in 12.92ms
	'bar' property accessed 10000 times in 10.73ms
	'magic' property accessed 10000 times in 2.76ms

## Test It Yourself ##

Maybe, it is just my laptop? Measure the technique in the shell of your own development or production environment:

	git clone https://github.com/osmianski/tests.git
	cd tests
	composer update
	php tests/lazy_properties.php

## Instead Of Conclusion ##

Practical use of "magic" lazy properties is subject for discussion: 

1. Depending on number of avoided method calls, overall performance gains may be not worth the effort.

2. Main performance gain comes from avoiding method calls. It seems that reducing other unnecessary method (function, callback) calls, especially in loops, may give additional performance gains. 

3. "Magic" property is basically a public property and, hence, it increases the risk of unintended value overwrite in user code. 

	The team should agree on using such properties as read-only properties. In addition, an auditing tool reporting unintended assignments to such properties would be useful.  

	It is worth noting, however, that user code can access and assign your protected or private properties anyway:
	
		$property = new \ReflectionProperty($object, 'foo');
	    $property->setAccessible(true);
	    $property->setValue($object, 'unexpected'); 

Post your test results or discuss this post [on Hacker News](https://news.ycombinator.com/item?id=20703888) or [on GitHub](https://github.com/osmianski/blog).