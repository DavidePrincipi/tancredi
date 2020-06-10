---
nav_order: 2
---

# Templates
{: .no_toc }

When a phone is powered on it asks one or more files containing its
configuration from an HTTP server. Tancredi generates the requested file
dynamically, by interpolating a template file with the values of the phone
configuration variables.

- TOC
{:toc}

## Custom templates

The template file is processed by [Twig 1](https://twig.symfony.com/doc/1.x/) 
and is stored under the `data/templates` directory with the `.tmpl`
extension. The system administrator can override an existing template
file by adding a file with the same name under the `templates-custom` directory.

The base path of the `templates-custom` directory is defined in the
`/etc/tancredi.conf` configuration file, by the `rw_dir` parameter.

## Template/Request binding: the `patterns.d` directory

When a phone requests a file, Tancredi establishes what template has to be
interpolated to build the response by looking at the `.ini` files under the
`data/patterns.d` directory.

Each `.ini` file in that directory defines one or more request matching rules.
The rule definition has the following lines:

1. The `[rule identifier]` section

1. The `pattern = ` line is a regexp matching the HTTP request file name. The
rule is effective when the match occurs. As rules are evaluated in alphabetical
order, the first match stops the rules evaluation.

1. The `scopeid = ` line is the identifier of the returned scope. Variable
values depend also on values inherited from parent scopes (see below [Variable value inheritance](#variable-value-inheritance) for details) and from
"run-time" variables (see the sections below).

1. The `template = ` line is the **name of a variable** containing the template
file name.  This makes possible to set an alternative template for a specific
phone or selecting the file format depending on the requested file extension. 

1. The `content_type = ` line sets the HTTP response mime type and must reflect the template 
output format.

This is an example of `.ini` file inside `patterns.d`:

```ini
[yealink-MAC-1]
pattern = "(00)(15)(65)([a-f0-9]{2})([a-f0-9]{2})([a-f0-9]{2})\.cfg"
scopeid = "$1-$2-$3-$4-$5-$6"
template = "tmpl_phone"
content_type = "text/plain; charset=utf-8"
```

## Variable value inheritance

Variables values are assigned inside a scope. A scope is stored in an `.ini`
file under the `scopes` directory. A scope can have a parent scope from which
variables are inherited.

A variable value is generally defined by the *most specific scope rule*. The
value is assigned by looking to the parent scopes in the following order:

1. defaults
2. model
3. phone

A variable value is **inherited** from the parent scope to the specific one and
is **overridden** by child scopes. In other words, if the child scope does not
define a variable the value from the parent is considered.

For example:

1. the *defaults* scope has the following vars: `{v0: a, v1: b, v2: c}`,
2. the *model* scope with id "yealink" has: `{v1: q, v3: p}`
3. the *phone* scope with id "00-00-00-00-00-00" has: `{v2: x, v4: y}`

When a template is expanded for the phone with MAC "00-00-00-00-00-00" the
variables context is: `{v0: a, v1: q, v2: x, v3: p, v4: y}`.

## External data sources for run-time variables

Sometimes it is useful to retrieve a variable value from an external data
source. It is possible to plug in a filter that adds or modifies the variables
passed to the Twig template context.

To add a runtime filter class (e.g. `SampleFilter`) add to the 
configuration file the following line:

    runtime_filters = "SampleFilter"

The file `src/Entity/SampleFilter.php` provides an example 
filter implementation.

The `SampleFilter` PHP class is instantiated just before the 
template is rendered and its `__invoke()` function is called. 
The scope variables are passed as an array argument.

The `__invoke()` function implementation can add, 
remove or modify any variable by returning a completely 
new array, representing the modified variables scope.

Filter classes are able to access the Tancredi configuration. 
For instance, add the following line to `/etc/tancredi.conf`:

    samplefilter_format = "d M Y H:i:s"

The `SampleFilter` class can access it with the `$config` array. 
