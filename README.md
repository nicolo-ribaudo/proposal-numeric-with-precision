# "Numeric values with precision" proposal

**Champions**: Jesse Alama (Igalia)

**Authors**: Jesse Alama (Igalia), Nicolò Ribaudo (Igalia)

**Stage**: Not presented yet to TC39

## Use cases and goals

While usually numeric values are interpreted as points on a number line, so that 1.2 and 1.20 represent the same entity, this is not always the case. Some examples where this matters are:
- when representing physical measurements, "1.2 meaters" and "1.20 meaters" include informtion about the resolution of the measurement tool, which can affect how you use that value and how you combine it with others
- when displaying numeric values, wether they have trailing zeroes or not can affect how the word that they are describing is pluralized (e.g. _"1 star"_ vs _"1.0 stars"_). When using Intl, you are currently required to pass the same precision data to multiple constructors/functions to make sure that they they behave coherently, and don't result in, for example _"1.0 star"_.
- when interacting with external numeric systems that do preserve/expose precision (for examples, complete implementations of IEEE 754's decimal128 type), you may want to be able to round-trip values as they are, without changing the meaning that the external system might give to them.

The [Decimal](https://github.com/tc39/proposal-decimal) proposal could be used at some point to solve this problem, as it used to expose the number of fractional difits of a Decimal value. However, this feature has been removed by the proposal because:
- it would prevent from ever introducing a primitive `"decimal"` type in the future for which `Decimal` is the wrapper object, because `1.0 === 1.00` would mean that the two values are not observably different;
- `Decimal` automatically propagates the precision of operands across arithmetic operations, according to IEEE 754 semantics. However, in many cases the way precision propagates is not the expected one, and it would be better to require users that need precision tracking to think about how they want it to propagate.

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

There would probably also be `Number.parseWithFractionalDigits` and `Decimal.parseWithFractionalDigits` methods that parse a string and return a numeric value together with the number of fractional digits (i.e. `Number.parseWithFractionalDigits("1.20")` would return an object with `number` set to `1.2` and `precision` set to `2`).

## Examples

<details open>
<summary>Computing an avergage rating for a resturant, formatting it with one fractional digit, and showing the result:</summary>

```js
const ratings = [2, 0, 0, 1, 3, 0];

const average = ratings.reduce((a, b) => a + b, 0) / ratings.length;
const averageWithPrecision = average.withFractionalDigits(1);

const pr = new Intl.PluralRules('en');
const plurals = { one: 'star', other: 'stars' };

console.log(`The restaurant has ${averageWithPrecision.toLocaleString("en")} ${plurals[pr.select(averageWithPrecision)]}`);
```

</details>

<details open>
<summary>Propagating precision through arithmetic operations acording to IEEE-754:</summary>

```js
const IEEE_754 = {
  add(a, b) {
    return a.number.add(b.number).withFractionalDigits(Math.min(a.precision, b.precision));
  },
  sub(a, b) {
    return a.number.sub(b.number).withFractionalDigits(Math.min(a.precision, b.precision));
  },
  multiply(a, b) {
    return a.number.mul(b.number).withFractionalDigits(a.precision + b.precision);
  },
  divide(a, b) {
    return a.number.div(b.number).withFractionalDigits(a.precision - b.precision);
  },
};

IEEE_754.add(
  new Decimal("1.2").withFractionalDigits(3),
  new Decimal("0.03").withFractionalDigits(5)
).toString(); // "1.230"
```

</details>

<details>
<summary>Propagating precision through arithmetic operations according to confidence intervals, and rounding the resulting interval radius to a power of 10:</summary>

```js
function computeCapacitorVoltage(charges, capacitance) {
  // formula: ΔV = ∑qᵢ / C

  const totalCharge = charges.reduce((tot, q) => tot + q.number, 0);
  const chargeError = charges.reduce((err, q) => err + 10 ** -q.precision, 0);

  const capacitanceError = 10 ** -capacitance.precision;

  const voltage = totalCharge / capacitance.number;
  const voltageError = (chargeError + capacitanceError * voltage) / capacitance.number;

  const fractionalDigits = Math.floor(-Math.log10(voltageError));
  return voltage.withFractionalDigits(fractionalDigits);
}
```

</details>

<details>
<summary>Using a custom class that wraps a <code>Decimal</code> and propagates error, implementing the "numeric with precision" protocol:</summary>

```js
class DecimalWithError {
  constructor(value, error) {
    this.#value = decimal;
    this.#error = error;
  }

  toString() {
    return `${this.#value} ± ${this.#error}`;
  }

  add(other) {
    return new DecimalWithError(
      this.#value.add(other.#value),
      this.#error.add(other.#error)
    );
  }

  subtract(other) {
    return new DecimalWithError(
      this.#value.subtract(other.#value),
      this.#error.add(other.#error)
    );
  }

  multiply(other) {
    return new DecimalWithError(
      this.#value.multiply(other.#value),
      this.#error.multiply(other.#value).add(other.#error.multiply(this.#value))
    );
  }

  divide(other) {
    const result = this.#value.divide(other.#value);
    return new DecimalWithError(
      result,
      other.#error.multiply(result).add(this.#error).divide(other.#error)
    );
  }

  scale(factor) {
    return new DecimalWithError(
      this.#value.multiply(factor),
      this.#error.multiply(factor)
    );
  }

  [Symbol.withFractionalDigit]() {
    const fractionalDigits = Math.floor(-Math.log10(Number(this.#error)));
    return this.#value.withFractionalDigits(fractionalDigits);
  }
}

function computeCapacitorVoltage(charges, capacitance) {
  // formula: ΔV = ∑qᵢ / C

  const totalCharge = charges.reduce((a, b) => a.add(b));
  return totalCharge.divide(capacitance);
}

const voltage = computeCapacitorVoltage(
  [
    new DecimalWithError(new Decimal("1.2"), new Decimal("0.001")),
    new DecimalWithError(new Decimal("1.8"), new Decimal("0.001")),
    new DecimalWithError(new Decimal("0.3"), new Decimal("0.0005"))
  ],
  new DecimalWithError(new Decimal("0.035"), new Decimal("0.0002"))
);
const voltage1000 = voltage.scale(-1000);

console.log(voltage1000.toString()); // "0.09428571428571428571428571428571428 ± 0.0006102040816326530612244897959183673"
console.log(voltage1000[Symbol.withFractionalDigit]().precision); // 3
console.log(voltage1000[Symbol.withFractionalDigit]().number); // Decimal { 0.094 }
console.log(new Intl.NumberFormat("en").format(voltage) + " kV"); // "0.094 kV"
```

</details>

## Open questions

**Fractional or significant digits?**

The current proposal uses fractional digits, but it could be changed to support significant digits as well. For some use cases (Intl, and accounting) fractional digits are better, but for others significant digits may be more appropriate.

Assuming that the proposal supports negative numbers of fractional digits (for example, `10` with precision `-1` means that the number is `10` but the last digit is not significant), the two approaches are equivalent. The conversion between one and the other is:
- _significantDigits_ = _fractionalDigits_ + ceil(log10(abs(_number_))).
- _fractionalDigits_ = _significantDigits_ - ceil(log10(abs(_number_))).;

**Names!**

- What should this class be called? Is `WithFractionalDigits` a good name?
- What should the property to get back the underlying numeric object be called? `number`, `numericObject`, `value`, `magnitude`?
- What should the property to get the precision be called? `precision`, `fractionalDigits` (or `significantDigits`)?

**Should this be supported on BigInts?**

BigInts do not have fractional digits, but the concept of precision also applies to them.

**Relationship with the "smart units" proposal**

There is a proposal under development that defines an object that holds a number together with a measurment unit (such as `2 kg`, or `3 inches`). The proposals could be combined to represent "a number with extra properties", or they could be kept separate and composable.
