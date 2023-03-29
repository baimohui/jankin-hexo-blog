---
title: Date 对象
categories: 
- JavaScript
tags:
- JavaScript
- Date

---

JavaScript Date objects represent a single moment in time in a platform-independent format. Date objects contain a Number that represents milliseconds since 1 January 1970 UTC.

<!-- more -->

### Constructor

#### `new Date()`
When called as a constructor, returns a new Date object.

#### `Date()`
When called as a function, returns a string representation of the current date and time, exactly as `new Date().toString()` does.
```js
Date() // 'Sun Dec 11 2022 00:38:05 GMT+0800 (中国标准时间)'
new Date() // Sun Dec 11 2022 00:38:22 GMT+0800 (中国标准时间)
```

### Static Methods
#### `Date.now()`
Returns the numeric value corresponding to the current time—the number of milliseconds elapsed since January 1, 1970 00:00:00 UTC, with leap seconds ignored.
```js
Date.now() // 1670745980856
```

#### `Date.parse()`
Parses a string representation of a date and returns the number of milliseconds since 1 January, 1970, 00:00:00 UTC, with leap seconds ignored.

```js
Date.parse('Sun Dec 11 2022 00:38:05 GMT+0800 (中国标准时间)') // 1670690285000
```
Note: Parsing of strings with Date.parse is strongly discouraged due to browser differences and inconsistencies.

### Instance Methods
#### `Date.prototype.getDate()`
Returns the day of the month (1–31) for the specified date according to local time.
```js
new Date().getDate() // 11
```

#### `Date.prototype.getDay()`
Returns the day of the week (0–6) for the specified date according to local time.
```js
new Date().getDay() // 0
```

#### `Date.prototype.getFullYear()`
Returns the year (4 digits for 4-digit years) of the specified date according to local time.
```js
new Date().getFullYear() // 2022
```

#### `Date.prototype.getHours()`
Returns the hour (0–23) in the specified date according to local time.
```js
new Date().getHours() // 16
```

#### `Date.prototype.getMilliseconds()`
Returns the milliseconds (0–999) in the specified date according to local time.

#### `Date.prototype.getMinutes()`
Returns the minutes (0–59) in the specified date according to local time.

#### `Date.prototype.getMonth()`
Returns the month (0–11) in the specified date according to local time.

#### `Date.prototype.getSeconds()`
Returns the seconds (0–59) in the specified date according to local time.

#### `Date.prototype.getTime()`
Returns the numeric value of the specified date as the number of milliseconds since January 1, 1970, 00:00:00 UTC. (Negative values are returned for prior times.)
```js
new Date().getTime() // 1670746028587
```

#### `Date.prototype.getTimezoneOffset()`
Returns the time-zone offset in minutes for the current locale.
```js
new Date().getTimezoneOffset() // -480
```

#### `Date.prototype.getUTCDate()`
Returns the day (date) of the month (1–31) in the specified date according to universal time.

#### `Date.prototype.getUTCDay()`
Returns the day of the week (0–6) in the specified date according to universal time.

#### `Date.prototype.getUTCFullYear()`
Returns the year (4 digits for 4-digit years) in the specified date according to universal time.

#### `Date.prototype.getUTCHours()`
Returns the hours (0–23) in the specified date according to universal time.

#### `Date.prototype.getUTCMilliseconds()`
Returns the milliseconds (0–999) in the specified date according to universal time.

#### `Date.prototype.getUTCMinutes()`
Returns the minutes (0–59) in the specified date according to universal time.

#### `Date.prototype.getUTCMonth()`
Returns the month (0–11) in the specified date according to universal time.

#### `Date.prototype.getUTCSeconds()`
Returns the seconds (0–59) in the specified date according to universal time.