+++
title = "Python scripts from the command line"
slug = "python-16-scripts"
weight = 16
+++

In this lesson we'll work with bash command line, instead of the Jupyter notebook. We want to write a python script
'read.py' that takes a set of gapminder_gdp_*.csv files (one/few/many) as an argument and prints out various statistic
(min, max, mean) for each year for all countries in that file:

~~~ {.bash}
$ ./read.py --min data-python/gapminder_gdp_{americas,europe}.csv   # minimum for each year
$ ./read.py --max data-python/gapminder_gdp_europe.csv              # maximum for each year
$ ./read.py --mean data-python/gapminder_gdp_europe.csv data-python/gapminder_gdp_asia.csv
~~~

It would be best to open two shells for the following programming, and keep nano always open in one shell.

Put the following into a file called read.py:

~~~ {.python}
#!/usr/bin/env python
import sys
print('argument is', sys.argv)
~~~

and then run it from bash:

~~~ {.bash}
$ chmod u+x read.py
$ ./read.py     # it'll produce: argument is ['read.py'] (the name of the script is always there)
$ ./read.py one two three     # it'll produce: argument is ['read.py', 'one', 'two', 'three']
~~~

Let's modify our script:

~~~ {.python}
import sys
print(sys.argv[1:])
~~~
~~~ {.bash}
$ python read.py one two three     # it'll produce ['one', 'two', 'three']
~~~

Let's modify our script:

~~~ {.python}
import sys
for f in sys.argv[1:]:
    print(f)
~~~
~~~ {.bash}
$ python read.py one two three     # it'll produce one two three (one per line)
$ python read.py data-python/gapminder_gdp_*
$ python read.py data-python/gapminder_gdp_americas.csv data-python/gapminder_gdp_europe.csv
$ python read.py data-python/gapminder_gdp_{americas,europe}.csv     # same as last line
~~~

Let's actually read the data:

~~~ {.python}
import sys
import pandas as pd
for f in sys.argv[1:]:
    print('\n', f[26:-4].capitalize())
    data = pd.read_csv(f, index_col='country')
    print(data.shape, '\n')
~~~
~~~ {.bash}
$ python read.py data-python/gapminder_gdp_{americas,europe}.csv
$ wc -l !$      # the output should be consistent
~~~

Let's modify our script changing print(data.shape) -> print(data.mean()) to compute average per year for each file:

~~~ {.bash}
$ python read.py data-python/gapminder_gdp_{americas,europe}.csv
~~~

Let's add an *action* flag and calculate statistics based on the flag:

~~~ {.python}
import sys
import pandas as pd
action = sys.argv[1]
for f in sys.argv[2:]:
    print('\n', f[26:-4].capitalize())
    data = pd.read_csv(f, index_col='country')
    if 'continent' in data.columns.tolist():   # if 'continent' column exists
        data = data.drop('continent',1)        #     delete it (1 stands for column, 0 for row)
    if action == '--min':
        values = data.min().tolist()
    elif action == '--mean':
        values = data.mean().tolist()
    elif action == '--max':
        values = data.max().tolist()
    print(values,'\n')
~~~
~~~ {.bash}
$ python read.py --min data-python/gapminder_gdp_{americas,europe}.csv
$ python read.py --max data-python/gapminder_gdp_{americas,europe}.csv
$ python read.py --mean data-python/gapminder_gdp_{americas,europe}.csv
$ python read.py --mean data-python/gapminder_gdp_*.csv   # should print data for all five continents
~~~

Let's add assertion for action (syntax is: assert n > 0.0, 'should be positive') right after the
definition of *action*:

~~~ {.python}
assert action in ['--min', '--mean', '--max'], 'action must be one of: --min --mean --max'
~~~

and try passing some other action.

Now let's package processing of a file (reading + computing + printing) as a function:

~~~ {.python}
import sys
import pandas as pd
action = sys.argv[1]
assert action in ['--min', '--mean', '--max'], 'action must be one of: --min --mean --max'
filenames = sys.argv[2:]
def process(filename, action):
    print('\n', filename[26:-4].capitalize())
    data = pd.read_csv(filename, index_col='country')
    if 'continent' in data.columns.tolist():   # if 'continent' column exists
        data = data.drop('continent',1)        #     delete it (1 stands for column, 0 for row)
    if action == '--min':
        values = data.min().tolist()
    elif action == '--mean':
        values = data.mean().tolist()
    elif action == '--max':
        values = data.max().tolist()
    print(values,'\n')
for f in filenames:
    process(f, action)
~~~

## Adding standard input support to Python scripts

Python scripts can process standard input. Consider the following script:

~~~
#!/usr/bin/env python
import sys
for line in sys.stdin:
    print(line, end='')
~~~

In the terminal make it executable (`chmod u+x scriptName.py`), and then run it:

~~~
./scriptName.py                          # repeat each line you type until Ctrl-C
echo one two three | ./scriptName.py     # print back the line
cat file.extension | ./scriptName.py     # process this file (filename=sys.stdin) from standard input
tail -1 scriptName.py | ./scriptName.py      # print its own last line
~~~

Let's add support for Unix standard input. Delete 'print('\n', f[26:-4].capitalize())' and change the last two lines to:

~~~ {.python}
if len(filenames) == 0:
    process(sys.stdin,action)       # process standard input
else:
    for f in filenames:             # same as before
        print('\n', f[26:-4].capitalize())
        process(f,action)
~~~
~~~ {.bash}
$ python read.py --mean data-python/gapminder_gdp_europe.csv
$ python read.py --mean < data-python/gapminder_gdp_europe.csv    # anyone knows why this could be useful?
~~~

Here is why: you can do preprocessing in bash before passing the data to your Python script, e.g.

~~~ {.bash}
$ head -5 data-python/gapminder_gdp_europe.csv | python read.py --mean    # process only first five countries
cp data-python/gapminder_gdp_asia.csv a1
cat data-python/gapminder_gdp_europe.csv | sed '1d' >> a1   # add all European data without the header
cat a1 | python read.py --mean          # merge Asian and European data and calculate joint statistics
~~~

Here is our full script:

~~~ {.python}
import sys
import pandas as pd
action = sys.argv[1]
assert action in ['--min', '--mean', '--max'], 'action must be one of: --min --mean --max'
filenames = sys.argv[2:]
def process(filename, action):
    data = pd.read_csv(filename, index_col='country')
    if 'continent' in data.columns.tolist():   # if 'continent' column exists
        data = data.drop('continent',1)        #     delete it (1 stands for column, 0 for row)
    if action == '--min':
        values = data.min().tolist()
    elif action == '--mean':
        values = data.mean().tolist()
    elif action == '--max':
        values = data.max().tolist()
    print(values,'\n')
if len(filenames) == 0:
    process(sys.stdin,action)       # process standard input
else:
    for f in filenames:             # same as before
        print('\n', f[26:-4].capitalize())
        process(f,action)
~~~
