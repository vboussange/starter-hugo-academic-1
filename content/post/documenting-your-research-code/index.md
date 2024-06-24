---
featured: false
subtitle: ""
summary: "Documentation serves multiple purposes and may be useful for various audiences, including your future self, collaborators, users and contributors - should you aim at packaging some of your code into a general-purpose library."
date: "2024-06-11"
header-includes:
  - "\\newcommand{\\M}{\\mathcal{M}}"
draft: false
title: "A multi-language overview on how to document your research project code"
tags:
  - Julia
authors:
  - admin
  - Mauro Werder
lastmod: "2024-06-11"
---

Documentation serves multiple purposes and may be useful for various audiences, including your future self, collaborators, users and contributors - should you aim at packaging some of your code into a general-purpose library. 

This post is part of a series of posts on best practices for managing research project code. Much of this material was developed in collaboration with [Mauro Werder](https://github.com/mauro3) as part of the [Course On Reproducible Research, Data Pipelines, and Scientific Computing (CORDS)](https://github.com/mauro3/CORDS/tree/master). If you have experiences to share or spot any errors, please reach out!


- [Style guides](#style-guides)
- [Comments](#comments)
- [Literal documentation](#literal-documentation)
  - [README](#readme)
  - [API documentation / doc strings](#api-documentation--doc-strings)
  - [Type annotations](#type-annotations)
  - [Consider raising errors](#consider-raising-errors)
  - [Tutorials](#tutorials)
- [Accessing documentation](#accessing-documentation)
- [Doc testing](#doc-testing)
- [Useful packages to help you write and lint your documentation](#useful-packages-to-help-you-write-and-lint-your-documentation)
- [More resources](#more-resources)
- [Take home messages](#take-home-messages)


### Style guides
The best documentation starts by writing self-explanatory code with good conventions.

Correctly naming your variables enhances code clarity.

> There are only two hard things in Computer Science: cache invalidation and naming things.
*Martin Fowler*

Instead of using generic names like `l` for a list:

```
for l in L:
    pass
```

Use descriptive names like
```
for line in lines:
    pass
```


Using style guides for your chosen language ensures consistency and readability in your code. Here are some resources:

- [Julia style guide](https://docs.julialang.org/en/v1/manual/style-guide/)
- [Google python style guide](https://google.github.io/styleguide/pyguide.html)

Do not hesitate to refactor your code regularly and remove dead code to prevent confusion for yourself and others.

### Comments

In-line comments should be used sparingly. Aim to write self-explanatory code instead. Use comments to provide context not apparent from the code itself, such as references to papers, Stack Overflow topics, or TODOs.

Use single-line comments for brief explanations and multi-line comments for more detailed information.

**julia**
```julia
#=
This is a multi-line
comment
=#
```

**python**
```python
"""
This is a multi-line
comment
"""
```

Tip: use vscode `rewrap comment/text` to nicely format multiline comments.

On top of nicely formatting your code and appending comments where necessary, a literal documentation greatly facilitates the maintenance, understandability and reproducibility of your code.
### Literal documentation

Literal documentation helps users understand your tool and get started with it.

#### README
A README file is essential for any research repository. It is displayed on under the code structure when accessing a GitHub repo.
It should contain:
- (Badges showing tests, and a nice logo)
- A one-sentence description of your project
- A longer description
- An overview of the repository structure and files
- A _Getting started_ or _Examples_ section
- An _Installation_ section with dependencies
- A _Citation_/_Reference_ section
- (A link to the documentation)
- (A _How to contribute_ section)
- An _Acknowledgement_ section
- A _License_ section

Some examples:
<!-- - [HighDimPDE.jl](https://github.com/SciML/HighDimPDE.jl)
- [Code for HighDimPDE paper](https://github.com/SciML/HighDimPDE.jl)
- [Code for PiecewiseInference paper](https://github.com/vboussange/partitioning-time-series) -->
- [SatClip](https://github.com/microsoft/satclip)
- [GraphCast](https://github.com/google-deepmind/graphcast)
- [Alphafold](https://github.com/google-deepmind/alphafold)



#### API documentation / doc strings
API documentation describes the usage of functions, classes (types) and modules (packages). Parsers usually support markdown styles, which also enhances raw readability for humans. In short, markdown styles consists in using

  - backticks `` ` `` for variable names
  - `#` for titles,
  - ...
  - [See here for an introduction to markdown](https://github.com/carpentries-incubator/markdown-intro)

Doc strings in python live inside the function

```python
def best_function_ever(a_param, another_parameter):
    """
    this is the docstring
    """
    # do some stuff
```

But above the function or type definition in Julia
```julia
"""
this is the docstring
"""
function best_function_ever(a_param, another_parameter)
# do some stuff
end

"Tell whether there are too foo items in the array."
foo(xs::Array) = ...
```

**Best practice for docstrings include**
- (in Julia: insert the signature of your function )
- Short description
- Arguments (Args, Input,...)
- Returns
- Examples

Several flavours may be used, even for a single language.

**python**
3 Different documentation style flavours
- [reST (reStructuredText)](https://sphinx-rtd-tutorial.readthedocs.io/en/latest/docstrings.html)
- [Google style](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html)
- [Numpy style](https://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_numpy.html)


Google style is easier to read for humans

```python
def add(a, b):
    """
    Add two integers.

    This function takes two integer arguments and returns their sum.

    # Parameters:
    a: The first integer to be added.
    b: The second integer to be added.

    # Return:
    int: The sum of the two integers.

    # Raise:
    TypeError: If either of the arguments is not an integer.

    Examples:
    >>> add(2, 3)
    5
    >>> add(-1, 1)
    0
    >>> add('a', 1)
    Traceback (most recent call last):
        ...
    TypeError: Both arguments must be integers.
    """
    if not isinstance(a, int) or not isinstance(b, int):
        raise TypeError("Both arguments must be integers")
    return a + b
```

**julia**
````julia
"""
    add(a, b)

Adds two integers.

This function takes two integer arguments and returns their sum.

# Arguments
- `a`: The first integer to be added.
- `b`: The second integer to be added.

# Returns
- The sum of the two integers.

# Examples

```julia-repl
julia> add(2, 3)
5

julia> add(-1, 1)
0
```
"""
function add(a, b)
    return a + b
end
````

You may use tools like [`Documenter.jl`]() or [`Sphinx`](https://docs.readthedocs.io/en/stable/intro/getting-started-with-sphinx.html) to automatically render your API documentation on a website. Github actions can automatize the process of building the documentation for you, similarly to how it can automate testing.

Docstrings may be accompanied by typing.

#### Type annotations
Typing refers to the specification of variable types and function return types within a programming language. It helps define what kind of data a function or variable can handle, ensuring type safety and reducing runtime errors. It

-  clearly indicates the expected input and output types, making the code easier to understand.
-  helps catch type-related errors early in the development process.
-  encourages consistent usage of types throughout the codebase.

**python**
```python
def add(a: int, b: int) -> int:
    return a + b
```
In Python, using typing does not enforce type checking at runtime! You may use decorators to enforce it.

**julia**
```julia
function add(a::Int, b::Int)
    return a + b
end
```
In Julia, types are enforced at runtime! Type annotations help the Julia compiler optimize performance by making type inferences easier.

#### Consider raising errors
- We do not like reading manuals. But we are foreced to read error messages. Use assertions and error messages to handle unexpected inputs and guide users.


**python**
- `assert`: When an assert doesn’t pass, it raises an AssertionError. You can optionally add an error message at the end.
- `NotImplementedError`, `ValueError`, `NameError`: Commonly used, generic errors you can raise. I probably overuse `NotImplementedError` compared to other types.

```python
def convolve_vectors(vec1, vec2):
    if not isinstance(vec1, list) or not isinstance(vec2, list):
        raise ValueError("Both inputs must be lists.")
    # convolve the vectors
```

#### Tutorials
Create tutorial Jupyter notebooks or vignettes in R to demonstrate the usage of your code. Those can be placed in a folder `examples` or `tutorials`. Format them as e.g.

- vignettes in R,
- or using Jupyter notebooks, which are the perfect format for tutorials

### Accessing documentation

**julia**

```julia
?cos
?@time
?r""
```

**python**

```python
help(myfun)
```

But e.g. VSCode can be also quite helpful, and this works also with your own code!

![](figures/hover_doc.png)


### Doc testing
Doc testing, or doctest, allows you to test your code by running examples embedded in the documentation (docstrings). It compares the output of the examples with the expected results given in the docstrings, ensuring the code works as documented.

Why doc testing?
- Ensures that the code examples in your documentation are accurate and up-to-date.
- Simple to write and understand, making it accessible for both writing and reading tests.
- Promotes writing comprehensive docstrings which enhance code readability and maintainability.

**Python**
```python
def add(a, b):
    """
    Adds two numbers.

    >>> add(2, 3)
    5
    >>> add(-1, 1)
    0
    """
    return a + b
```
To run the test:

```
python -m doctest your_module.py
```
or from within a script
```python
if __name__ == "__main__":
    import doctest
    doctest.testmod()
```

**julia**
Available through `Documenter.jl`

````julia
"""
Adds two numbers.

```jldoctest
julia> add(2, 3)
5

julia> add(-1, 1)
0
```
"""

function add(a, b)
    return a + b
end

````

### Useful packages to help you write and lint your documentation
- Better Comments
![](figures/bettercomments.jpeg)
- Automatic doc string generation
![](figures/autodocstring.jpeg)
- Python test explorer
![](figures/pythontest.jpeg)


### More resources
- [Julia documentation recommendations](https://docs.julialang.org/en/v1/manual/documentation/#Writing-Documentation)
- [Good research tutorial on documentation](https://goodresearch.dev/docs)


- [Carpentries incubator on packaging and publish python - type roles](https://carpentries-incubator.github.io/python-packaging-publishing/05-documentation-types-roles/index.html)
- [Carpentries incubator on packaging and publish python - documentation in code](https://carpentries-incubator.github.io/python-packaging-publishing/06-documentation-in-code/index.html)


### Take home messages
- Good documentation helps maintain the long-term memory of a project.
- Refactor code to reduce complexity instead of documenting tricky code.
- Writing unit tests is often more productive than extensive documentation.
- Types of documentation include literal, API, and tutorial/example documentation.
- Literal documentation explains the big picture and setup.
- API documentation lives in docstrings and explains function usage.
- Examples connect the details to common tasks.
- Consider using tools like ChatGPT to assist with documenting your functions.
