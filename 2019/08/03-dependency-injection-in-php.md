# Dependency Injection In PHP #

Managing class dependencies is important part of object-oriented programming. 

This post describes why class dependencies are important, what the [Dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) is and what are the benefits of using it for managing class dependencies.

The follow-up article will describe shortcomings of dependency injection and it will provide alternative approach.

Contents:

{{ toc }}

## Dependencies ##

Simply put, class A is **dependent** on class B if its code uses properties and methods of class B. Class B instance is called a **dependency** of class A instance.

Example:

* `Customers` class manages customer records in the database and queries them as needed.
* `Db` class provides methods for running queries on database connection.    
* `Customers` class uses methods of `Db` for running queries.  

In this example, `Customers` class is dependent upon `Db` class:

![Dependency](dependency.png)

Sometimes, `Customers` class is also called a **client** of `Db` class and `Db` class is called a **service** of `Customers` class.

Some dependencies are **mandatory** (without which class can't function properly), others are **optional**.

## Less Dependencies Is Better ##

Classes with less dependencies are:

* **More maintainable**. In order to understand class inner workings, you have to know what its dependencies do, so minimizing number of dependencies makes class code easier to read.  
* **More reusable**. Reusing a class in another project requires all its dependencies working properly in that project as well, so with less dependencies less preparation is needed. 

Optimizing class dependencies involves finding a balance between:

* defining smaller classes doing one thing well as suggested by [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) and, this way, having less dependencies in each class.
* avoiding unnecessary complexity of having too many classes.

## Dependency Injection ##

Dependency injection is a common approach for managing class dependencies. 

On one hand, dependency injection introduces a discipline of writing classes in a certain way: 

* mandatory dependencies are listed as parameters in class constructor
* optional dependencies are defined as setter methods

This way, class **requests its dependencies** via syntax of its constructor and setter methods.

On the other hand, dependency injection provides means of configuring how dependencies of each class are **resolved**, that is, which object instances are actually passed as constructor or setter arguments.

### Requesting Dependencies ###

In the example below, `Db` class is mandatory dependency of `Customers` class and `Logger` class is optional dependency. Comments describe how to read source code:

	class Customers
	{
	    /**
	     * @var Db
	     */
	    protected $db;

	    /**
	     * @var Logger
	     */
	    protected $logger;
	
		// if you see Db in constructor parameters, you can make 
		// a conclusion that Db is mandatory dependency 
	    public function __construct(Db $db) {
	        $this->db = $db;
	    }
	
		// if you see a setter with Logger class, you can make a
		// conclusion that Logger is optional dependency
		public function setLogger(Logger $logger) {
			$this->logger = $logger;
		}

	    public function insert($firstName, $lastName) {
			// when using optional dependencies, it should be checked
			// for possible null value
			if ($this->logger) {
				$this->logger->log(...);
			}

			// on the contrary, using mandatory dependency is 
			// straightforward
	        $this->db->query(...);
	        return $this->db->getLastInsertedId();
	    }
	}  

It is responsibility of caller code (in project source or in unit tests) to provide dependencies, either as constructor argument or by calling setter method.

### Avoiding Hidden Dependencies ###

Avoid hiding dependencies in method bodies as it requires searching for dependencies in the whole class source file instead of some small part of it. It would also make class hard to test.

Below is the example of hidden dependency. `getDb()` function returns an instance of `Db` class. `Customers` class uses `Db::query()` method, so it is dependent on `Db` class. However, in order to understand that, the reader will have to scan all the method bodies until she finds this dependency: 

	class Customers
	{
	    public function insert($firstName, $lastName) {
	        getDb()->query(...);
	        return getDb()->getLastInsertedId();
	    }
	}     

### Dependency Injection Container ###

Instantiating lots of classes and providing dependencies to them can require significant effort. This task is a lot easier with dependency injection (DI) container. Most frameworks have DI container included, in smaller project, you may use standalone DI container, such as [Pimple](https://pimple.symfony.com/).

Before instantiating classes, create the container and configure how each class should be instantiated:

	use Pimple\Container;
	
	$container = new Container();
	
	// if instance of Db class is requested, just call the constructor.
	// Remember created object, so that on subsequent Db requests, 
	// just return that instance
	$container[Db::class] = function () {
	    return new Db();
	};
	
	// if instance of Customers class is requested, request Db 
	// instance first and pass it as an argument to Customers constructor.
	// Remember created object and return it on subsequent requests. 
	$container[Customers::class] = function (Container $container) {
	    return new Customers($container[Db::class]);
	};

After DI container is configured, you can create and use class instances easily:

	// requesting instance of Customers class internally uses callback 
	// function specified in configuration to create the instance.
  
	/* @var Customers $customers */
	$customers = $container[Customers::class];

	$customers->insert('John', 'Doe');
	$customers->insert('Jane', 'Doe');

### Running The Example ###

You can run and tinker this example code on your machine by running these shell commands:

	git clone https://github.com/osmianski/tests.git
	cd tests
	composer update
	php tests/injected_dependencies.php

## The Benefits Of Dependency Injection ##

Writing classes as described above has several important benefits. Such classes are:

1. framework-agnostic; 
2. easier to maintain; 
3. testable. 

### Framework-Agnostic Classes ###

As you can see from the example, class definition has no knowledge about DI container. 

Such classes are framework-agnostic. It means that you can use such classes in any project, under any framework.   

The only thing you will need to do is configuring class dependencies using DI container of that framework.

For example, in Laravel project you would need to configure class dependencies in `register()` method of service provider class:

	<?php
	
	namespace App\Providers;
	
	use Illuminate\Support\ServiceProvider;
	
	class YourServiceProvider extends ServiceProvider
	{
	    public function register()
	    {
	        $this->app->singleton(Db::class, function () {
	            return new Db();
	        });

	        $this->app->singleton(Customers::class, function ($app) {
	            return new Customers($app[Db::class]);
	        });
	    }
	} 

### Maintainable Dependencies ###

In classes written as described above, dependencies are easy to detect - just read definition of constructor and setters and:

* treat all references you see in constructor argument list as mandatory dependencies;
* treat every setter as optional dependency.

### Testable Classes ###

Even if you don't test a class with unit tests, it makes sense to write it in a way that it is easy to test later.

In unit tests, you may need to create class instance in isolation and customize its dependencies, for instance, to inject [mock objects](https://phpunit.readthedocs.io/en/8.3/test-doubles.html#mock-objects).  

By writing classes in dependency injection friendly way (as described above), you also make the classes easier to test. 

Below is an example of passing a mock of `Db` class to the unit test of `Customers` class (code use [PHPUnit](https://phpunit.de/) testing framework): 

	public function testInsert() {
		// create mock object
		$db = $this->getMockBuilder(Db::class)
             ->setMethods(['query', 'getLastInsertedId'])
             ->getMock(); 

		// configure mock object expectations which would force
		// this test to fail if Db maethos are not called
		$db->expects($this->once())
			->method('query');
		$db->expects($this->once())
			->method('getLastInsertedId');

		// inject mock object
		$customers = new Customers($db);

		// run code being tested
		$customers->insert('John', 'Doe');
	}

## What's Next ##

To sum up:

* managing class dependencies is important;
* the less dependencies, the better;
* hidden dependencies are bad;
* class testability is important;
* DI provides a solid approach for managing class dependencies and writing testable code.

The next article will describe shortcomings of dependency injection and it will provide alternative approach.

Discuss this post [on Hacker News]() or [on GitHub](https://github.com/osmianski/blog).    