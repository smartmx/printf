# Standalone printf/sprintf formatted printing function library

This is a small but **fully-loaded** implementation of C's formatted printing family of functions. It was originally designed by Marco Paland, with the primary use case being in embedded systems - where these functions are unavailable, or when one needs to avoid the memory footprint of linking against a full-fledged libc. The library can be made even smaller by partially excluding some of the supported format specifiers during compilation. The library stands alone, with **No external dependencies**.

### CMake options and preprocessor definitions

Options used both in CMake and in the library source code via a preprocessor define:

| Option name                            | Default | Description  |
|----------------------------------------|---------|--------------|
| PRINTF_ALIAS_STANDARD_FUNCTION_NAMES   | NO      |  Alias the standard library function names (`printf()`, `sprintf()` etc.) to the library's functions.<br>**Note:** If you build the library with this option turned on, you must also have written<br>`#define PRINTF_ALIAS_STANDARD_FUNCTION_NAMES 1`<br>before including the `printf.h` header. |
| PRINTF_INTEGER_BUFFER_SIZE             | 32      |  ntoa (integer) conversion buffer size. This must be big enough to hold one converted numeric number _including_ leading zeros, normally 32 is a sufficient value. Created on the stack. |
| PRINTF_DECIMAL_BUFFER_SIZE             | 32      |  ftoa (float) conversion buffer size. This must be big enough to hold one converted float number _including_ leading zeros, normally 32 is a sufficient value. Created on the stack. |
| PRINTF_DEFAULT_FLOAT_PRECISION         | 6       |  Define the default floating point precision|
| PRINTF_MAX_INTEGRAL_DIGITS_FOR_DECIMAL | 9       |  Maximum number of integral-part digits of a floating-point value for which printing with %f uses decimal (non-exponential) notation |
| PRINTF_SUPPORT_DECIMAL_SPECIFIERS      | YES     |  Support decimal notation floating-point conversion specifiers (%f, %F) |
| PRINTF_SUPPORT_EXPONENTIAL_SPECIFIERS  | YES     |  Support exponential floating point format conversion specifiers (%e, %E, %g, %G) |
| SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS  | YES     |  Support the 'I' + bit size integer specifiers (%I8, %I16, %I32, %I64) as in Microsoft Visual C++ |
| PRINTF_SUPPORT_WRITEBACK_SPECIFIER     | YES     |  Support the length write-back specifier (%n) |
| PRINTF_SUPPORT_LONG_LONG               | YES     |  Support long long integral types (allows for the ll length modifier and affects %p) |

Within CMake, these options lack the `PRINTF_` prefix.

CMake-only options:

| Option name                            | Default | Description  |
|----------------------------------------|---------|--------------|
| PRINTF_BUILD_STATIC_LIBRARY            | NO      |  Build a library out of a shared object (dynamically linked at load time) rather than a static one (baked into the executables you build) |

Source-only options:

| Option name                            | Default | Description  |
|----------------------------------------|---------|--------------|
| PRINTF_INCLUDE_CONFIG_H                | NO      |  Triggers inclusing by `printf.c` of a "printf_config.h" file, which in turn contains the values of all of the CMake-and-preprocessor options above. A CMake build of the library uses this mechanism to apply the user's choice of options, so it can't have the mechanism itself as an option. |

Note: The preprocessor definitions are taken into account when compiling `printf.c`, _not_ when using the compiled library by including `printf.h`.

## Library API

### Implemented functions

The library offers the following, with the same signatures as in the standard C library (plus an extra underscore):
```
int printf_(const char* format, ...);
int sprintf_(char* s, const char* format, ...);
int vsprintf_(char* s, const char* format, va_list arg);
int snprintf_(char* s, size_t n, const char* format, ...);
int vsnprintf_(char* s, size_t n, const char* format, va_list arg);
int vprintf_(const char* format, va_list arg);
```
Note that `printf()` and `vprintf()`  don't actually write anything on their own: In addition to their parameters, you must provide them with a lower-level `putchar_()` function which they can call for actual printing. This is part of this library's independence: It is isolated from dealing with console/serial output, files etc.

