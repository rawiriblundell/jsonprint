# jsonprint.sh
A shell function library to assist with formatting output into json

## Pre-emptive FAQ

Let's get real.  Nobody is asking any questions at all, let alone frequently.

This is, therefore, a *pre-emptive* FAQ where I guess what you'll ask.

### What is it?

`jsonprint.sh` is a library of functions that you source into your shell scripts,
you can then use its functions to format your outputs into a json structure.

### What isn't it?

* A tool that will magically take any input and figure it all out for you
* A tool that is full of input validation and hand-holding
* Robust, stable, a smart idea
* Something you should use if you can avoid it

### Example

For a basic example, let's take the formatting of `uname` and json-ify it.
Because `uname` does not natively give out safely parsable output, we do have to
make multiple calls to it, which is unfortunate and annoying, but relatively
low impact.

Here's the code, without comments:

```
include ../lib/jsonprint.sh

json_open
  json_obj_open uname
    json_str nodename "$(uname -n)"
    json_str_append os_kernel "$(uname -s)"
    uname -o >/dev/null 2>&1 && json_str_append os_name "$(uname -o)"
    json_str_append os_version "$(uname -v)"
    json_str_append release_level "$(uname -r)"
    json_str_append hardware_type "$(uname -m)"
    uname -i >/dev/null 2>&1 && json_str_append platform "$(uname -i)"
    uname -p >/dev/null 2>&1 && json_str_append processor "$(uname -p)"
  json_obj_close
json_close
```

This is what the output looks like when pretty-printed via `jq`, you can see
how the indentations match up with how I've indented the code above:

```
▓▒░$ bash json_uname | jq -r '.'
{
  "uname": {
    "nodename": "minty",
    "os_kernel": "Linux",
    "os_name": "GNU/Linux",
    "os_version": "#30~18.04.1-Ubuntu SMP Fri Jan 17 06:14:09 UTC 2020",
    "release_level": "5.3.0-28-generic",
    "hardware_type": "x86_64",
    "platform": "x86_64",
    "processor": "x86_64"
  }
}
```

`json_open()` simply prints `{`, and likewise, `json_close()` simply prints `}`

`json_obj_open` will open an object, in this case we have given it an argument,
so it will generate `"uname": {`

We have selected the nodename to be our first key value pair, and we know that
the value will be a string, so we call `json_str()` with the args `nodename` and
`"$(uname -n)"`.  This will give the output:

```
"nodename": "minty"
```

We know that subsequent key value pairs will be stacked onto this, 
so we call `json_str_append()` for those extra entries.  This function differs
from `json_str()` in that it prepends a comma, giving us

```
"nodename": "minty", "os_kernel": "Linux"
```

Subsequent invocations of `json_str_append()` will continue to stack keyvals.

Some versions of `uname` support options that may be nice to have, so we use
idioms like `uname -o >/dev/null 2>&1 && json_str_append os_name "$(uname -o)"`
That is to say:  If `uname -o` works, then generate e.g. `"os_name": "GNU/Linux"`,
otherwise, if `uname -o` doesn't work, then don't do anything here.  This kind
of idiom allows us to script portably, but to also throw in GNU-ish nice-to-haves
when and where we decide it's appropriate.

When all is said and done, we get an output that looks like this (line wrapping mine):

More advanced examples are available in the `bin/` directory.

### Why not a 'real' language? [insert smug face here]

In my professional past, I have worked on rusty-iron commercial unix machines
that do not have the likes of `python` or even `perl`.  Yes, seriously.  And,
even if they had one or both of these, they'd be ancient versions and very
likely devoid of their respective json modules.

There's also the wee issue of very-lightweight containers where the likes of
`python` might be strictly verboten.

You know what will *always* be there?  A POSIX compliant shell.

While this project is *really* for my own amusement, there are, for better or
worse, practical applications.  Perhaps you've got some small monitoring script
that you want to output in json format, but firing up a `python` instance is
really overkill.  Who knows?

### Why would you want to deal with json in shell at all?  That's nuts!

I agree, it is nuts.  Mostly for interactive use where throwing glue into
streams is fine.  But for shell scripting, you really want as much
robustness as you can get your hands on.

Consider this:  Experienced practitioners of the Unix shell are familiar
with its myriad warts, syntax oddities and edge cases.  One of the most classic
of which is the parsing `ls` trap.

Many newbie shell scripters will try to write code where they pluck details out
of `ls`, usually with some inefficient code like:

