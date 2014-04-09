# brian talks about angular with lots of data

This is my lightning talk about performance considerations for
Angular apps displaying lots of data.

It starts with a simplified explanation of `$digest` and then gives a few
examples of how to apply this knowledge.


## Measure things

Seriously you probably want to measure to make sure you know what's slow.


## What factors are important?

* [DOM manipulation][] (causes [reflows][], etc)
* The number of "dynamic things" your user sees.

Angular is pretty good at helping you at the first thing.
In some cases you can write your own directive


## Counting Dynamic Things

Consider the following chunk of an app:

```html
<div>
  {{thing}}

  <div ng-repeat="item in items">{{item.name}}</div>
</div>
```

Here's a list of the things Angular thinks might update:

```javascript
// angular figures this out based on directives, which add `$watch`s
var thingsThatMightUpdate = [
  $scope.thing,

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

*(This is all psuedocode btw)*


## Digest cycle

This is a simplification.
[The docs on scopes have the full story](http://docs.angularjs.org/guide/scope).

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

tl;dr – watch fewer things


## Make it faster

Here's a few strats.

### Paginate

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


### You know the data is populated once

* [bind-once (coming to 1.3 soon)](https://github.com/angular/angular.js/issues/5408)

[this article](http://blog.scalyr.com/2013/10/31/angularjs-1200ms-to-35ms/)

### You know the data is populated occasionally

Use event listeners.

```javascript
lol
```

In your directive:
```javascript
//
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


## Credits

data via [data.sfgov.org][]


## License
MIT


[reflows]: https://developers.google.com/speed/articles/reflow
[DOM manipulation]: https://developers.google.com/speed/articles/javascript-dom
[data.sfgov.org]: https://data.sfgov.org/Arts-Culture-and-Recreation-/Film-Locations-in-San-Francisco/yitu-d5am
