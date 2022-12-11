---
title: Math 常用方法
categories: 
- JavaScript
tags:
- JavaScript
- Math
---

Math is a built-in object that has properties and methods for mathematical constants and functions. It's not a function object.

Math works with the Number type. It doesn't work with BigInt.

### Description
Unlike many other global objects, Math is not a constructor. All properties and methods of Math are static. You refer to the constant pi as Math.PI and you call the sine function as Math.sin(x), where x is the method's argument. Constants are defined with the full precision of real numbers in JavaScript.

### Static Properties
#### `Math.PI`
Ratio of a circle's circumference to its diameter; approximately 3.14159.

### Static Methods
#### `Math.abs(x)`
Returns the absolute value of x.
```js
Math.abs(-0.99) // 0.99
```

#### `Math.ceil(x)`
Returns the smallest integer greater than or equal to x.
```js
Math.ceil(0.1) // 1
```

#### `Math.floor(x)`
Returns the largest integer less than or equal to x.
```js
Math.floor(0.9) // 0
```

#### `Math.round(x)`
Returns the value of the number x rounded to the nearest integer.
```JS
Math.round(0.49) // 0
Math.round(0.5) // 1
```

#### `Math.max([x[, y[, …]]])`
Returns the largest of zero or more numbers.
```js
Math.max(1, 2, 3, 3.15, Math.PI) // 3.15
Math.max(1, 2, NaN) // NaN
Math.max() // -Infinity
```

#### `Math.min([x[, y[, …]]])`
Returns the smallest of zero or more numbers.
```js
Math.min(0, -1) // -1
Math.min(0, -1, NaN) // NaN
Math.min() // Infinity
```

#### `Math.pow(x, y)`
Returns base x to the exponent power y (that is, x^y).
```js
Math.pow(2, 3) // 8
```

#### `Math.random()`
Returns a pseudo-random number between 0 and 1.

```js
// Returning a random integer between two bounds
function random(min, max) {
  const num = Math.floor(Math.random() * (max - min + 1)) + min;
  return num;
}

random(1, 10); // 7
```
