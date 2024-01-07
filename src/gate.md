# Notes on Gate.lua

Tim Menzies  
Jan 6 2024

GATE  is a  simple demonstrator  of  an incremental  optimization method  called
sequential model optimzation.  GATE assumes that (a) data divides into X and Y
columns, and (b) it  is expensive to access the Y values.  In that case, GATE
learns how to recognize good Y-values, using just very few  Y values.

This is a four phase process:

| What | Called | Notes |
|------|--------|-------|
| G    | guess  | quickly generate some examples |
| A    | assess | using a surrogate model, assess which examples are interesting |
| T    | test   | test if the items found during assess are actually useful |
| E    | extend | extend the surrogate |

GATE uses _N1_ examples (picked at random) to initializes a Naive Bayes
classifier that  distinguishes the sqrt(_N_) _best_ examples from the _rest_.

Then, _N2_ times, we look for as-yet-unlabelled examples that might confuse that
classifier.  Specfifically, if an example has probabilties _b,r_ of belonging to
_best,rest_, then the most confused example is the one that maximizes
_abs(b+r)/abs(b-r)_.

This example is then evaluated and the model is extended.


## Installation

GATE is a single file. To grab the source code:

    curl https://raw.githubusercontent.com/timm/lo/6jan24/src/gate.lua

To find sample files:

    curl https://github.com/timm/lo/blob/6jan24/data/x

where  `x`   is  one   of  auto93.csv,  china.csv,   coc1000.csv, coc10000.csv,
healthCloseIsses12mths0001-hard.csv, healthCloseIsses12mths0011-easy.csv,
nasa93dem.csv, pom.csv.

## Inteface

## Data Format

GATE reads comma-seperated files (e.g. auto93.csv) whose first row names the columns.

- Names starting with uppercase are numeries; e.g. `Salary`. All other names are symbolic columns.
- Names ending with X are ignored by the reasoning; e.g. `SexM`.
- Numeric names ending with `-` or `+` are goals to be minimized or maximized; e.g. `Weight-` and `Salary+`.
- Symolic names ending with `!` are classes to be recognized; e.g. `happy!`.

## Code Format

gate.lua has sections: help text, classes, library functions, examples.

### Help Text

The top of gate.lua is a help string from which this code extracts the system's
config.

```txt
gate: guess, assess, try, expand
(c) 2023, Tim Menzies, BSD-2
Learn a little, guess a lot, try the strangest guess, learn a little more, repeat

USAGE:
  lua gate.lua [OPTIONS]

OPTIONS:
  -c --cohen    small effect size               = .35
  -f --file     csv data file name              = ../data/diabetes.csv
  -h --help     show help                       = false
  -k --k        low class frequency kludge      = 1
  -m --m        low attribute frequency kludge  = 2
  -s --seed     random number seed              = 31210
  -t --todo     start up action                 = help
```

The parser for this help is very simple (it just looks for a regular
expression that finds a word after two dashes: 

       [-][-]([%S]+)[^=]+= ([%S]+)

```lua
function l.settings(s,    t,pat)
  t,pat = {}, "[-][-]([%S]+)[^=]+= ([%S]+)"
  for k, s1 in s:gmatch(pat) do t[k] = l.coerce(s1) end
  t._help = s
  return t end
```

That parser needs to coerse a string:

```
function l.coerce(s1,    fun)
  function fun(s2)
    if s2=="nil" then return nil else return s2=="true" or (s2~="false" and s2) end end
  return math.tointeger(s1) or tonumber(s1) or fun(s1:match'^%s*(.*%S)') end
```