```
PERMISSIONS=`ls -la $FILE | cut -d ' ' -f1`
OWNER=`ls -la $FILE | cut -d ' ' -f3`
GROUP=`ls -la $FILE | cut -d ' ' -f4`
FILENAME=`ls -la $FILE | cut -d ' ' -f9-`
```

There's a number of problems here.

* UPPERCASE variables.  For Kildall's sake, just stop it.
    - Shell doesn't have strict scoping/namespacing
    - That said, UPPERCASE is, *de facto via convention*, the global scope
    - In other languages, you don't clobber the global scope
    - We should adopt good habits and practices from other languages where possible
    - Ergo:  **Don't use UPPERCASE unless you know why you need to**
    - There are, annoyingly, exceptions to the rule.  Like `$http_proxy`
* Backtick command substitution.  I'm damn near 40.  This crap was superseded
  by `$()` when I was soiling nappies.  Just stop it.
    - Unless you're writing SVR4 `sh` package scripts for Solaris packages
* Multiple avoidable calls to an external program
* Because date/timestamps might be different, there's no guarantee that the
  filename will be at the suggested field.  I've seen an attempt at working around
  this with a double invocation of `rev` e.g. `ls -la $FILE | rev | cut -d '' -f1 | rev`
  ...or something similarly nonsensical
* It's an unspoken golden rule of shell scripting:  Do not [parse ls](https://mywiki.wooledge.org/ParsingLs)

A slightly saner approach to this example might look something more like

```
while read -r; do
  set -- "${REPLY}"
  fsobj_mode="${1}"
  fsobj_owner="${3}"
  fsobj_group="${4}
  shift 8
  fsobj_name="${*}"
done < <(ls -la "${fsobj}")
```

Of course this is still prone to errors like date/timestamp vs locale issues.

In a json structure, these problems go away somewhat e.g:

```
▓▒░$ bash json_ls | jq -r '.[env.PWD][] | select(.fileName=="json_uname")'
{
  "fileName": "json_uname",
  "fileOwner": "rawiri",
  "fileGroup": "rawiri",
  "fileMode": 664,
  "sizeBytes": 919,
  "fileModified": 1582941110,
  "fileAccessed": 1582941111,
  "fileType": "regular file",
  "dereference": "json_uname"
}
```

And the greatest part?  We can build that structure using `stat`, and stay well
clear of `ls`.

### What about json's special characters?

`jsonprint.sh` provides a function, `json_str_escape()` which handles this.
It is currently not plumbed in and only works when manually called.

### What are the main problems with this, apart from everything else?

Apart from everything else?  Right now, the main gotcha is handling the logic
around an unknown number of inputs when looping.

Say you're looping over some lines and formatting it into key value pairs.  For
the sake of demonstration, we'll use four lines of input.

If you ran this blindly through a formatting function, this might come out like:

```
{"a": "b", "c": "d", "e": "f", "g": "h",}
```

The issue is very subtle - there is a trailing comma on the last pair.  That's
going to break things.

To get around this, I've naturally tried out a number of approaches.  Perhaps the
simplest is to use one of the append functions and a single-use variable to
track how many times you've gone through a loop.  For example

```
loop_iter=0
json_obj_open
  while read -r _key _value; do
    if (( loop_iter == 0 )); then
      json_str "${_key}" "${_value}"
      (( loop_iter++ ))
    else
      json_str_append "${_key}" "${_value}"
    fi
  done < <(some_input)
json_obj_close
```

If we step through this, we open an object with `json_obj_open`, which produces:

```
{
```

Next, we read into `_key` and `_value`, then test whether `loop_iter` is 0.  
If it's 0, then we're on our very first run through the loop, and so we need to
use `json_str()`.  This, combined with `json_obj_open` gives us:

```
{"a": "b"
```

Then we iterate `loop_iter` up by one, making it equal to `1`.

If that's the only object to be generated, then the loop finishes, there's no
trailing comma, and all is well.  `json_obj_close` is called, and we get:

```
{"a": "b"}
```

If there's more objects to be generated, because `loop_iter` is now `1`, we 
switch over to `json_str_append()`.  So the next line of input would be 
generated like (note the preceding comma and space):

```
, "c": "d"
```

Meaning a stacked output of:

```
{"a": "b", "c": "d"
```

There is no trailing comma, so it can be safely closed at any point.  And as we 
loop through the input, we simply stack our objects this way.
After a full run through, we have:

```
{"a": "b", "c": "d", "e": "f", "g": "h"}
```

