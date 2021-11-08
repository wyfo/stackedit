# Story of a contribution to CPython

As a reminder, CPython is 
> The canonical implementation of the Python programming language, as distributed on [python.org](https://www.python.org/).

Some days ago, I was reading the source code of the CPython [JSON standard library](https://docs.python.org/3/library/json.html), and I've noticed some parts of the implementation that could be optimized. I like this kind of code exercise, and you know, CPython is open source, so let's try to do our bit.

*I know that "premature optimization is the root of evil", but to extend a little bit the [quote](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.103.6084&rep=rep1&type=pdf):*
> We  _should_  forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil.
> Yet we should not pass up our opportunities in that critical 3%.

*So I take the opportunity. Moreover, as explained latter, the code I'm talking about is critical, therefore a good candidate to optimize.*

![](https://memegenerator.net/img/instances/42970028.jpg =400x)

## A quick dive into CPython code

CPython source code is available in [Github](https://github.com/python/cpython). We will only be interested in the code of the standard library, which is mainly located in the `Lib` directory — you can find the complete description in [CPython developer guide](https://devguide.python.org/setup/#directory-structure); for example, JSON module is in `Lib/json`. However, if you look at `Lib/json/encoder.py`, there is at the beginning of the file (code has been a little bit simplified):
```python
try:  
    from _json import encode_basestring as c_encode_basestring  
except ImportError:  
    c_encode_basestring = None  

def py_encode_basestring(s):
	...

encode_basestring = (c_encode_basestring or py_encode_basestring)
```
In fact, if modules of the standard library are often implemented in pure Python, some of them can also use an optimized implementation, written in C in the case of CPython, when it's available.
These C modules are located in `Modules` directory, `Modules/_json.c` in our case. 

In this post, I will only talk about JSON unicode string encoding. Actually, string encoding is maybe the most critical part of JSON serialization: quotes, backslashes and ASCII control characters must be escaped, so each string must be scanned entirely, and copied with escaped characters when needed. And there is lots of strings in a JSON document, if only the object properties.

That operation takes place in `escape_unicode` function (code is copied as is):
```c
static PyObject *
escape_unicode(PyObject *pystr)
{
    /* Take a PyUnicode pystr and return a new escaped PyUnicode */
    Py_ssize_t i;
    Py_ssize_t input_chars;
    Py_ssize_t output_size;
    Py_ssize_t chars;
    PyObject *rval;
    const void *input;
    int kind;
    Py_UCS4 maxchar;

    if (PyUnicode_READY(pystr) == -1)
        return NULL;

    maxchar = PyUnicode_MAX_CHAR_VALUE(pystr);
    input_chars = PyUnicode_GET_LENGTH(pystr);
    input = PyUnicode_DATA(pystr);
    kind = PyUnicode_KIND(pystr);

    /* Compute the output size */
    for (i = 0, output_size = 2; i < input_chars; i++) {
        Py_UCS4 c = PyUnicode_READ(kind, input, i);
        Py_ssize_t d;
        switch (c) {
        case '\\': case '"': case '\b': case '\f':
        case '\n': case '\r': case '\t':
            d = 2;
            break;
        default:
            if (c <= 0x1f)
                d = 6;
            else
                d = 1;
        }
        if (output_size > PY_SSIZE_T_MAX - d) {
            PyErr_SetString(PyExc_OverflowError, "string is too long to escape");
            return NULL;
        }
        output_size += d;
    }

    rval = PyUnicode_New(output_size, maxchar);
    if (rval == NULL)
        return NULL;

    kind = PyUnicode_KIND(rval);

#define ENCODE_OUTPUT do { \
        chars = 0; \
        output[chars++] = '"'; \
        for (i = 0; i < input_chars; i++) { \
            Py_UCS4 c = PyUnicode_READ(kind, input, i); \
            switch (c) { \
            case '\\': output[chars++] = '\\'; output[chars++] = c; break; \
            case '"':  output[chars++] = '\\'; output[chars++] = c; break; \
            case '\b': output[chars++] = '\\'; output[chars++] = 'b'; break; \
            case '\f': output[chars++] = '\\'; output[chars++] = 'f'; break; \
            case '\n': output[chars++] = '\\'; output[chars++] = 'n'; break; \
            case '\r': output[chars++] = '\\'; output[chars++] = 'r'; break; \
            case '\t': output[chars++] = '\\'; output[chars++] = 't'; break; \
            default: \
                if (c <= 0x1f) { \
                    output[chars++] = '\\'; \
                    output[chars++] = 'u'; \
                    output[chars++] = '0'; \
                    output[chars++] = '0'; \
                    output[chars++] = Py_hexdigits[(c >> 4) & 0xf]; \
                    output[chars++] = Py_hexdigits[(c     ) & 0xf]; \
                } else { \
                    output[chars++] = c; \
                } \
            } \
        } \
        output[chars++] = '"'; \
    } while (0)

    if (kind == PyUnicode_1BYTE_KIND) {
        Py_UCS1 *output = PyUnicode_1BYTE_DATA(rval);
        ENCODE_OUTPUT;
    } else if (kind == PyUnicode_2BYTE_KIND) {
        Py_UCS2 *output = PyUnicode_2BYTE_DATA(rval);
        ENCODE_OUTPUT;
    } else {
        Py_UCS4 *output = PyUnicode_4BYTE_DATA(rval);
        assert(kind == PyUnicode_4BYTE_KIND);
        ENCODE_OUTPUT;
    }
#undef ENCODE_OUTPUT

#ifdef Py_DEBUG
    assert(_PyUnicode_CheckConsistency(rval, 1));
#endif
    return rval;
}
```
I will assume that you're enough familiar with C code, but here are a few elements to understand CPython specific parts:

- Python objects are manipulated using `PyObject*`, a pointer to `PyObject` struct which is "inherited" by all other object structs like `PyTupleObject` or `PyUnicodeObject`; it means that the first field of these structs has the type `PyObject`. As a consequence `PyUnicodeObject*` can safely be casted to `PyObject*` — that's a way to implement single inheritance.
- Argument `pystr` is assumed to be an Unicode string — this is tested in the caller function — so functions like `PyUnicode_GET_LENGTH` can safely be called.
- There is 3 kinds of unicode strings, `PyUnicode_1BYTE_KIND`/`PyUnicode_2BYTE_KIND`/`PyUnicode_4BYTE_KIND`, which give the number of bytes on which is stored a character. Unicode strings characters are stored in an array whose pointer is obtained using `PyUnicode_DATA`.

As written at the beginning, by reading this particular function, I've noticed some ways the code could be optimized. Maybe you did too — you can also take a longer look at it if you want.

![](https://www.nicepng.com/png/full/310-3101250_photo-meme-wondering.png =400x)

*Disclaimer: I don't pretend to make the best optimization possible, and maybe you will notice the same things than me, or even better things to do. Again it's open source, feel free to improve.* 😀

## Switch case prevails!

This first optimization has nothing about algorithmic. The code use `switch`statements, but mix them with an `if` statement in the `default` case. It's indeed simpler to write, however, it can maybe not be as much optimized by the compiler as it would be with a full `switch`.

However, that may not be trivial at all. I've indeed done some experimentations to compare pure `switch` and `switch`/`if` mix, and I found case where the latter was better optimized. Compiler are very complex, and they can do pretty what they want with your code. You think your `switch` will be compiled into a [jump table](https://en.wikipedia.org/wiki/Branch_table) and your `if` in conditional jumps, but GCC can decide to do the opposite.

So we need to test. Here `c <= 0x1f` is the same than `c <= 31` — it tests if `c` is an ASCII control çharacter — that's a few lines of code to write all the cases, but performance improvement can be worth it. Also, there are two `switch`, so instead of writing all the cases twice, we will write a macro:
```c
#define CONTROL_CHAR \  
0: case 1: case 2: case 3: \  
case 4: case 5: case 6: case 7: \  
/* case 8: case 9: case 10: */ case 11: \  
/* case 12: case 13: */ case 14: case 15: \  
case 16: case 17: case 18: case 19: \  
case 20: case 21: case 22: case 23: \  
case 24: case 25: case 26: case 27: \  
case 28: case 29: case 30: case 31
```
(You will notice that I did not included neither the first `case`, nor the last `:`, so that it allows writing `case CONTROL_CHAR:` in the `switch` statement, but it's purely a matter of taste.)

Let's then rewrite the `switch` statements (here is the first one):
```c
    switch (c) {  
    case '\\': case '"': case '\b': case '\f':  
    case '\n': case '\r': case '\t':  
        d = 2;  
        break;  
    case CONTROL_CHAR:  
        d = 6;
        break;
    default:  
        d = 1;  
    }
```
and test it!

### How to test

The instruction for installing and running CPython are documented in the [CPython developer guide](https://devguide.python.org/setup/). However the default compilation command use `-O0` flag, which must not be used to test optimization. Code must in fact be compiled using `-O3` in order to measure the real impact that modification will have to CPython, as it is [released with `-O3` compilation](https://discuss.python.org/t/why-does-cpython-use-o3/1915).
By reading the Makefile, we see that optimization flag is set in `OPT` variable, so we can replace it by executing `make -j2 OPT=-O3`. Also, if there was a previous compilation with other flags, don't forget to execute `make clean`.

Once the code is compiled, we can execute it. However, no need to write benchmark, Python has already everything we need, and that's cool because our code is already wrapped to be used in Python.
For example, let's write something like this (on my macOS system, compiled CPython executable is `python.exe`):
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
500000 loops, best of 5: 486 nsec per loop
```

Now, let's see when compiled with the `switch` optimization:
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
1000000 loops, best of 5: 364 nsec per loop
```

![](https://media.makeameme.org/created/looks-at-those.jpg =400x)

*Disclaimer: Optimization is a complex and multifactorial subject. This result prove that on my computer architecture using my compiler version, this modification is an optimization, but that's about it. I'm still confident that this result can be generalized, and will have to test it on other platform. Also, developers which will review my PR will surely have cleverer opinion than mine about it.*

## Check for overflow only when it can happens

The loop computing the output size checks at each iteration if it overflows the its type capacity. This check is necessary, as it integer overflow would surely make the interpreter crash (you can also ask [Ariane 5](https://en.wikipedia.org/wiki/Integer_overflow#Examples) engineers about this kind of bug 🚀). 

However, the initial string size doesn't overflow, so overflow can only be caused by escapement. And escapement is bounded, 1 or 5 characters more. So, for a string to overflow, it would require at least `PY_SSIZE_T_MAX / 6` characters, all of them escaped with 5 additional characters. We can thus iterate safely on the first characters. 
In case of very long string having more than `(PY_SSIZE_T_MAX / 6) - 1` characters, we should iterate first iterate on this safe range of characters; it would give us the escapement real size, and allow us to compute the next safe range.

By the way, by removing the systematic overflow check, it removes also the need for the temporary `d` variable. Moreover, it's not necessary to count every characters, only escapement additional size is enough because it can then be added to the string length (for convenience, enclosing quotes are counted in the escaping size, which is then initailized at 2).

Here is how it can be implemented:
```c
    /* variable declaration to be put at the top with others */
    Py_ssize_t escapement_size;
    /* Compute the output size */  
    escapement_size = 2;  
  
#define COUNT_ESCAPEMENT(i) do { \  
    Py_UCS4 c = PyUnicode_READ(kind, input, i); \  
    switch (c) { \  
    case '\\': case '"': case '\b': case '\f': \  
    case '\n': case '\r': case '\t': \  
        escapement_size += 1; \  
        break; \  
    case CONTROL_CHAR: \  
        escapement_size += 5; \  
        break; \  
    } \  
} while (0)  
  
    i = 0;  
    while (1) {  
	    /* safe_bound is defined such as i + escapement_size + (safe_bound - i) * 6 <= PY_SSIZE_T_MAX */  
	    Py_ssize_t safe_bound = i + (PY_SSIZE_T_MAX - i - escapement_size) / 6;  
	    if (input_chars <= safe_bound) {  
	        for (; i < input_chars; i++) {  
	            COUNT_ESCAPEMENT(i);  
	        }  
	        break;  
	    } else if (safe_bound == i) {  
	        for (; i < input_chars; i++) {  
	            COUNT_ESCAPEMENT(i);  
	            if (i > PY_SSIZE_T_MAX - escapement_size) {  
	                PyErr_SetString(PyExc_OverflowError, "string is too long to escape");  
	                return NULL;  
	            }  
	        }  
	        break;  
	    } else {  
	        for (; i < safe_bound; i++) {  
	            COUNT_ESCAPEMENT(i);  
	        }  
	    }  
	}
#undef ENCODE_OUTPUT  
    output_size = input_chars + escapement_size;
```

Let's remind the benchmark before before:
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
1000000 loops, best of 5: 364 nsec per loop
```
And now:
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
1000000 loops, best of 5: 369 nsec per loop
```
![enter image description here](https://i.imgur.com/TNaoJqD.png =400x)
*Actually, difference may not be significative between this two results, but several retries after, the modification is indeed slowing the code.*

I would have expected at least a slice performance improvement, not the contrary.  Optimization is rarely a trivial thing, they say…
But the algorithmic optimization should have more incidence when the strings are bigger, let's try:
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = ''.join(map(str, range(100)))" "c_encode_basestring(s)"
200000 loops, best of 5: 1.66 usec per loop
```
vs.
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = ''.join(map(str, range(100)))" "c_encode_basestring(s)"
200000 loops, best of 5: 1.55 usec per loop
```
Indeed, bigger strings benefit of the optimization. Now, why are small strings so much impacted? Some investigation after, the culprit seems to be the division! Integer division is in fact a very slow operation (several dozens of CPU cycles). 

However, small strings are way more current, so there encoding should not be slowed down by an optimization targeting bigger ones. We could then add a first `if`, and replace the division by a bit-shift — it would be a division by 8 instead of 6, `PY_SSIZE_T_MAX` such a big number anyway. Talking about `PY_SSIZE_T_MAX`, we can even consider that `PY_SSIZE_T_MAX >> 3` (2^28^≈5.36E8 on a 32-bit architecture,  2^59^≈5.76E17 on 64-bits) is already bigger than most if not all the strings that will be encoded by this function.
The previous code could then be simplified:
```c
if (input_chars <= (PY_SSIZE_T_MAX >> 3)) {  
    for (i = 0; i < input_chars; i++) {  
        COUNT_ESCAPEMENT(i);  
    }  
} else {  
    for (i = 0; i < input_chars; i++) {  
        COUNT_ESCAPEMENT(i);  
        if (i > PY_SSIZE_T_MAX - escapement_size) {  
            PyErr_SetString(PyExc_OverflowError, "string is too long to escape");  
            return NULL;  
        }  
    }  
}
```
I did not even optimized the very (very) big string part, as I think it should never happen. It allows keeping the code simpler, I think CPython developers will prefer it that way, even if very very big strings would have benefit a lot of the optimization (but who have ever seen a JSON with a 100000000TB string inside?).
```bash
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
500000 loops, best of 5: 353 nsec per loop
```
Better now!
*Optimization is not really significative for small string, but we saw that it's more the case with bigger strings, and it's simple enough to keep it in my opinion.*

## To `memcpy` or not to `memcpy`

After the computation of the output size, a new string is initialized, and must then be filled with its quoted escaped content.  This is currently done in a `for` loop within the `ENCODE_OUTPUT` macro:
```c
	chars = 0;
	output[chars++] = '"';
	for (i = 0; i < input_chars; i++) {
		Py_UCS4 c = PyUnicode_READ(kind, input, i);
		switch (c) {
		/* omitted cases for concision */
		default:
			output[chars++] = c;
		}
	}
	output[chars++] = '"';
```
Actually, escaping is not the common case at all, and as written above, common case looks like a lot to a naive `memcpy` implementation. Python also have a [documented function](https://docs.python.org/3/c-api/unicode.html#c.PyUnicode_CopyCharacters) to copy a string into another: `PyUnicode_CopyCharacters` (which falls back on `memcpy`); I've also found `_PyUnicode_FastCopyCharacters` in CPython code, which seems to be better suited here.

So, we should use `_PyUnicode_FastCopyCharacters` for each range of consecutive not-escaped characters. We can for example store the indices of escaped characters in the first iteration, and use them in the second part. 

However, we don't know the number of escaped characters in advance. Using a dynamic array is out of question, because of the allocation cost. In that kind of problem, a common solution is to optimize the most common case (few escapements) using a static array with a small constant size like 8. The case where array size is not enough is handled in less optimized code.

Now it looks like:
```c
#define CONTROL_CHAR \  
0: case 1: case 2: case 3: \  
case 4: case 5: case 6: case 7: \  
/* case 8: case 9: case 10: */ case 11: \  
/* case 12: case 13: */ case 14: case 15: \  
case 16: case 17: case 18: case 19: \  
case 20: case 21: case 22: case 23: \  
case 24: case 25: case 26: case 27: \  
case 28: case 29: case 30: case 31  
  
#define MAX_ESCAPEMENTS_STORED 8  
  
static PyObject *  
escape_unicode(PyObject *pystr)  
{  
    /* Take a PyUnicode pystr and return a new escaped PyUnicode */  
  Py_ssize_t i;  
    Py_ssize_t input_chars;  
    Py_ssize_t escapement_size;  
    Py_ssize_t escapement_indices[MAX_ESCAPEMENTS_STORED];  
    Py_ssize_t escapement_count;  
    Py_ssize_t output_size;  
    Py_ssize_t chars;  
    PyObject *rval;  
    const void *input;  
    int kind;  
    Py_UCS4 maxchar;  
  
    if (PyUnicode_READY(pystr) == -1)  
        return NULL;  
  
    maxchar = PyUnicode_MAX_CHAR_VALUE(pystr);  
    input_chars = PyUnicode_GET_LENGTH(pystr);  
    input = PyUnicode_DATA(pystr);  
    kind = PyUnicode_KIND(pystr);  
  
    /* Compute the output size */  
  escapement_size = 2;  
    escapement_count = 0;  
  
#define COUNT_ESCAPEMENT(i) do { \  
    Py_UCS4 c = PyUnicode_READ(kind, input, i); \  
    switch (c) { \  
    case '\\': case '"': case '\b': case '\f': \  
    case '\n': case '\r': case '\t': \  
        escapement_size += 1; \  
        if (escapement_count < MAX_ESCAPEMENTS_STORED) { \  
            escapement_indices[escapement_count] = i; \  
        } \  
        escapement_count++; \  
        break; \  
    case CONTROL_CHAR: \  
        escapement_size += 5; \  
        if (escapement_count < MAX_ESCAPEMENTS_STORED) { \  
            escapement_indices[escapement_count] = i; \  
        } \  
        escapement_count++; \  
        break; \  
    } \  
} while (0)  
  
    if (input_chars <= (PY_SSIZE_T_MAX >> 3)) {  
        for (i = 0; i < input_chars; i++) {  
            COUNT_ESCAPEMENT(i);  
        }  
    } else {  
        for (i = 0; i < input_chars; i++) {  
            COUNT_ESCAPEMENT(i);  
            if (i > PY_SSIZE_T_MAX - escapement_size) {  
                PyErr_SetString(PyExc_OverflowError, "string is too long to escape");  
                return NULL;  
            }  
        }  
    }  
#undef ENCODE_OUTPUT  
    output_size = input_chars + escapement_size;  
  
    rval = PyUnicode_New(output_size, maxchar);  
    if (rval == NULL)  
        return NULL;  
  
    kind = PyUnicode_KIND(rval);  
#define ESCAPE_CHAR(chars, i) do { \  
    Py_UCS4 c = PyUnicode_READ(kind, input, i); \  
    switch (c) { \  
    case '\\': output[chars++] = '\\'; output[chars++] = c; break; \  
    case '"':  output[chars++] = '\\'; output[chars++] = c; break; \  
    case '\b': output[chars++] = '\\'; output[chars++] = 'b'; break; \  
    case '\f': output[chars++] = '\\'; output[chars++] = 'f'; break; \  
    case '\n': output[chars++] = '\\'; output[chars++] = 'n'; break; \  
    case '\r': output[chars++] = '\\'; output[chars++] = 'r'; break; \  
    case '\t': output[chars++] = '\\'; output[chars++] = 't'; break; \  
    case CONTROL_CHAR: \  
        output[chars++] = '\\'; \  
        output[chars++] = 'u'; \  
        output[chars++] = '0'; \  
        output[chars++] = '0'; \  
        output[chars++] = Py_hexdigits[(c >> 4) & 0xf]; \  
        output[chars++] = Py_hexdigits[(c     ) & 0xf]; \  
        break; \  
    default: \  
        output[chars++] = c; \  
    } \  
} while (0)  
  
#define ENCODE_OUTPUT do { \  
        output[0] = '"'; \  
        Py_ssize_t copy_from; \  
        Py_ssize_t escapement_bound = Py_MIN(escapement_count, MAX_ESCAPEMENTS_STORED); \  
        for (i = 0, copy_from = 0, chars = 1; i < escapement_bound; i++) { \  
            Py_ssize_t escapement_index = escapement_indices[i]; \  
            Py_ssize_t copy_size = escapement_index - copy_from; \  
            _PyUnicode_FastCopyCharacters(rval, chars, pystr, copy_from, copy_size); \  
            copy_from = escapement_index + 1; \  
            chars += copy_size; \  
            ESCAPE_CHAR(chars, escapement_index); \  
        } \  
        if (escapement_count <= MAX_ESCAPEMENTS_STORED) { \  
            _PyUnicode_FastCopyCharacters(rval, chars, pystr, copy_from, input_chars - copy_from); \  
            chars += input_chars - copy_from; \  
        } else { \  
            for (i = copy_from; i < input_chars; i++) { \  
                ESCAPE_CHAR(chars, i); \  
            } \  
        } \  
        output[chars] = '"'; \  
    } while (0)  
  
    if (kind == PyUnicode_1BYTE_KIND) {  
        Py_UCS1 *output = PyUnicode_1BYTE_DATA(rval);  
        ENCODE_OUTPUT;  
    } else if (kind == PyUnicode_2BYTE_KIND) {  
        Py_UCS2 *output = PyUnicode_2BYTE_DATA(rval);  
        ENCODE_OUTPUT;  
    } else {  
        Py_UCS4 *output = PyUnicode_4BYTE_DATA(rval);  
        assert(kind == PyUnicode_4BYTE_KIND);  
        ENCODE_OUTPUT;  
    }  
#undef ENCODE_OUTPUT  
  
#ifdef Py_DEBUG  
    assert(_PyUnicode_CheckConsistency(rval, 1));  
#endif  
 return rval;  
}
```
Let's check the performance:
Before:
```bash
# Without escaping
./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
1000000 loops, best of 5: 354 nsec per loop
# With escaping
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random \"string\"'" "c_encode_basestring(s)"
500000 loops, best of 5: 405 nsec per loop
# Bigger string
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = ''.join(map(str, range(100)))" "c_encode_basestring(s)"
200000 loops, best of 5: 1.54 usec per loop
```
After: 
```bash
# Without escaping
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random string'" "c_encode_basestring(s)"
1000000 loops, best of 5: 315 nsec per loop
# With escaping
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = 'some random \"string\"'" "c_encode_basestring(s)"
500000 loops, best of 5: 391 nsec per loop
# Bigger string
$ ./python.exe -m timeit -s "from json.encoder import c_encode_basestring; s = ''.join(map(str, range(100)))" "c_encode_basestring(s)"
500000 loops, best of 5: 877 nsec per loop
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQwMjU5MDc0MiwtNTgxMDg2NDU2XX0=
-->