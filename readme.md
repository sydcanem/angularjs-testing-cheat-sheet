# AngularJS Testing Cheat Sheet

### Unit testing routes

#### URLProvider

Let's assume we have this routing setup:

```js
angular.module('myApp', ['ngRoute'])

.config(['$routeProvider', function($routeProvider) {
	$routeProvider
		.when('/', {
			'templateUrl' : 'templates/main.html',
			'controller' : 'HomeController'
		})
		.when('/login', {
			'templateUrl' : 'templates/login.html',
			'controller' : 'LoginController'
		})
		.otherwise({
			'redirectTo' : '/'
		});
}]);
```

In order to set up our unit tests so that they can test our routing code, we need to do a few things:

- Inject the **$route**, **$location** and **$rootScope** services
- Set up a mock back end to handle XHR fetching template code
- Set a location and run a $digest lifecycle

```js
describe('Routes test', function() {
	// Mock our module in our tests
	beforeEach(module('myApp'));

	// We want to store a copy of the three services we'll use in our tests
	// so we can later reference these services in our tests.
	var $location, $route, $rootScope;

	// We added _ in our dependencies to avoid conflicting with our variables.
	// Angularjs will strip out the _ and inject the dependencies.
	beforeEach(inject(function(_$location_, _$route_, _$rootScope_){
		$location = _$location_;
		$route = _$route_;
		$rootScope = _$rootScope_;
	}));

	// We need to setup a mock backend to handle the fetching of templates from the 'templateUrl'.
	beforeEach(inject(function($httpBackend){
		$httpBackend.expectGET('templates/main.html').respond(200, 'main HTML');
		// or we can use $templateCache service to store the template.
		// $routeProvider will search for the template in the $templateCache first
		// before fetching it using http
		// $templateCache.put('templates/main.html', 'main HTML');
	}));

	// Our test code is set up. We can now start writing the tests.

	// When a user navigates to the index page, they are shown the index page with the proper
	// controller
	it('should load the index page on successful load of /', function(){
		expect($location.path()).toBe('');

		$location.path('/');

		// The router works with the digest lifecycle, wherein after the location is set,
		// it takes a single digest loop cycle to process the route,
		// transform the page content, and finish the routing.
		// In order to trigger the location request, we’ll run a digest cycle (on the $rootScope) 
		// and check that the controller is as expected.
		$rootScope.$digest();

		expect($location.path()).toBe( '/' );
		expect($route.current.controller).toBe('HomeController');
	});

	it('should redirect to the index path on non-existent route', function(){
		expect($location.path()).toBe('');

		$location.path('/a/non-existent/route');

		$rootScope.$digest();

		expect($location.path()).toBe( '/' );
		expect($route.current.controller).toBe('HomeController');
	});
});
```

**_resolves_**

Resolves are a way of executing and finishing asynchronous tasks before a particular route is loaded.
This is a great way to check if user is logged in, has authorizations and permissions, and even pre-load
some data before a controller and route are loaded into the view. 

Consider this routing configuration where we add a resolve which returns a **promise**:

```js
angular.module('myApp', ['ngRoute'])

.config(['$routeProvider', function($routeProvider) {
	$routeProvider
		.when('/', {
			'templateUrl' : 'templates/main.html',
			'controller' : 'HomeController',
			'resolve' : {
				'allowAccess' : ['$http', function($http){
					return $http.get('/api/has_access');
				}]
			}
		}
		})
		.otherwise({
			'redirectTo' : '/'
		});
}]);
```

In our test, we mock out each resolves dependencies and spy on each method using jasmine.createSpy()
if you care if the method has been invoked.

```js
describe('Routes test with resolves', function() {
	var httpMock = {};

	beforeEach(module('myApp', function($provide){
		$provide('$http', httpMock);
	}));

	var $location, $route, $rootScope;

	beforeEach(inject(function(_$location_, _$route_, _$rootScope_, $httpBackend, $templateCache){
		$location = _$location_;
		$route = _$route_;
		$rootScope = _$rootScope_;

		$templateCache.put('templates/main.html', 'main HTML');

		httpMock.get = jasmine.createSpy('spy').and.returnValue('test');
	}));

	it('should load the index page on successful load of /', 
		inject(function($injector){
			expect($location.path()).toBe( '' );
			$location.path('/');

			$rootScope.$digest();

			expect($location.path()).toBe( '/' );
			expect($route.current.controller).toBe('HomeController');

			// We need to do $injector.invoke to resolve dependencies
			expect($injector.invoke($route.current.resolve.allowAccess)).toBe('test');
	}));
});
```