Two additional functions are provided beyond those available in the standard library:
```
int fctprintf(void (*out)(char c, void* extra_arg), void* extra_arg, const char* format, ...);
int vfctprintf(void (*out)(char c, void* extra_arg), void* extra_arg, const char* format, va_list arg);
```
These higher-order functions allow for better flexibility of use: You can decide to do different things with the individual output characters: Encode them, compress them, filter them, append them to a buffer or a file, or just discard them. This is achieved by you passing a pointer to your own state information - through `(v)fctprintf()` and all the way to your own `out()` function.

#### "... but I don't like the underscore-suffix names :-("

You can [configure](#CMake-options-and-preprocessor-definitions) the library to alias the standard library's names, in which case it exposes `printf()`, `sprintf()`, `vsprintf()` and so on.

If you alias the standard library function names, *be careful of GCC/clang's `printf()` optimizations!*: GCC and clang recognize patterns such as `printf("%s", str)` or `printf("%c", ch)`, and perform a "strength reduction" of sorts by invoking `puts(stdout, str)` or `putchar(ch)`. If you enable the `PRINTF_ALIAS_STANDARD_FUNCTION_NAMES` option (see below), and do not ensure your code is compiled with the `-fno-builtin-printf` option - you might inadvertantly pull in the standard library implementation - either succeeding and depending on it, or failing with a linker error. When using `printf` as a CMake imported target, that should already be arranged for, but again: Double-check.

<br>

Alternatively, you can write short wrappers with your preferred names. This is completely trivial with the v-functions, e.g.:
```
int my_vprintf(const char* format, va_list va)
{
  return vprintf_(format, va);
}
```
and is still pretty straightforward with the variable-number-of-arguments functions:
```
int my_sprintf(char* buffer, const char* format, ...)
{
  va_list va;
  va_start(va, format);
  const int ret = vsprintf_(buffer, format, va);
  va_end(va);
  return ret;
}
```

### Supported Format Specifiers

A format specifier follows this prototype: `%[flags][width][.precision][length]type`
The following format specifiers are supported:

#### Types

| Type       | Output                   |
|------------|--------------------------|
| `d` or `i` | Signed decimal integer   |
| `u`        | Unsigned decimal integer	|
| `b`        | Unsigned binary          |
| `o`        | Unsigned octal           |
| `x`        | Unsigned hexadecimal integer (lowercase) |
| `X`        | Unsigned hexadecimal integer (uppercase) |
| `f` or `F` | Decimal floating point   |
| `e` or `E` | Scientific-notation (exponential) floating point |
| `g` or `G` | Scientific or decimal floating point |
| `c`        | Single character         |
| `s`        | String of characters     |
| `p`        | Pointer address          |
| `n`        | None; number of characters produced so far written to argument pointer |

Notes:

* The `%a` specifier for hexadecimal floating-point notation (introduced in C99 and C++11) is _not_ currently supported.
* If you want to print the percent sign (`%`, US-ASCII character 37), use "%%" in your format string.
* The C standard library's `printf()`-style functions don't accept `float` arguments, only `double`'s; that is true for this library as well. `float`'s get converted to `double`'s.

#### Flags

| Flags   | Description |
|---------|-------------|
| `-`     | Left-justify within the given field width; Right justification is the default. |
| `+`     | Forces to precede the result with a plus or minus sign (+ or -) even for positive numbers.<br>By default, only negative numbers are preceded with a - sign. |
| (space) | If no sign is going to be written, a blank space is inserted before the value. |
| `#`     | Used with o, b, x or X specifiers the value is preceded with 0, 0b, 0x or 0X respectively for values different than zero.<br>Used with f, F it forces the written output to contain a decimal point even if no more digits follow. By default, if no digits follow, no decimal point is written. |
| `0`     | Left-pads the number with zeros (0) instead of spaces when padding is specified (see width sub-specifier). |


#### Width Specifiers

| Width    | Description |
|----------|-------------|
| (number) | Minimum number of characters to be printed. If the value to be printed is shorter than this number, the result is padded with blank spaces. The value is not truncated even if the result is larger. |
| `*`      | The width is not specified in the format string, but as an additional integer value argument preceding the argument that has to be formatted. |


#### Precision Specifiers

| Precision   | Description |
|-------------|-------------|
| `.`(number) | For integer specifiers (d, i, o, u, x, X): precision specifies the minimum number of digits to be written. If the value to be written is shorter than this number, the result is padded with leading zeros. The value is not truncated even if the result is longer. A precision of 0 means that no character is written for the value 0.<br>For f and F specifiers: this is the number of digits to be printed after the decimal point. **By default, this is 6, and a maximum is defined when building the library**.<br>For s: this is the maximum number of characters to be printed. By default all characters are printed until the ending null character is encountered.<br>If the period is specified without an explicit value for precision, 0 is assumed. |
| `.*`        | The precision is not specified in the format string, but as an additional integer value argument preceding the argument that has to be formatted. |


#### Length modifiers

The length sub-specifier modifies the length of the data type.

| Length   | With `d`, `i`               | With `u`,`o`,`x`, `X`    | Support enabled by...                 |
|----------|-----------------------------|--------------------------|---------------------------------------|
| (none)   | `int`                       | `unsigned int`           |                                       |
| `hh`     | `signed char`               | `unsigned char`          |                                       |
| `h`      | `short int`                 | `unsigned short int`     |                                       |
| `l`      | `long int`                  | `unsigned long int`      |                                       |
| `ll`     | `long long int`             | `unsigned long long int` | PRINTF_SUPPORT_LONG_LONG              |
| `j`      | `intmax_t`                  | `uintmax_t`              |                                       |
| `z`      | signed version of `size_t`  | `size_t`                 |                                       |
| `t`      | `ptrdiff_t`                 | `ptrdiff_t`              |                                       |
| `I8`     | `int8_t`                    | `uint8_t`                | SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS |
| `I16`    | `int16_t`                   | `uint16_t`               | SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS |
| `I32`    | `int32_t`                   | `uint32_t`               | SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS |
| `I64`    | `int64_t`                   | `uint64_t`               | SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS |


Notes:

* The `L` modifier, for `long double`, is not currently supported.
* A `"%zd"` or `"%zi"` takes a signed integer of the same size as `size_t`. 
* The implementation currently assumes each of `intmax_t`, signed `size_t`, and `ptrdiff_t` has the same size as `long int` or as `long long int`. If this is not the case for your platform, please open an issue.
* The `Ixx` length modifiers are not in the C (nor C++) standard, but are somewhat popular, as it makes it easier to handle integer types of specific size. One must specify the argument size in bits immediately after the `I`. The printing is "integer-promotion-safe", i.e. the fact that an `int8_t` may actually be passed in promoted into a larger `int` will not prevent it from being printed using its original value.

### Return Value

Upon successful return, all functions return the number of characters written, _excluding_ the terminating NUL character used to end the string.
Functions `snprintf()` and `vsnprintf()` don't write more than `count` bytes, including the terminating NUL character ('\0').
Anyway, if the output was truncated due to this limit, the return value is the number of characters that _could_ have been written.
Notice that a value equal or larger than `count` indicates a truncation. Only when the returned value is non-negative and less than `count`,
the string has been completely written with a terminating NUL.
If any error is encountered, `-1` is returned.

If `NULL` is passed for the `buffer` parameter, nothing is written, but the formatted length is returned. For example:
```C
int length = sprintf(NULL, "Hello, world"); // length is set to 12
```

## License

This library is published under the terms of the [MIT license](http://www.opensource.org/licenses/MIT).

