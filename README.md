# brian talks about angular with lots of data

This is my lightning talk about performance considerations for
Angular apps displaying lots of data.

It starts with a simplified explanation of `$digest` and then gives a few
examples of how to apply this knowledge to cases when you're repeating over
a bunch of elements.


## Measure things

Make sure you know what's slow.
Lots of tools that can help.

* [Chrome devtools](https://developers.google.com/chrome-developer-tools/docs/timeline)
* [Firefox profiler](https://developer.mozilla.org/en-US/docs/Performance/Profiling_with_the_Built-in_Profiler)

I'm not going to talk about them.


## What factors are important?

What causes slowness?

### DOM manipulation

* Causes [reflows][]
* Lots of strategies [DOM manipulation][]

Angular is pretty good at helping you with this.
In some cases you can write your own directive to

### dynamic things

The number of "dynamic things" your user sees.
This is called the number of `$watch`s.


## Counting Dynamic Things

What adds a `$watch`?

Consider the following chunk of an app:

```html
<div>
  <p>{{thing}}</p>
  <p>{{getThing()}}</p>

  <div ng-repeat="item in items">{{item.name}}</div>
</div>
```

Here's a list of the things Angular `$watch`s for updates:

```javascript
// angular figures this out based on directives, which add `$watch`s
var thingsThatMightUpdate = [
  $scope.thing,
  $scope.getThing(), // invoke this each time

  // from ngRepeat
  $scope.items.length,
  $scope.items[0], // obj refs
  $scope.items[1],
  $scope.items[2],
  $scope.items[3],

  // from {{item.name}} inside ngRepeat
  $scope.items[0].name, // values
  $scope.items[1].name,
  $scope.items[2].name,
  $scope.items[3].name,
];
```


## Digest cycle

This is a simplification (cuz lightning talk).
[The documentation on scopes](http://docs.angularjs.org/guide/scope) have the full story.

```javascript
// map thingName to value
var oldValues = {
  '$scope.thing': 'hello',
  '$scope.items.length': 4,
  /* ... */
};

function digest () {
  var thingsThatChanged;
  do {
    thingsThatChanged = [];

    thingsThatMightUpdate.forEach(function (thing) {
      if (oldValue[thingName] !== $scope[thingName]) {
        thingsThatChanged.push(thingName);
        oldValue[thingName] = $scope[thingName];
      }
    });

    // trigger `$watch`s, updating the DOM, etc

  } while(thingsThatChanged.length > 0);
}
```

Note that we have to eval `$scope.getThing()` on each digest.
If `getThing()` is slow, then all your digests are slow.

If you have a bunch of things in the array, the digest will be slow.

tl;dr – watch fewer things, keep `$watch`s fast.


## How to reduce expensive watches

$$$$$$$$$$$$$$$$$$$$$

### Transform the data before binding

Rather than use filters/functions in your bindings,
transform the data before binding to it.

#### Before

Controller:
```javascript
angular.module('mySlowApp', []).controller('MyController', [
  '$scope', '$http', function ($scope, $http) {

    // this has like a million elts
    $http.get('all-the-things.json').success(function (data) {
      $scope.data = data;
    });

    $scope.transform = function (item) {
      // do something expensive
      return item;
    };
  }]);
```

HTML:
```html
<div ng-repeat="item in data">{{transform(item.name)}}</div>
```

#### After

Controller:
```javascript
angular.module('myFastApp', []).controller('MyController', [
  '$scope', '$http', function ($scope, $http) {

    // this has like a million elts
    $http.get('all-the-things.json').success(function (data) {
      $scope.data = data.map(transform);
    });

    function transform (item) {
      // do something expensive
      return item;
    };
  }]);
```

HTML:
```html
<div ng-repeat="item in data">{{item.name}}</div>
```


## How to watch fewer things

Here are a few strategies.

### Paginate

ngRepeat over a subset of the things.

Controller:

```javascript
angular.module('myApp', []).controller('MyController', [
  '$scope', function ($scope) {

    // this has like a million elts
    $scope.bigArray = [ /* ... */ ];

    $scope.page = 0;

    $scope.$watch('page', function (index) {
      $scope.littleArray = $scope.bigArray.slice(index, index + 5);
    });

  }]);
```

Template:
```html
<div>
  <div ng-repeat="item in littleArray">{{item.name}}</div>
  <button ng-click="prev -= 1">prev</button>
  <button ng-click="page += 1">next</button>
</div>
```

You can use a similar strategy but with [infinite][infinite scrolling]/[virtual][virtual scrolling]
scrolling.

### You know the data is populated once

~Coming to 1.3 Soon~

* [bind once (GH #5408)](https://github.com/angular/angular.js/issues/5408)

~3rd Party Modules~

* [angular-once](https://github.com/Pasvaz/bindonce)
* [bindonce](https://github.com/Pasvaz/bindonce)
* [this article](http://blog.scalyr.com/2013/10/31/angularjs-1200ms-to-35ms/)

### You know the data is populated occasionally

Opt out of data binding; use event listeners.

In your controller:
```javascript
angular.module('myApp', []).controller('MyController', [
  '$scope', '$interval', '$http', function ($scope, $interval, $http) {

    // this has like a million elts
    $http.get('all-the-things.json').success(function (data) {
      $scope.$broadcast('dataUpdated', data);
    });
  }]);
```

In your directive:
```javascript
{
  link: function (scope, elt) {
    scope.$on('dataUpdated', function (ev, data) {
      // do DOM manipulation
    });
  }
}
```


## Summary

Angular makes the 95% case fast, but gives you the tools to take control of your own performance
destiny when you want to.


## License
MIT


[reflows]: https://developers.google.com/speed/articles/reflow
[DOM manipulation]: https://developers.google.com/speed/articles/javascript-dom
[data.sfgov.org]: https://data.sfgov.org/Arts-Culture-and-Recreation-/Film-Locations-in-San-Francisco/yitu-d5am
[virtual scrolling]: https://github.com/stackfull/angular-virtual-scroll
[infinite scrolling]: http://binarymuse.github.io/ngInfiniteScroll/
