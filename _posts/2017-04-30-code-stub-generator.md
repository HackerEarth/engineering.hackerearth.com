---
layout: post
title: "Generating code stubs"
description: "Generating input/output stub for multiple programming languages"
category:
tags: [Python, ]
---
{% include JB/setup %}

### What is code stub

Code stub is a harness which includes the code which contains -

    - Code to read input parameters from standard input.
    - Empty function/method to write problem logic.
    - Code to pass input parameters to the user function.
    - Take the output from user function and print it to standard output.

### Why is it needed
HackerEarth provides online code editor to solve problems on the platform. It may seem trivial for coders who are familiar with online judges, but someone new to this kind of thing get confused as to how to do basic I/O. We built code stubs as a solution to these problems. Using code stub programmers can concentrate on writing the solution rather than worrying about I/O.

### How to generate code stub
Let's take an example of generating code stub for the following problem.
Print sum of two integers.
Input -
First line contains first integer
Second line contains second integer
Output -
Print sum of both integers

*stubgen* is the package which contains stub generators for all the supported programming languages. All stub generators are children of *BaseStubGenerator*, which has abstract methods for input and output code for each supported datatype. Children of *BaseStubGenerator* implement those abstract methods as they are programming language dependant. To get input and output code,  *BaseStubGenerator* calls the method responsible for getting I/O code for each datatype and passes that as context to a template and then the final stub is rendered.
This can be understood better in the following diagram.

<img style="display: block; margin: 0 auto;" src="/images/codestub_diagram.png" alt="Codestub Architecture" title="Depiction of the codestub architecture"/>

Keeping this structure in mind, to generate above problem's code stub -

{% highlight python %}
from stubgen import CStubGenerator
stub_generator = CStubGenerator('user_fn',
                                {'n1': INTEGER, 'n2': INTEGER},
                                INTEGER,
                                function_doc='Write solution here')
{% endhighlight %}

The configuration parameters are -

    - user_fn: name of user function where users will write their solution
    - {'n1': INTEGER, 'n2': INTEGER}: mapping of input parameter names and their type. In this case code will be generated to read two parameters both of type integer. This will be passed to get_input_code() method. Which in turn, will call get_input_code_integer() method twice, once with param_name='n1' and again with param_name='n2'.
    - INTEGER: Type of output. In this case, get_output_code() will be called with INTEGER, which in turn will call _get_output_code_integer.
    - function_doc: Optional comments to be written inside the user function, which may be used to provide any hints to the user.

### Conclusion

After We have collected code for input and output, the next problem is how to put everything into a single working program, where the user can write solution for the problem. We use jinja2 template for that. A fixed template is created for each programming language which has holes where the generated code can be slotted into. This could also be done with string interpolation, but using a template gives us more flexibility to fit different programming languages especially whitespace sensitive languages like Python.