And for the most part this works, I just haven't thought of a cleaner way to handle this (yet?)

### I'm looking at the code, why aren't you using local variables?

Not all shells support the `local` keyword/scope.  So as a convention, I use
underscore prepended variables and explicitly `unset` them at the end of each funcion.

This means that the library itself is more readily portable, and if it's not 
immediately portable, then it shouldn't be much effort to update it.

The downside is that if a function exits mid-flight, there's no trapping to
ensure that the variables are unset.

## List of Functions

All functions start with `json_`, even when this may seem weird.  I might change
that standard in the future to something like `jprint_` or `printj_`.  Or not.

### json_vorhees()

This function prints an exception message to stderr and immediately exits.
It is our variant of `die()`.

It has two aliases for the humourless:  `json_die` and `json_exception`.

### json_open()

Very simply prints a curly opening bracket: `{`

Would normally be used to denote the opening of a block of json.

### json_close()

The opposite of `json_open`.  It prints: `}`.

### json_comma()

This function literally prints a `,`, just in case you need one of those.

### json_decomma()

This function removes a trailing comma from its input.  This is from when the
library was structured differently, but may still be of some use.

### json_require()

**Args:** (Required).  Any number of paths to files or command names.

**Example:** `json_require /proc/cpuinfo lscpu`

If your script requires a file (or files) or a command (or commands), then you
can use `json_require()` to check the existence of these required files/commands.

Failure will emit a message like:

```
{ "Warning": "the_thing not found or not readable" }
```

And the function will also invoke `exit 1`.  This means that you should use this
basically immediately after sourcing the library, so that your script fails early.

### json_gettype()

**Args:** (Required).  One string.

**Example:** `json_gettype 0.05`

This function attempts to determine the "type" that a particular value is.  It
emits a determination from the following list:

* float
* int
* bool
* string

Floats and Integers are obvious.  Booleans are determined, case insensitive,
from the following values:

* on
* off
* yes
* no
* true
* false

Everything else is classed as a string.

The purpose of this function is to allow you to determine which output function
to select, for example:

```
case $(json_gettype "${_value}") in
  (int|float) json_num_append "${_key}" "${_value}" ;;
  (bool)      json_bool_append "${_key}" "${_value}" ;;
  (string)    json_str_append "${_key}" "${_value}" ;;
esac
```

### json_arr_open()

**Args:** (Optional).  One string.

**Example:** `json_arr_open jboss_server_stats`

This function opens an array block and accepts an optional arg.

If no argument is supplied, it simply outputs:

```
[
```

If an argument is supplied, it outputs:

```
"arg": [
```

### json_arr_close()

**Options:** `-c` or `--comma`.  When selected, this emits a trailing comma.

The opposite of `json_arr_open`.  It prints: `]`.

If used with `-c` or `--comma`, it prints `],`.  This is the opposite approach
to the `_append` functions.

### json_arr_append()

**Args:** (Optional).  One string.

**Example:** `json_arr_append jboss_application_stats`

This function appends an array to another array.  It emits the closing block for
the previous array, the comma seperator, and then opens the new array.

If no argument is supplied, it simply outputs:

```
],[
```

If an argument is supplied, it outputs:

```
], "arg": [
```

### json_obj_open()

**Args:** (Optional).  One string.

**Example:** `json_obj_open jboss_queue_stats`

This function opens an array block and accepts an optional arg.

If no argument is supplied, it simply outputs:

```
{
```

If an argument is supplied, it outputs:

```
"arg": {
```

### json_obj_close()

**Options:** `-c` or `--comma`.  When selected, this emits a trailing comma.

The opposite of `json_obj_open`.  It prints: `}`.

If used with `-c` or `--comma`, it prints `},`.  This is the opposite approach
to the `_append` functions.

### json_obj_append()

**Args:** (Optional).  One string.

**Example:** `json_obj_append jboss_memory_stats`

This function appends an object to another object.  It emits the closing block for
the previous object, the comma seperator, and then opens the new object.

If no argument is supplied, it simply outputs:

```
},{
```

If an argument is supplied, it outputs:

```
}, "arg": {
```

### json_str_escape()

**Args:** (None).  This functions reads stdin from a pipe.

**Example:** `somecommand | json_str_escape`

Some characters in json must be escaped.  A lot of advice at the better end of a
google will center around using `perl` or `python` to do this.  If we assume that,
then we may as well just use `perl` or `python` for everything else, right?!

