
# Ітеративні об’єкти

*Ітеративні* об’єкти є узагальненням масивів. Це концепція, яка дозволяє нам зробити будь-який об’єкт придатним для використання в циклі `for..of`.

Звичайно, по масивам можна ітеруватися. Але є багато інших вбудованих об’єктів, які також можна ітерувати. Наприклад, рядки також можна ітерувати.

Якщо об’єкт технічно не є масивом, а представляє колекцію (list, set) чогось, то `for..of` - чудовий синтаксис для його обходу, тому давайте подивимося, як змусити його працювати.


## Symbol.iterator

Ми можемо легко зрозуміти концепцію ітеративних об’єктіва, зробивши її власноруч.

Наприклад, у нас є об’єкт, який не є масивом, але виглядає придатним для `for..of`.

Як, наприклад, об’єкт `range`, який представляє інтервал чисел:

```js
let range = {
  from: 1,
  to: 5
};

// Ми хочемо, щоб for..of працював:
// for(let num of range) ... num=1,2,3,4,5
```

Щоб зробити об’єкт `range` ітерабельним (і таким чином дозволити `for..of` працювати), нам потрібно додати метод до об’єкта з назвою `Symbol.iterator` (спеціальний вбудований символ саме для цього).

1. Коли `for..of` запускається, він викликає цей метод один раз (або викликає помилку, якщо цей метод не знайдено). Метод повинен повернути *ітератор* -- об’єкт з методом `next`.
2. Далі `for..of` працює *лише з поверненим об’єктом*.
3. Коли `for..of` хоче отримати наступне значення, він викликає `next()` на цьому об’єкті.
4. Результат `next()` повинен мати вигляд `{done: Boolean, value: any}`, де `done=true` означає, що ітерація завершена, інакше `value` -- це наступне значення.

Ось повна реалізація об’єкту `range` із зауваженнями:

```js run
let range = {
  from: 1,
  to: 5
};

// 1. виклик for..of спочатку викликає цю функцію
range[Symbol.iterator] = function() {

  // ...вона повертає об’єкт ітератора:
  // 2. Далі, for..of працює тільки з цим ітератором, запитуючи у нього наступні значення
  return {
    current: this.from,
    last: this.to,      

    // 3. next() викликається на кожній ітерації циклом for..of
    next() {
      // 4. він повинен повертати значення як об’єкт {done:.., value :...}
      if (this.current <= this.last) {
        return { done: false, value: this.current++ };
      } else {
        return { done: true };
      }
    }
  };
};

// тепер це працює!
for (let num of range) {
  alert(num); // 1, потім 2, 3, 4, 5
}
```

Будь ласка, зверніть увагу на основну особливість ітеративних об’єктів: розділення проблем.

- Сам `range` не має методу `next()`.
- Натомість інший об’єкт, так званий "ітератор", створюється за допомогою виклику `range[Symbol.iterator]()`, а його `next()` генерує значення для ітерації.

Отже, об’єкт, що ітерує відокремлений від об’єкта, який він ітерує.

Технічно, ми можемо об’єднати їх і використовувати `range` в якості ітератора, щоб зробити код більш простим.

Подібно до цього:

```js run
let range = {
  from: 1,
  to: 5,

  [Symbol.iterator]() {
    this.current = this.from;
    return this;
  },

  next() {
    if (this.current <= this.to) {
      return { done: false, value: this.current++ };
    } else {
      return { done: true };
    }
  }
};

for (let num of range) {
  alert(num); // 1, тоді 2, 3, 4, 5
}
```

Тепер `range[Symbol.iterator]()` повертає сам об'єкт `range`: він має необхідний `next()` метод і пам'ятає поточну ітерацію прогресу в `this.current`. Коротше? Так. А іноді це також добре.

Недоліком є те, що тепер неможливо мати два `for..of` цикли паралельно для проходження через об’єкт: вони будуть ділити ітераційний стан, тому що є тільки один ітератор -- сам об'єкт. Але два параллельних for-of це рідкісний випадок, навіть у асинхронізованих сценаріях.

```smart header="Infinite iterators"
Також можливі нескінченні ітератори. Наприклад, `range` стає нескінченним для `range.to = Infinity`. Або ми можемо зробити ітерований об'єкт, який генерує нескінченну послідовність псевдорандомних чисел. Це також може бути корисним.
Немає обмежень на `next`, він може повертати все більше і більше значень, це нормально.

Звичайно, `for..of` цикли через такий об’єкт буде безкінечним. Але ми завжди можемо зупинити його за допомогою `break`.
```


## Рядок є ітерованим

Масиви та рядки найбільш широко використовуються вбудовані ітератори.

Для рядка, `for..of` цикл проходить по символам:

```js run
for (let char of "test") {
  // викликається 4 рази: один раз для кожного символу
  alert( char ); // t, потім e, потім s, потім t
}
```

І це правильно працює з сурогатними парами!

```js run
let str = '𝒳😂';
for (let char of str) {
    alert( char ); // 𝒳, і потім 😂
}
```

## Виклик ітератора явно

Для більш глибокого розуміння, давайте подивимося, як явно використовувати ітератор.

Ми будемо ітерувати рядок точно так само, як для `for..of`, але з прямими викликами. Цей код створює ітератор рядка і отримує значення від нього "вручну":

