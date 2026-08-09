# Alterant Script
This is a scripting language specifically designed to be used in the VST synth [Spektral](https://shop.kaixo.me/product/Spektral). More specifically, this script can be used to create custom 'Alterants'. To read more about what alterants are, check [Spektral's user manual](https://shop.kaixo.me/assets/synths/Spektral/Spektral%20%E2%80%93%20User%20Manual.pdf).

## Examples
Check the examples folder in this repository for some example scripts. This folder contains all of the built-in alterants in Spektral, recreated in alterant script.

# How Does It Work
Alterants let you transform spectral data in some way. This scripting language allows you to do that efficiently. You can apply transformation on entire buffers in a single instruction. Here is a short example program that shows off some of the functionality:
```
@name Utility
@inplace true
@graph eq base = 0.5
@param 0 name="Gain"  default=1.0 transform=range[0,100]    format=percent
@param 1 name="Width" default=1.0 transform=range[0,100]    format=percent
@param 2 name="Pan"   default=0.5 transform=range[-100,100] format=pan 

gain  = param[0] * graph
width = param[1]
pan   = 2 * stereo(cos(param[2] * 0.5 * pi),
                   sin(param[2] * 0.5 * pi)) / sqrt2

mid = input.left + input.right
side = input.left - input.right

output = stereo(mid + side * width,
                mid - side * width) * pan * gain
```
Let's start at the top, here we define some configuration: we set our script name, tell it that the algorithm we are writing can be performed in-place, tell it what the default graph should be on initialization, and we give it information how many parameters we want, and how to display them in the UI.

After configuration we read and transform our parameters. Let's first look at the line `gain = param[0] * graph`. Here we set the `gain` variable to the first parameter multiplied by `graph`. `graph` is a global that's always available, and it represents the graph data. This global is a stereo buffer, and multiplying it with the scalar `param[0]` multiplies every value in the stereo buffer with `param[0]`, resulting in a new stereo buffer. When assigning `pan` we use the function `stereo()`, this function takes 1 or 2 mono parameters, and turns it into a stereo value. This works for both mono scalar values, and mono buffers. After reading the parameters, we access `input` to get the mid and side channels. `input` is another global that is always available, and contains the spectral data that is fed into this alterant. Just like `graph`, this is a stereo buffer, which allows us to read `.left` and `.right` buffers. Doing `input.left + input.right` performs addition on all elements of the buffer at once, in a single instruction. Finally, we combine `mid` and `side` again using `stereo()` and our `width` parameter, and then we multiply the resulting stereo buffer with `pan` and `gain`.

## Operations
As seen in the example, it is possible to do operations between different types. That behaviour is defined like this:

| Type A        | Type B        | Result Type     | Behaviour | 
| ------------- | ------------- | --------------- | --------- |
| scalar        | scalar        | scalar          | `a + b` |
| scalar        | stereo        | stereo          | `{ a + b.left, a + b.right }` |
| scalar        | buffer        | buffer          | `[a + b[0], ...]` |
| scalar        | stereo buffer | stereo buffer   | `[{ a + b[0].left, a + b[0].right }, ...]` |
| stereo        | stereo        | stereo          | `{ a.left + b.left, a.right + b.right }` |
| stereo        | buffer        | stereo buffer   | `[{ a.left + b[0], a.right + b[0] }, ...]` |
| stereo        | stereo buffer | stereo buffer   | `[{ a.left + b[0].left, a.right + b[0].right }, ...]` |
| buffer        | buffer        | buffer          | `[a[0] + b[0], a[1] + b[1], ...]` |
| buffer        | stereo buffer | stereo buffer   | `[{ a[0] + b[0].left, a[0] + b[0].right }, ...]` |
| stereo buffer | stereo buffer | stereo buffer   | `[{ a[0].left + b[0].left, a[0].right + b[0].right }, ...]` |

## Operators
This is a list of all supported operators. 
All of these support all types as arguments, unless otherwise specified. 
Mixing different types behaves the same as defined above.

All bit manipulation operators only work with integers. 
If the variables contain floating point numbers, 
they will be truncated to an integer before the bit operation.

Note how operators like `+=` or `++` are not supported!

| Operator   | Behaviour                                               | Operator            | Behaviour                                                 |
| ---------- | ------------------------------------------------------- | ------------------- | --------------------------------------------------------- |
| `a + b`    | Addition.                                               | `a > b `            | Greater than.                                             |
| `a - b`    | Subtraction.                                            | `a >= b`            | Greater than or equals.                                   |
| `a * b`    | Multiplication.                                         | `a == b`            | Equals.                                                   |
| `a / b`    | Division.                                               | `a != b`            | Not equals.                                               |
| `a % b`    | Modulo, this works with floating point numbers as well. | `+a`                | Plus. This has no effect.                                 |
| `a << b`   | Bitwise shift left.                                     | `-a`                | Minus.                                                    |
| `a >> b`   | Bitwise shift right.                                    | `!a`                | Logic not.                                                |
| `a && b`   | Logic and.                                              | `~a`                | Bitwise not.                                              |
| `a \|\| b` | Logic or.                                               | `a ? b : c`         | Ternary conditional.                                      |
| `a & b`    | Bitwise and and.                                        | `a = b`             | Assignment.                                               |
| `a \| b`   | Bitwise and or.                                         | `a()`               | Function call.                                            |
| `a ^ b`    | Bitwise and xor.                                        | `a[]`               | Indexing.                                                 |
| `a < b`    | Less than.                                              | `a.left`, `a.right` | Accessing left and right channels of a stereo value.      |
| `a <= b`   | Less than or equals.                                    | `a, b`              | Comma operator. Returns the last expression in the list.  |

## Globals
Each script has access to some globals. Here is that list of globals:

| Variable      | What does it mean |
| -----------   | ----------------- |
| i             | Mono buffer containing indices: `[0, 1, 2, 3, 4, ..., 511]`.  |
| size          | The size of a buffer. This is always equal to 512.  |
| sample_rate   | The rate at which this script is evaluated. Note that this is not equal to the actual sample rate of the audio, as alterants are not evaluated every sample on lower quality settings. |
| frequency     | The frequency in Hz of the voice that this alterant is influencing. |
| param         | The list of up to 5 parameter values.  |
| trigger       | Boolean that is only true once when the voice that this alterant influences has been triggered. Can be used to initialize or reset values. |
| deterministic | Boolean, corresponds with the 'fixed' toggle in the UI.    |
| wrap          | Boolean, corresponds with the 'wrap' toggle in the UI.     |
| stereo        | Boolean, corresponds with the 'stereo' toggle in the UI.   |
| input         | Stereo buffer containing the input spectral data. |
| graph         | Stereo buffer containing the graph.               |
| output        | Stereo buffer to write the result to.             |
| bipolar       | Boolean that is true if the data coming into the alterant is bipolar. If any bipolar table feeds data into this alterant, it is set to true. |
| key_tracked   | Boolean that is true if the data coming into the alterant is key tracked. If all tables that feed into this alterant are key tracked, it is set to true. |
| phase_data    | Boolean that is true if the data coming into the alterant is phase data. If all tables that feed into this alterant are phase data, it is set to true. |

Note that these are treated as keywords, you cannot define any variable with these names.

## Constants
Besides globals, there are also some constants: `e`, `log2e`, `log10e`, `pi`, `two_pi`, `inv_pi`, `inv_two_pi`, `inv_sqrtpi`, `ln2`, `ln10`, `sqrt2`, `sqrt3`, `inv_sqrt3`, `egamma`, `phi`. 

Note that these are treated as keywords, you cannot define any variable with these names.

## Functions
There are also some built-in functions. Custom functions are not supported. 
All of these functions support all types as arguments, unless otherwise specified. 
Mixing different types behaves the same as defined above.

| Function         | What does it do |
| ---------------- | --------------- |
| `noise()`        | Generate a buffer of random values between `0` and `1`. |
| `noise(a)`       | Generate a buffer of random values between `0` and `a`. `a` must be a constant. |
| `noise(a, b)`    | Generate a buffer of random values between `a` and `b`. `a` and `b` must be a constant. |
| `random()`       | Generate a random value between `0` and `1`. |
| `random(a)`      | Generate a random value between `0` and `a`. `a` must be a constant. |
| `random(a, b)`   | Generate a random value between `a` and `b`. `a` and `b` must be a constant. |
| `stereo(a)`      | Construct a stereo value from mono value `a`: `{ a, a }`. Works for both mono scalar and mono buffer values. |
| `stereo(a, b)`   | Construct a stereo value from mono values `a` and `b`: `{ a, b }`. Works for both mono scalar and mono buffer values. |
| `floor(a)`       | Rounds `a` towards negative infinity. |
| `trunc(a)`       | Rounds `a` towards `0`. |
| `ceil(a)`        | Rounds `a` towards positive infinity. |
| `round(a)`       | Rounds `a` towards the nearest integer. |
| `abs(a)  `       | Calculate the absolute value of `a`. |
| `sqrt(a)`        | Calculate the square root of `a`. |
| `sin(a)`         | Calculate sin of `a`. Uses a fast polynomial approximation. |
| `cos(a)`         | Calculate cos of `a`. Uses a fast polynomial approximation. |
| `log(a)`         | Calculate log of `a`. |
| `exp(a)`         | Calculate exp of `a`. |
| `pow(a, b)`      | Calculate `a` to the power `b`. |
| `min(a, b)`      | Returns the smallest between `a` and `b`. Equivalent to `a < b ? a : b`. |
| `max(a, b)`      | Returns the largest between `a` and `b`. Equivalent to `a < b ? b : a`. |  
| `clamp(a, b, c)` | Clamp `a` between `b` and `c`. Equivalent to `min(max(a, b), c)`. |
| `lerp(a, b, c)`  | Linearly interpolate between `b` and `c` using `a`. Equivalent to `b + a * (c - b)`. |

## Indexing
Another powerful feature is the ability to index into buffers using expressions. And it is possible to configure how to handle boundaries, and whether to do any interpolation when the index is not a whole number. 

There are 3 different boundary modes: `wrap`, `clamp` and `zero`. 
- `wrap` mode wraps the index around, so 512 becomes 0. Equivalent to `data[index & 0x1FF]`. 
- `clamp` simply clamps the index between 0 and 511. Equivalent to `data[min(max(index, 0), 511)]`.
- `zero` returns `0` when reading out of bounds. Equivalent to `index >= 0 && index < 512 ? data[index] : 0`

There are 2 interpolation modes: `none` and `linear`.
- `none` does no interpolation, this simply floors the floating point. Equivalent to `data[floor(index)]`.
- `linear` does linear interpolation. Equivalent to `lerp(index - floor(index), data[floor(index)], data[floor(index) + 1])`.

You can also set either one to `fast`, and it will use the fastest method, if you don't care about boundary mode. Right now, `fast` boundary is equivalent to `wrap`, and `fast` interpolation is `none`.

These indexing modes become incredibly powerful when combined with buffers, you can do things like:
```
output[i] = input[graph * size : linear wrap]
```

Note that `output[i]` is equivalent to just typing `output`.

It is also possible to write to scattered indices, however, this has some unexpected behaviour:
```
output[graph * size : linear wrap] = input[i]
```
This works to some degree, however, this is evaluated linearly, so if there are multiple writes to the same index, it will only maintain the latest value. 
This becomes even less intuitive when assigning to itself:
```
output[i] = 0
output[graph * size : linear wrap] = max(output[i], input[i])
```
You would expect this expression to keep the maximum value instead of the latest value. However, buffer operations are evaluated on the entire buffer at once,
so it is actually evaluating `max(output[i], input[i])` first on the whole buffer, before it starts assigning to `output[graph * size : linear wrap]`. So it is essentially equivalent to:
```
output[i] = 0
temp = max(output[i], input[i])
output[graph * size : linear wrap] = temp[i]
```

You can also configure a global boundary and interpolation mode using a config statement. If you do not specify a boundary or interpolation mode in the indexing operator, it will fall back to these. By default they are both set to `fast`.
```
@boundary clamp
@interpolation linear
```

## Conditionals
Like in any programming language, it is also possible to write an if-statement.

```
if (wrap)
  output = input[graph * size : wrap]
else {
  output = input[graph * size : clamp]
}
```

You can only use a mono scalar value as a condition, you cannot use a stereo or buffer value. It is, however, possible to use both stereo
and buffer values in ternary conditionals:

```
output = graph > 0.5 ? input : 0
```

## Loops
Since most operations happen on entire buffers, you generally do not need loops. However, you still can write for-loops and while-loops if
you need them. But keep in mind that these are extremely slow compared to simple buffer operations! But still, if you need something like a 
cummulative loop, you can still write them like this:

```
for (index = 0; index < size; index = index + 1) {
  output[index + 1] = output[index]
}
```

Or as a while-loop:
```
index = 0;
while (index < size) {
  output[index + 1] = output[index]
  index = index + 1
}
```

These loops also support `break` and `continue` statements.

## Persistent Data
For some algorithms you might need persistent data, like for an envelope follower for example.
You can add any number of persistent variables to the script by defining them using the `@data` configuration.
This persistent data will keep its value for as long as the voice that the alterant influence is active.
The data is fully reset to 0 when a voice is retriggered. You can use the `trigger` global to
initialize your persistent data to any value you want.

Here is an example:
```
@data stereo_buffer my_data
@data scalar counter

// Initialize the data on trigger
if (trigger) {
  my_data = noise()
}

if (counter == 100) {
  my_data = noise()
  counter = 0
}

output = input * my_data

// Update the data
delta_time = 1.0 / sample_rate
my_data = (my_data + delta_time) % 1
counter = counter + 1
```

## Configuration
### `@name`
The name that will be displayed in the UI.

Usage: `@name <string|identifier>`

Examples:
```
@name "A Name"
@name Word
```

### `@inplace`
Whether the algorithm works inplace, meaning it can use the same output buffer as the input buffer.
This is `true` by default.

Usage: `@inplace <true|false>`

Examples:
```
@inplace true
@inplace false
```

### `@graph`
Configure the default graph when initializing this alterant.

Usage: `@graph <line|eq|bar|none> [base=<number>] [points=<points>]`

Where `<points>` is 1 or more `[<number>, <number>]` or `[<number>, <number>, <number>]` separated by spaces.

Examples:
```
@graph none
@graph eq base = 0.5 points = [0, 0, 2]
@graph bar base = 0
@graph line points = [0, 0] [1, 1]
```

### `@param`
Configure parameters.

Usage: `@param <0|1|2|3|4> [name=<string|identifier>] [default=<number>] [transform=<transform>] [format=<format>]`

Where `<transform>` is one of:
- `default`
- `range[<number>,<number>]`
- `integer[<number>,<number>]`
- `power[<number>,<number>,<number>]`
- `bipolar_power[<number>,<number>,<number>]`
- `inverted_power[<number>,<number>,<number>]`
- `bipolar_inverted_power[<number>,<number>,<number>]`
- `decibels[<number>,<number>]`

Where `<format>` is one of: `default`, `integer`, `decibels`, `percent`, `frequency`, `pan`, `time`, `detune`, `multiplier`.

Examples:
```
@param 0 name="Mix" default=1 transform=range[0, 100] format=percent
@param 1 name=Gain transform=decibels[0, 1] format=decibels default=1
@param 2 name=Pan transform=range[-100, 100] default=0.5 format = pan
@param 3 name = "Attack" default = 0.0 transform = power[0, 10000, 2] format = time
@param 4 name = Scale default = 0.0 transform = range[1,10] format=multiplier
```

### `@data`
Add persistent state.

Usage `@data <scalar|stereo|buffer|stereo_buffer> <identifier>`

Examples:
```
@data stereo_buffer position
@data scalar counter
```

### `@boundary`
Set the default boundary handling.

Usage: `@boundary <fast|wrap|clamp|zero>`

Examples:
```
@boundary fast
@boundary wrap
```

### `@interpolation`
Set the default interpolation handling.

Usage: `@interpolation <fast|none|linear>`

Examples:
```
@interpolation none
@interpolation linear
```

