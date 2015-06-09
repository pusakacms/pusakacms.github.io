Lex
===

[![Build Status](https://secure.travis-ci.org/pyrocms/lex.png?branch=master)](http://travis-ci.org/pyrocms/lex)

Lex is a lightweight template parser.

_Lex is released under the MIT License and is Copyrighted 2011 - 2014 PyroCMS Team._

Change Log
==========

2.3.2
-----

* Convert objects with `->toArray()` at the beginning of the `parser()` method only.
* As much as we want to say goodbye to PHP 5.3, we brought it back for now.

2.3.1
-----

* Added an ArrayableInterface that automatically converts objects to arrays. This allows an opportunity to define how an object will be converted to an array by adding a `->toArray()` method.
* Looped data is now checked if it is an array or if it implements \IteratorAggregate. This prevents "invalid arguments in foreach" errors while allowing arrays and Collection objects to be iterated.
* Dropped support for PHP 5.3

2.3.0
-----

* Added support for self-closing Callback Tags (e.g. `{{ foo.bar /}}` instead of `{{ foo.bar }}{{ /foo.bar }}`).
* Rolled back the change to make Callback Tags less greedy.  Callback Tags are now greedy again.  If you want to use both Single and Block tags in the same template, you must close the Single tags.

2.2.3
-----

* Fixes issue which caused all callbacks to be processed as single tags, even if they were a block.

2.2.2
-----

* Fixed #7 - Conditionals inside looped data tags now work as expected.

2.2.1
-----

* Changed injectNoparse to be static.

2.2.0
-----

* Fixed a test which was PHP 5.4 only.
* Added PHPUnit as a composer dev requirement.
* Added a Lex\ParsingException class which is thrown when a parsing exception occurs.

2.1.1
-----

* Fixed an issue where strings returned by callbacks inside a comparison conditional were being processed incorrectly, causing the conditional to always fail.

2.1.0
-----

* Undefined variables in a conditional now evaluate to NULL, so `{{ if foo }}` now works properly.
* Added the `exists` keyword.
* Added the `not` keyword.

2.0.3
-----

* Fixes composer autoloading.
* Moved classes into lib folder.

2.0.2
-----

* Fixed a bug introduced in 2.0.1 where NULL variables were not being displayed.

2.0.1
-----

* Fixed a bug where variables with a "falsey" (e.g. 0, "0", -1, etc.) value were not displayed.

2.0.0
-----

* All code now follows PSR-0, 1 and 2.
* Lex_Parser has been moved to the `Lex` namespace and renamed to `Parser`.
* Lex_Autoloader has been removed.  It is now PSR-0 compliant.
* Added the support for `{{ unless }}` and `{{ elseunless }}`.



Basic Usage
===========

Using Lex
-------------

Lex is a Composer package named `pyrocms/lex`.  To use it, simply add it to the `require` section of you `composer.json` file.

    {
        "require": {
            "pyrocms/lex": "2.2.*"
        }
    }

After adding Lex to your `composer.json` file, simply use the class as normal.

    $parser = new Lex\Parser();

Using Lex
---------

Basic parsing of a file:

    $parser = new Lex\Parser();
    $template = $parser->parse(file_get_contents('template.lex'), $data);

You can also set the Scope Glue (see "Scope Glue" under Syntax below):

    $parser = new Lex\Parser();
    $parser->scopeGlue(':');
    $template = $parser->parse(file_get_contents('template.lex'), $data);

To allow noparse extractions to accumulate so they don't get parsed by a later call to the parser set cumulativeNoparse to true:

    $parser = new Lex\Parser();
    $parser->cumulativeNoparse(true);
    $template = $parser->parse(file_get_contents('template.lex'), $data);
	// Second parse on the same text somewhere else in your app
	$template = $parser->parse($template, $data);
	// Now that all parsing is done we inject the contents between the {{ noparse }} tags back into the template text
	Lex\Parser::injectNoparse($template);

If you only want to parse a data array and not worry about callback tags or comments, you can do use the `parseVariables()` method:

    $parser = new Lex\Parser();
    $template = $parser->parseVariables(file_get_contents('template.lex'), $data);


### PHP in Templates

By default PHP is encoded, and not executed.  This is for security reasons.  However, you may at times want to enable it.  To do that simply send `true` as the fourth parameter to your `parse()` call.

    $parser = new Lex\Parser();
    $template = $parser->parse(file_get_contents('template.lex'), $data, $callback, true);

Syntax
======

General
-------

All Lex code is delimeted by double curly braces (`{{ }}`).  These delimeters were chosen to reduce the chance of conflicts with JavaScript and CSS.

Here is an example of some Lex template code:

    Hello, {{name}}


Scope Glue
----------

Scope Glue is/are the character(s) used by Lex to trigger a scope change.  A scope change is what happens when, for instance, you are accessing a nested variable inside and array/object, or when scoping a custom callback tag.

By default a dot (`.`) is used as the Scope Glue, although you can select any character(s).

`Setting Scope Glue`

    $parser->scopeGlue(':');

Whitespace
----------

Whitespace before or after the delimeters is allowed, however, in certain cases, whitespace within the tag is prohibited (explained in the following sections).

**Some valid examples:**

    {{ name }}
    {{name }}
    {{ name}}
    {{  name  }}
    {{
      name
    }}

**Some invalid examples:**

    {{ na me }}
    { {name} }


Comments
--------

You can add comments to your templates by wrapping the text in `{{# #}}`.

**Example**

    {{# This will not be parsed or shown in the resulting HTML #}}

    {{#
        They can be multi-line too.
    #}}

Prevent Parsing
---------------

You can prevent the parser from parsing blocks of code by wrapping it in `{{ noparse }}{{ /noparse }}` tags.

**Example**

    {{ noparse }}
        Hello, {{ name }}!
    {{ /noparse }}

Variable Tags
-------------

When dealing with variables, you can: access single variables, access deeply nested variables inside arrays/objects, and loop over an array.  You can even loop over nested arrays.

### Simple Variable Tags

For our basic examples, lets assume you have the following array of variables (sent to the parser):

    array(
        'title'     => 'Lex is Awesome!',
        'name'      => 'World',
        'real_name' => array(
            'first' => 'Lex',
            'last'  => 'Luther',
        )
    )

**Basic Example:**

	{{# Parsed: Hello, World! #}}
    Hello, {{ name }}!

	{{# Parsed: <h1>Lex is Awesome!</h1> #}}
    <h1>{{ title }}</h1>

	{{# Parsed: My real name is Lex Luther!</h1> #}}
    My real name is {{ real_name.first }} {{ real_name.last }}

The `{{ real_name.first }}` and `{{ real_name.last }}` tags check if `real_name` exists, then check if `first` and `last` respectively exist inside the `real_name` array/object then returns it.

### Looped Variable Tags

Looped Variable tags are just like Simple Variable tags, except they correspond to an array of arrays/objects, which is looped over.

A Looped Variable tag is a closed tag which wraps the looped content.  The closing tag must match the opening tag exactly, except it must be prefixed with a forward slash (`/`).  There can be **no** whitespace between the forward slash and the tag name (whitespace before the forward slash is allowed).

**Valid Example:**

    {{ projects }} Some Content Here {{ /projects }}

**Invalid Example:**

    {{ projects }} Some Content Here {{/ projects }}

The looped content is what is contained between the opening and closing tags.  This content is looped through and output for every item in the looped array.

When in a Looped Tag you have access to any sub-variables for the current element in the loop.

In the following example, let's assume you have the following array/object of variables:

    array(
        'title'     => 'Current Projects',
        'projects'  => array(
            array(
                'name' => 'Acme Site',
                'assignees' => array(
                    array('name' => 'Dan'),
                    array('name' => 'Phil'),
                ),
            ),
            array(
                'name' => 'Lex',
                'contributors' => array(
                    array('name' => 'Dan'),
                    array('name' => 'Ziggy'),
					array('name' => 'Jerel')
                ),
            ),
        ),
    )

In the template, we will want to display the title, followed by a list of projects and their assignees.

    <h1>{{ title }}</h1>
    {{ projects }}
        <h3>{{ name }}</h3>
        <h4>Assignees</h4>
        <ul>
        {{ assignees }}
            <li>{{ name }}</li>
        {{ /assignees }}
        </ul>
    {{ /projects }}

As you can see inside each project element we have access to that project's assignees.  You can also see that you can loop over sub-values, exactly like you can any other array.

Conditionals
-------------

Conditionals in Lex are simple and easy to use.  It allows for the standard `if`, `elseif`, and `else` but it also adds `unless` and `elseunless`.

The `unless` and `elseunless` are the EXACT same as using `{{ if ! (expression) }}` and `{{ elseif ! (expression) }}` respectively.  They are added as a nicer, more understandable syntax.

All `if` blocks must be closed with the `{{ endif }}` tag.

Variables inside of if Conditionals, do not, and should not, use the Tag delimeters (it will cause wierd issues with your output).

A Conditional can contain any Comparison Operators you would do in PHP (`==`, `!=`, `===`, `!==`, `>`, `<`, `<=`, `>=`).  You can also use any of the Logical Operators (`!`, `not`, `||`, `&&`, `and`, `or`).

**Examples**

    {{ if show_name }}
        <p>My name is {{real_name.first}} {{real_name.last}}</p>
    {{ endif }}

    {{ if user.group == 'admin' }}
        <p>You are an Admin!</p>
    {{ elseif user.group == 'user' }}
        <p>You are a normal User.</p>
    {{ else }}
        <p>I don't know what you are.</p>
    {{ endif }}

    {{ if show_real_name }}
        <p>My name is {{real_name.first}} {{real_name.last}}</p>
    {{ else }}
        <p>My name is John Doe</p>
    {{ endif }}

    {{ unless age > 21 }}
        <p>You are to young.</p>
    {{ elseunless age < 80 }}
        <p>You are to old...it'll kill ya!</p>
    {{ else }}
        <p>Go ahead and drink!</p>
    {{ endif }}

### The `not` Operator

The `not` operator is equivilent to using the `!` operator.  They are completely interchangable (in-fact `not` is translated to `!` prior to compilation).

### Undefined Variables in Conditionals

Undefined variables in conditionals are evaluated to `null`.  This means you can do things like `{{ if foo }}` and not have to worry if the variable is defined or not.

### Checking if a Variable Exists

To check if a variable exists in a conditional, you use the `exists` keyword.

**Examples**

    {{ if exists foo }}
        Foo Exists
    {{ elseif not exists foo }}
        Foo Does Not Exist
    {{ endif }}

You can also combine it with other conditions:

    {{ if exists foo and foo !== 'bar' }}
        Something here
    {{ endif }}

The expression `exists foo` evaluates to either `true` or `false`.  Therefore something like this works as well:

    {{ if exists foo == false }}
    {{ endif }}

### Callback Tags in Conditionals

Using a callback tag in a conditional is simple.  Use it just like any other variable except for one exception.  When you need to provide attributes for the callback tag, you are required to surround the tag with a ***single*** set of braces (you can optionally use them for all callback tags).

**Note: When using braces inside of a conditional there CANNOT be any whitespace after the opening brace, or before the closing brace of the callback tag within the conditional.  Doing so will result in errors.**

**Examples**

    {{ if user.logged_in }} {{ endif }}

    {{ if user.logged_in and {user.is_group group="admin"} }} {{ endif }}


Callback Tags
-------------

Callback tags allow you to have tags with attributes that get sent through a callback.  This makes it easy to create a nice plugin system.

Here is an example

    {{ template.partial name="navigation" }}

You can also close the tag to make it a **Callback Block**:

    {{ template.partial name="navigation" }}
    {{ /template.partial }}

Note that attributes are not required.  When no attributes are given, the tag will first be checked to see if it is a data variable, and then execute it as a callback.

    {{ template.partial }}

### The Callback

The callback can be any valid PHP callable.  It is sent to the `parse()` method as the third parameter:

    $parser->parse(file_get_contents('template.lex'), $data, 'my_callback');

The callback must accept the 3 parameters below (in this order):

    $name - The name of the callback tag (it would be "template.partial" in the above examples)
    $attributes - An associative array of the attributes set
    $content - If it the tag is a block tag, it will be the content contained, else a blank string

The callback must also return a string, which will replace the tag in the content.

**Example**

    function my_callback($name, $attributes, $content)
    {
        // Do something useful
        return $result;
    }

### Closing Callback Tags

If a Callback Tag can be used in single or block form, then when using it in it's singular form, it must be closed (just like HTML).

**Example**

    {{ foo.bar.baz }}{{ /foo.bar.baz }}

    {{ foo.bar.baz }}
        Content
    {{ /foo.bar.baz }}

#### Self Closing Callback Tags

You can shorten the above by using self-closing tags, just like in HTML.  You simply put a `/` at the end of the tag (there MUST be NO space between the `/` and the `}}`).

**Example**

    {{ foo.bar.baz /}}

    {{ foo.bar.baz }}
        Content
    {{ /foo.bar.baz }}


Recursive Callback Blocks
-------------

The recursive callback tag allows you to loop through a child's element with the same output as the main block. It is triggered
by using the ***recursive*** keyword along with the array key name. The two words must be surrounded by asterisks as shown in the example below.

**Example**

	function my_callback($name, $attributes, $content)
	{
		$data = array(
				'url' 		=> 'url_1',
				'title' 	=> 'First Title',
				'children'	=> array(
					array(
						'url' 		=> 'url_2',
						'title'		=> 'Second Title',
						'children' 	=> array(
							array(
								'url' 	=> 'url_3',
								'title'	=> 'Third Title'
							)
						)
					),
					array(
						'url'		=> 'url_4',
						'title'		=> 'Fourth Title',
						'children'	=> array(
							array(
								'url' 	=> 'url_5',
								'title'	=> 'Fifth Title'
							)
						)
					)
				)
		);

		$parser = new Lex\Parser();
		return $parser->parse($content, $data);
	}


In the template set it up as shown below. If `children` is not empty Lex will
parse the contents between the `{{ navigation }}` tags again for each of `children`'s arrays.
The resulting text will then be inserted in place of `{{ *recursive children* }}`. This can be done many levels deep.

	<ul>
		{{ navigation }}
			<li><a href="{{ url }}">{{ title }}</a>
				{{ if children }}
					<ul>
						{{ *recursive children* }}
					</ul>
				{{ endif }}
			</li>
		{{ /navigation }}
	</ul>


**Result**

	<ul>
		<li><a href="url_1">First Title</a>
			<ul>
				<li><a href="url_2">Second Title</a>
					<ul>
						<li><a href="url_3">Third Title</a></li>
					</ul>
				</li>

				<li><a href="url_4">Fourth Title</a>
					<ul>
						<li><a href="url_5">Fifth Title</a></li>
					</ul>
				</li>
			</ul>
		</li>
	</ul>