```js run
let str = "Привіт";

// робить те ж саме, як
// for (let char of str) alert(char);

*!*
let iterator = str[Symbol.iterator]();
*/!*

while (true) {
  let result = iterator.next();
  if (result.done) break;
  alert(result.value); // виводить символи один за одним
}
```

Це рідко потрібно, але дає нам більше контролю над процесом, ніж `для ..of`. Наприклад, ми можемо розділити процес ітерації: трохи ітерувати, а потім зупинитися, зробити щось інше, а потім відновити пізніше.

## Ітеровані об’єкти та псевдомасиви [#array-like]

Ці два офіційних терміни виглядають подібними, але дуже різні. Будь ласка, переконайтеся, що ви добре розумієте їх, щоб уникнути плутанини.

- *Ітеровані* -- це об'єкти, які реалізують метод `Symbol.iterator`, як описано вище.
- *Псевдомасиви* -- це об'єкти, які мають індекси та `length`, тому вони виглядають як масиви.

Коли ми використовуємо JavaScript для практичних завдань у браузері або будь-якому іншому середовищі, ми можемо зустріти об'єкти, які є ітерованими або масивами, або обома.
Наприклад, рядки є ітерованими об’єктами (`for..of` працює на них) та псевдомасивами (у них є числові індекси та `length`).

Але ітерований об’єкт може не бути масивом. І навпаки, псевдомасив може бути не ітерованим об’єктом.

Наприклад, `range` у прикладі вище є ітерованим об’єктом, але не масивом, тому що він не має індексованих властивостей та `length`.

І ось об'єкт, який є псевдомасивом, але не ітерованим об’єктом:

```js run
let arrayLike = { // має індекси та length => псевдомасив
  0: "Hello",
  1: "World",
  length: 2
};

*!*
// Error (немає Symbol.iterator)
for (let item of arrayLike) {}
*/!*
```

Обидва, ітерований об’єкт та псевдомасив, як правило є *не масивами*, вони не мають `push`,` pop` та ін. Це досить незручно, якщо у нас є такий об’єкт і ми хочемо працювати з ним як з масивом. НаприкладМи хотіли б працювати з `angy` за допомогою методів масиву. Як цього досягти?

## Array.from

There's a universal method [Array.from](mdn:js/Array/from) that takes an iterable or array-like value and makes a "real" `Array` from it. Then we can call array methods on it.

For instance:

```js run
let arrayLike = {
  0: "Hello",
  1: "World",
  length: 2
};

*!*
let arr = Array.from(arrayLike); // (*)
*/!*
alert(arr.pop()); // World (method works)
```

`Array.from` at the line `(*)` takes the object, examines it for being an iterable or array-like, then makes a new array and copies all items to it.

The same happens for an iterable:

```js
// assuming that range is taken from the example above
let arr = Array.from(range);
alert(arr); // 1,2,3,4,5 (array toString conversion works)
```

The full syntax for `Array.from` also allows us to provide an optional "mapping" function:
```js
Array.from(obj[, mapFn, thisArg])
```

The optional second argument `mapFn` can be a function that will be applied to each element before adding it to the array, and `thisArg` allows us to set `this` for it.

For instance:

```js
// assuming that range is taken from the example above

// square each number
let arr = Array.from(range, num => num * num);

alert(arr); // 1,4,9,16,25
```

Here we use `Array.from` to turn a string into an array of characters:

```js run
let str = '𝒳😂';

// splits str into array of characters
let chars = Array.from(str);

alert(chars[0]); // 𝒳
alert(chars[1]); // 😂
alert(chars.length); // 2
```

Unlike `str.split`, it relies on the iterable nature of the string and so, just like `for..of`, correctly works with surrogate pairs.

Technically here it does the same as:

```js run
let str = '𝒳😂';

let chars = []; // Array.from internally does the same loop
for (let char of str) {
  chars.push(char);
}

alert(chars);
```

...But it is shorter.    

We can even build surrogate-aware `slice` on it:

```js run
function slice(str, start, end) {
  return Array.from(str).slice(start, end).join('');
}

let str = '𝒳😂𩷶';

alert( slice(str, 1, 3) ); // 😂𩷶

// the native method does not support surrogate pairs
alert( str.slice(1, 3) ); // garbage (two pieces from different surrogate pairs)
```


## Summary

Objects that can be used in `for..of` are called *iterable*.

- Technically, iterables must implement the method named `Symbol.iterator`.
    - The result of `obj[Symbol.iterator]()` is called an *iterator*. It handles further iteration process.
    - An iterator must have the method named `next()` that returns an object `{done: Boolean, value: any}`, here `done:true` denotes the end of the iteration process, otherwise the `value` is the next value.
- The `Symbol.iterator` method is called automatically by `for..of`, but we also can do it directly.
- Built-in iterables like strings or arrays, also implement `Symbol.iterator`.
- String iterator knows about surrogate pairs.


Objects that have indexed properties and `length` are called *array-like*. Such objects may also have other properties and methods, but lack the built-in methods of arrays.

If we look inside the specification -- we'll see that most built-in methods assume that they work with iterables or array-likes instead of "real" arrays, because that's more abstract.

`Array.from(obj[, mapFn, thisArg])` makes a real `Array` from an iterable or array-like `obj`, and we can then use array methods on it. The optional arguments `mapFn` and `thisArg` allow us to apply a function to each item.