So this function converts its stdin into a single column of octals.  Then it finds
any undesirable octals and prints an escaped replacement.

This might be computationally expensive, so try to avoid it if you can.

### json_str()

**Args:** (Required).  Two args: Key and Value.  If the value is blank or 
literally 'null', we return `null` (unquoted)

**Options:** `-c` or `--comma`.  When selected, this emits a trailing comma.

**Example:** `json_str CPU_Model "${cpu_model}`

This function formats a string key value pair, in the format: `"key": "value"`.
String values are quoted.

If used with `-c` or `--comma`, it prints `"key": "value",`.
This is the opposite approach to the `_append` functions.

### json_str_append()

As per `json_str()`, it just drops the `-c`/`--comma` options, and pre-pends a
comma i.e. `, "key": "value"`.  It otherwise behaves exactly the same.

### json_num()

**Args:** (Required).  Two args: Key and Value.

**Options:** `-c` or `--comma`.  When selected, this emits a trailing comma.

**Example:** `json_num Memory "${memory_value}`

This function formats a number (int or float) key value pair, in the format: 
`"key": value`.  Numerical values are unquoted.

The value is validated to ensure that it is an integer or float, if it isn't,
an exception will be thrown and the script will exit via `json_vorhees()`.

Leading zeroes are not allowed in json as they can be interpreted as octal, so
this function strips them as well.  In order to handle this and floats, we use
`printf`'s float format rather than the signed decimal format specified in the json spec.

If used with `-c` or `--comma`, it prints `"key": value,`.
This is the opposite approach to the `_append` functions.

### json_num_append()

As per `json_num()`, it just drops the `-c`/`--comma` options, and pre-pends a
comma i.e. `, "key": value`.  It otherwise behaves exactly the same.

### json_bool()

**Args:** (Required).  Two args: Key and Value.

**Options:** `-c` or `--comma`.  When selected, this emits a trailing comma.

**Example:** `json_bool interfaceActive True`

This function formats a boolean true/false key value pair, in the format: 
`"key": value`.  Boolean values are unquoted.

The value is validated to ensure that it is one of a recognised set of options.
If it isn't, an exception will be thrown and the script will exit via `json_vorhees()`.

The list is as follows:

* on
* off
* yes
* no
* true
* false

These are recognised in a case-insensitive manner, and converted to their respective
`true` or `false` forms in lowercase.

If used with `-c` or `--comma`, it prints `"key": value,`.
This is the opposite approach to the `_append` functions.

### json_bool_append()

As per `json_bool()`, it just drops the `-c`/`--comma` options, and pre-pends a
comma i.e. `, "key": value`.  It otherwise behaves exactly the same.

### json_from_dkvp()

**NOTE: Work In Progress.  Do Not Use**

This function takes a comma or equals delimited key-value pair input and emits
it in a way that can be used by e.g. `json_str()`

**Example:** a variable named `line` that contains `Bytes: 22`

```
json_num $(json_from_dkvp "${line}"
"Bytes": 22
```

The intent with this function is to loop through a series of 
delimited key value pairs (i.e. dkvp) and to restructure them slightly
into a json-friendly visage.

### json_foreach()

**Args:** (Required).  Any number of key value pairs all in one line.

**Options:** `-n` or `--name`.  When selected, this gives the surrounding object a name.

**Example:** `json_foreach key1 value1 key2 value2 .. keyN valueN`

This function takes any number of parameters and blindly structures every pair 
in the sequence into json keypairs, within an optionally named object structure.

If the option `-n` or `--name` is used, the object is given a name e.g.

`json_foreach --name cpu_details Brand Intel Model Pentium-D MHz 2100`

Will be printed as

`"cpu_details": {"Brand": "Intel", "Model": "Pentium-D", "Mhz": 2100}`

If the object name option is not used, then the object simply isn't named.

The key is stripped of any trailing instance of `:` or `=` and both the key
and value are trimmed of whitespace either side of them.

The value type is then determined via `json_gettype()` and the appropriate output
function selected and used.

Finally, the object is closed.

This object is structured in isolation.  If you want to append it, you might use
`json_comma()` before invoking this function.

There is no major input validation here, you must ensure that the input is sane.

## More resources

* https://json.org
* https://stedolan.github.io/jq/
* https://jqplay.org/
* https://github.com/antonmedv/fx
* https://github.com/jpmens/jo
* https://github.com/Juniper/libxo
* https://github.com/kellyjonbrazil/jc
