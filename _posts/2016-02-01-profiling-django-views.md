---
layout: post
title: "Profiling django views for SQL queries"
description: "We created a SQL profiler for python functions which tells what
exact expressions inside the function body triggers some network call like SQL
queries by manipulating AST (Abstract Syntax Trees) of function code."
category:
tags: [Django, Profiling, Python AST, AST manipulation]
---
{% include JB/setup %}


We at HackerEarth regularly conduct 24-hours internal hackathons usually once a
month to boost ourselves to get familiar with new technologies and to come up
with great ideas and hacks to increase our productivity. A hackathon project
can be anything from creating a new product to creating some tools which helps
our own devlopment. In the hakathon in Dec 2015, I came up with the idea to
create a profiler which could tell which code inside django views causes SQL
queries so that we can optimize them easily.


## Initial thoughts ##

There are already many good django packages to profile views for SQL queries.
One of them is
[django-toolbar](https://github.com/django-debug-toolbar/django-debug-toolbar)
which we already use. Django-toolbar is great but it shows all raw SQL queries
which are happening inside a view and you have to analyze each query, see
the whole traceback and figure out which line of code inside the view is triggering
the query.
This way, you can only figure out the line number of the code in a file, not the exact
function or expression or attribute access which is causing the query.
What I wanted was that a profiler should tell about the exact expressions which
trigger the SQL queries.

There were some solutions which came up in my mind to get it done like using
tracebacks or by manipulating AST of the python function.
[Praveen](https://www.hackerearth.com/@praveen97uma) had just told me
about the python's [ast](https://docs.python.org/2/library/ast.html) module
which can parse and modify code during runtime and I
was fascinated about using it in future. I chose to use AST manipulation to
implement the profiler and the trick here is to patch every function call,
attribute access and other constructs. Here is what I thought:

Suppose there is a function `f` which internally calls `get_user` triggering a SQL query

```python
@profile
def f(request):
    ...
    user = get_user(request)
    ...
```

and decorating the function `f` with `profile` decorator will manipulate its
AST code.
It will find all
locations of function calls inside function body and will encapsulate it
in our special function `call_handler`. So the function `f`'s definition will
become

```python
def f(request):
    ...
    user = call_handler(get_user, request)
    ...
```

`call_handler` will call the passed function with given arguments. Before
calling it will start tracking for the SQL queries. And after the function
has been called it will collect the sql\_queries and will store it with the
function so that we can see later if during call any SQL query was made.
Moreover, to attain the functionality of collecting queries I will have to
patch the django code which makes SQL queries (I will talk about it later).
It's definition would be something like

```python
def call_handler(func, *args, **kwargs):
    sql_counter.start_tracking()
    result = func(*args, **kwargs)
    sql_queries = sql_counter.collect_queries()
    store_queries(func, sql_queries)
    return result
```

And then we'll be able to see what calls inside function made SQL queries.


## Implementation ##


### How to manipulate AST? ###

The python docs of ast doesn't give much information on usage of it. Then I
found [Green Tree Snakes](https://greentreesnakes.readthedocs.org/en/latest/index.html).
Also I found an example to convert python code to javascript in a
[stackoverflow's answer](http://stackoverflow.com/a/31081432/2073855) using ast
transformation. I found them pretty helpful.


There were following improvements in initial ideas while implementing:

* There are many cases other than function calling where python code gets executed
e.g. on attribute access, automatic coercion to bool in `if`/`else if`
statements and automatic coercion to iter in `for` statement. So I'd to cover
them up too.

* The `call_handler` was making whole function body unhygienic because any other usage of
name `call_handler` will clash with it. So I thought to make a namespace with
very unique name that nobody will define in any other place. Here, I came up
with word `goofy` and replaced decorator `profile` with `goofy.profile()` and
replaced `call_handler` with `goofy.call_handler`. And the only thing that I'll
have to import is `goofy`.

* In `goofy.call_handler` I need to pass other information like `lineno`, `colno`
too along with the calling function.


Here is the code which transforms the AST

```python
import ast

class Transformer(ast.NodeTransformer):
    def __init__(self, lineno):
        """ lineno is the actual position of function code in source file
        """
        self._lineno = lineno
        self._dec_lineno = 1

    def visit_FunctionDef(self, func):
        """ Remove decorators so that the decorators doesn't get applied more.
        """
        for dec in func.decorator_list:
            if (isinstance(dec, ast.Call) and isinstance(dec.func,
                    ast.Attribute) and isinstance(dec.func.value,
                    ast.Name) and dec.func.value.id == 'goofy' and
                    dec.func.attr == 'profile'):
                self._dec_lineno = dec.lineno
        func.decorator_list = []
        func_ast = self.generic_visit(func)
        return func_ast

    def visit_Call(self, call):
        """ Change the function calling syntax
        func(*args, **kwargs) will be transformed to
        goofy.call_handler(func, line_no, col_no, *args, **kwargs)
        """
        call_ast = self.generic_visit(call)
        call_ast.args.insert(0, call_ast.func)
        call_ast.func = ast.Attribute(
                value=ast.Name(id='goofy', ctx=ast.Load()),
                attr='call_handler', ctx=ast.Load())
        call_lineno = self._lineno - self._dec_lineno + call_ast.lineno
        call_colno = call_ast.col_offset + 1
        call_ast.args.insert(1, ast.Num(call_lineno))
        call_ast.args.insert(2, ast.Num(call_colno))
        return call_ast
```

There are many variants of statements and expressions in python, details of
which you can find in
[ast](https://docs.python.org/2/library/ast.html#abstract-grammar) docs's
Abstract grammar section. So any type of node that has to be changed while
transforming the AST, a corresponding method of
visit\_\<NodeClass\> in Transformer class has to be written.
(like visit\_FunctionDef and visit\_Call methods in above code). In actual code, I'd
to implement visit\_Assign, visit\_Attribute, visit\_If, visit\_BoolOp and
visit\_For etc. too.


And here's the code for `goofy` class.

```python
def goofy_profiler(f):
    frame, filename, line_number, _, lines, _ = inspect.stack()[1]
    if lines[0].startswith('def'):
        line_number -= 1
    source = inspect.getsource(f)
    decorator_lineno = source.count('\n', 0, source.index('@goofy.profile')+1) + 1
    tree = ast.parse(source)

    transformer = Transformer(line_number)
    transformed_tree = transformer.visit(tree)

    ast.fix_missing_locations(transformed_tree)
    ast.increment_lineno(transformed_tree, line_number - decorator_lineno)

    module_globals = inspect.getmodule(f).__dict__
    exec(compile(transformed_tree, filename=filename, mode="exec"), module_globals)
    func = eval(f.__name__, module_globals)
    func = deco(func)
    return func

def deco(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        SQLCounter.clear_data()
        SQLCounter.current_func(func)
        start_time = datetime.now()
        result = func(*args, **kwargs)
        timedelta = datetime.now() - start_time
        print 'Function: {}.{}'.format(
                func.__module__, func.func_name)
        print 'Total processing time: {} ms'.format(
                timedelta.total_seconds()*1000)
        SQLCounter.show_data()
        return result
    return wrapper

class goofy(object):
    @staticmethod
    def profile():
        return goofy_profiler

    @staticmethod
    def call_handler(func, lineno, colno,  *args, **kwargs):
        sargs = (" {}(..)".format(func.__name__), lineno, colno)
        SQLCounter.before(*sargs)
        result = func(*args, **kwargs)
        SQLCounter.after(*sargs)
        return result
```

Also I'd to patch the execute\_sql method of SQLCompiler to collect sql queries.

```python
from django.db.models.sql.compiler import SQLCompiler
from django.db.models.sql.datastructures import EmptyResultSet
from django.db.models.sql.constants import MULTI

def execute_sql(self, *args, **kwargs):
    try:
        q, params = self.as_sql()
        if not q:
            raise EmptyResultSet
    except EmptyResultSet:
        if kwargs.get('result_type', MULTI) == MULTI:
            return iter([])
        else:
            return
    start = datetime.now()
    try:
        return self.__execute_sql(*args, **kwargs)
    finally:
        d = (datetime.now() - start)
        SQLCounter.insert({
            'query' : q, 'type' : 'sql',
            'time' : 0.0 + d.seconds * 1000.0 + d.microseconds/1000.0
        })

SQLCompiler.__execute_sql = SQLCompiler.execute_sql
SQLCompiler.execute_sql = execute_sql
```

And here is the code of SQLCounter which collects SQL queries and also the
information of what SQL queries got executed at different position inside a
function.

```python
class SQLCounter(object):
    @classmethod
    def clear_data(cls):
        cls.current_code = ''
        cls.check = False
        cls.data = defaultdict(list)
        cls.hit_count = defaultdict(int)

    @classmethod
    def before(cls, activity, lineno, colno):
        cls.check = True
        cls.current_code = ("line no {:<4}: {}".format(lineno, activity),
                lineno)
        cls.hit_count[cls.current_code] += 1

    @classmethod
    def after(cls, activity, lineno, colno):
        cls.check = False

    @classmethod
    def insert(cls, data):
        cls.data[cls.current_code].append(data)

    @classmethod
    def current_func(cls, func):
        cls.func = func

    @classmethod
    def show_data(cls):
        if not cls.data:
            return
        data = sorted(cls.data.items(),key=lambda a: a[0][1])
        table = []
        headers = ['Location', 'Hit', 'Queries', 'Time (ms)', 'Avg Time (ms)']
        total_tm = 0.0
        total_qs = 0
        for current_code, qdata in data:
            activity, _ = current_code
            qs = len(qdata)
            total_qs += qs
            tm = sum(k['time'] for k in qdata)
            total_tm += tm
            avg_tm = tm / qs
            hit = cls.hit_count[current_code]
            table.append([activity, hit, qs, tm, avg_tm])
        print "Total SQL queries: {},  Total time: {} ms".format(
                total_qs, total_tm)
        tabular_table = tabulate(table, headers, tablefmt="simple")
        print tabular_table
```

## Usage ##

There is a view `get_bot_submission_response` in our codebase and we applied `goofy.profile()`
decorator on it to profile it.

```python
from goofy import goofy

@goofy.profile()
def get_bot_submission_response(request, game):
    ...
```

When the view gets called it prints following output to console.

```python
Function: problems.views.get_bot_submission_response
Total processing time: 201.499 ms
Total SQL queries: 8,  Total time: 9.856 ms
Location                                        Hit    Queries    Time (ms)    Avg Time (ms)
--------------------------------------------  -----  ---------  -----------  ---------------
line no 100 : .user                               1          1        0.97            0.97
line no 100 : .player1                            1          1        1.38            1.38
line no 112 : .problem                            1          1        1.52            1.52
line no 119 :  player_2_name(..)                  1          2        2.193           1.0965
line no 138 :  get_game_data(..)                  1          2        2.244           1.122
line no 163 :  render_to_string(..)               1          1        1.549           1.549
```

All the .\<attribute\> tells the location of attribute access which triggered SQL queries.
.user, .player1, .problem are attribute access and player\_2\_name(..), get\_game\_data(..),
render\_to\_string(..) are function calls. I've used the [tabulate](https://pypi.python.org/pypi/tabulate)
package to pretty print the table.

## What's next ##

Not only SQL but any type of queries can be profiled in this model.
We also added the feature to profile memcached queries in goofy profiler.
Lots of more improvements can be done upon it.
We'll soon clean the goofy profiler's code and open source it on github.
Stay tuned!


*Posted by [Shubham Jain](http://hck.re/shubham).*
*You can follow me on twitter [@sjiitr](https://twitter.com/sjiitr)*
