# "Numeric values with precision" proposal

**Champions**: Jesse Alama (Igalia)
**Authors**: Jesse Alama (Igalia), Nicolò Ribaudo (Igalia)

**Stage**: Not presented yet to TC39

## Use cases and goals

While usually numeric values are interpreted as points on a number line, so that 1.2 and 1.20 represent the same entity, this is not always the case. One example where this matters is when measuring physical quantities, where 1.20 might indicate a much more precise measurement than 1.2. Another example is when displaying values to users: when displaying "1 _something_" you use singular terms, while "1.0 _something_" requires a plural.

The [Decimal](https://github.com/tc39/proposal-decimal) proposal could be originally used to solve this problem, as it used to expose the number of fractional difits of a Decimal value. However, this feature has been removed by the proposal because:
- it would prevent from ever introducing a primitive `"decimal"` type in the future for which `Decimal` is the wrapper object, because `1.0 === 1.00` would mean that the two values are not observably different;
- `Decimal` automatically propagates the precision of operands across arithmetic operations, according to IEEE 754 semantics. However, in many cases the way precision propagates is not the expected one, and it would be better to require users that need precision tracking to think about how they want it to be propagates.

This proposal aims at providing a representation of a numeric value together with how precise it is. This representation would not support any arithmetic operations and automatic propagation of precision: instead, developers have to work with magnitude and precision separately and then combine them as needed.

This proposal is not necessarily specific to `Decimal`: the most common numeric type is `Number`, and at least as long as `Decimal` is object-based `Number` will remain the most common numeric type. For this reason, the scope of the proposal includes exploring solutions that can also apply to `Number`.

## Description

An initial version of this proposal was originally designed as part of the `Decimal` proposal. It defines a `Decimal.prototype.withFractionalDigits(digits: number)` method that, when called, would return an object with two properties (⚠️ their names are very much up for bikeshedding):
- `number`: the underlying `Decimal` object;
- `precision`: the number of fractional digits.

This object would have its prototype set to `%WithFractionalDigitsPrototype%`, which provides the following methods:
- `valueOf()`: calls `.valueOf()` on the underlying object (for `Decimal` this throws, but for `Number` it would return the number as a primitive);
- `toString()`: calls `.toFixed()` on the underlying numeric value, passing the appropriate number of digits;
- `toLocaleString()`: calls `.toLocaleString()` on the underlying numeric, using the underlying precision as the default value for the `minimumFractionalDigits`/`maximumFractionalDigits` options.

The various `Intl` utilities that work with numbers would recognize these objects, and always use the underlying precision as a fallback.

This "numeric value with precision" concept could be extended to be a protocol based on a well-known symbol. This would allow libraries to provide their own classes that wrap either a `Decimal` or a `Number`, and that support different ways of propagating precision through arithmetic operations.

## Open questions

**Fractional or significant digits?**

The current proposal uses fractional digits, but it could be changed to support significant digits as well. For some use cases (Intl, and accounting) fractional digits are better, but for others (measurements) significant digits are more appropriate.

Assuming that the proposal supports negative numbers of fractional digits (for example, `10` with precision `-1` means that the number is `10` but the last digit is not significant), the two approaches are equivalent. The conversion between one and the other is:
- _significantDigits_ = _fractionalDigits_ + ceil(log10(abs(_number_))).
- _fractionalDigits_ = _significantDigits_ - ceil(log10(abs(_number_))).;

**Names!**

- What should this class be called? Is `WithFractionalDigits` a good name?
- What should the property to get back the underlying numeric object be called? `number`, `numericObject`, `value`, `magnitude`?
- What should the property to get the precision be called? `precision`, `fractionalDigits` (or `significantDigits`)?

**Should this be supported on BigInts?**

BigInts do not have fractional digits, but the concept of precision also applies to them.
