+  元文書: [https://github.com/wycats/handlebars-site/blob/614718c76b8eb270a99b344bb60a97d4a5e7e9bb/src/pages/block_helpers.haml](https://github.com/wycats/handlebars-site/blob/614718c76b8eb270a99b344bb60a97d4a5e7e9bb/src/pages/block_helpers.haml)

Block helpers make it possible to define custom iterators and
other helpers that can invoke the passed block with a new
context.

## Basic Blocks {#basic-blocks}

Let's define a simple block helper that simply invokes the
block as though no helper existed.

```
<div class="entry">
  <h1>{{title}}</h1>
  <div class="body">
    {{#noop}}{{body}}{{/noop}}
  </div>
</div>
```

The <code>noop</code> helper will receive an options hash.
This options hash contains a function (<code>options.fn</code>)
that behaves like a normal compiled Handlebars template.
Specifically, the function will take a context and return a
String.

```
Handlebars.registerHelper('noop', function(options) {
  return options.fn(this);
});
```

Handlebars always invokes helpers with the current context as
<code>this</code>, so you can simply invoke the block with
<code>this</code> to evaluate the block in the current context.

## The <code>with</code> helper {#with-helper}

Based on the description of the <code>noop</code> helper, it
should be obvious how to implement a <code>with</code> helper.
Instead of invoking the block with the current context, we will
invoke it with whatever context the template passed in.

```
<div class="entry">
  <h1>{{title}}</h1>
  {{#with story}}
    <div class="intro">{{{intro}}}</div>
    <div class="body">{{{body}}}</div>
  {{/with}}
</div>
```

You might find a helper like this useful if a section of your
JSON object contains a lot of important properties, and you
don't want to constantly need to repeat the parent name. The
above template could be useful with a JSON like:

```
{
  title: "First Post",
  story: {
    intro: "Before the jump",
    body: "After the jump"
  }
}
```

Implementing a helper like this is a lot like implementing
the <code>noop</code> helper. Note that helpers take
parameters, and parameters are evaluated just like expressions
used directly inside <code>{{mustache}}</code> blocks.

```
Handlebars.registerHelper('with', function(context, options) {
  return options.fn(context);
});
```

Parameters are passed to helpers in the order that they
are passed, followed by the block function.

## Simple Iterators {#iterators}

A very common use-case for block helpers is using them to define
custom iterators. In fact, all Handlebars built-in helpers are
defined as regular Handlebars block helpers. Let's take a look
at how the built-in <code>each</code> helper works.

```
<div class="entry">
  <h1>{{title}}</h1>
  {{#with story}}
    <div class="intro">{{{intro}}}</div>
    <div class="body">{{{body}}}</div>
  {{/with}}
</div>
<div class="comments">
  {{#each comments}}
    <div class="comment">
      <h2>{{subject}}</h2>
      {{{body}}}
    </div>
  {{/each}}
</div>
```

In this case, we want to invoke the block passed to <code>each</code>
once for each element in the comments Array.

```
Handlebars.registerHelper('each', function(context, options) {
  var ret = "";

  for(var i=0, j=context.length; i<j; i++) {
    ret = ret + options.fn(context[i]);
  }

  return ret;
});
```

In this case, we iterate over the items in the passed parameter,
invoking the block once with each item. As we iterate, we build
up a String result, and then return it.

It is easy to see how you could use this to implement more advanced
iterators. For instance, let's create an iterator that creates a
<code>&lt;ul&gt;</code> wrapper, and wraps each resulting element
in an <code>&lt;li&gt;</code>.

```
{{#list nav}}
  <a href="{{url}}">{{title}}</a>
{{/list}}
```

You would evaluate this template using something like this as
the context:

```
{
  nav: [
    { url: "http://www.yehudakatz.com", title: "Katz Got Your Tongue" },
    { url: "http://www.sproutcore.com/block", title: "SproutCore Blog" },
  ]
}
```

The helper would not be that different from the original <code>each</code>
helper.

```
Handlebars.registerHelper('list', function(context, options) {
  var ret = "<ul>";

  for(var i=0, j=context.length; i<j; i++) {
    ret = ret + "<li>" + options.fn(context[i]) + "</li>";
  }

  return ret + "</ul>";
});
```

You could, of course, use a library like underscore.js or SproutCore's
runtime library to make this look a bit prettier. Using SproutCore's
runtime:

```
Handlebars.registerHelper('list', function(context, options) {
  return "<ul>" + context.map(function(item) {
    return "<li>" + options.fn(item) + "</li>";
  }).join("\n") + "</ul>";
});
```

## Conditionals {#conditionals}

Another relatively common use-case for block helpers is to implement
conditionals. Again, Handlebars' built-in <code>if</code> and
<code>unless</code> control structures are implemented as regular
Handlebars helpers.

```
{{#if isActive}}
  <img src="star.gif" alt="Active">
{{/if}}
```

Control structures typically do not change the current context, but
decide whether or not to invoke the block based upon some variable.

```
Handlebars.registerHelper('if', function(conditional, options) {
  if(conditional) {
    return options.fn(this);
  }
});
```

When writing a conditional, you will often want to make it possible
for templates to provide a block of HTML that your helper should
insert if the conditional evaluates to false. Handlebars handles
this problem by providing generic <code>else</code> functionality
to block helpers.

```
{{#if isActive}}
  <img src="star.gif" alt="Active">
{{else}}
  <img src="cry.gif" alt="Inactive">
{{/if}}
```

Handlebars provides the block for the <code>else</code> fragment
as <code>options.inverse</code>. If the template provides no inverse
section, Handlebars will automatically create a noop function, so
you don't need to check for the existance of the inverse.

```
Handlebars.registerHelper('if', function(conditional, options) {
  if(conditional) {
    return options.fn(this);
  } else {
    return options.inverse(this);
  }
});
```

Handlebars provides additional metadata to block helpers by attaching
them as properties of the options hash. Keep reading for more
examples.

## Hash Arguments {#hash-arguments}

Like regular helpers, block helpers can accept an optional Hash
as its final argument. Let's revisit the <code>list</code> helper
from earlier, and make it possible for us to add any number of
optional attributes to the <code>&lt;ul&gt;</code> element we will
create.

```
{{#list nav id="nav-bar" class="top"}}
  <a href="{{url}}">{{title}}</a>
{{/list}}
```

Handlebars provides the final hash as <code>options.hash</code>. This
makes it easier to accept a variable number of parameters, while
also accepting an optional Hash. If the template provides no hash
arguments, Handlebars will automatically pass an empty object
(<code>{}</code>), so you don't need to check for the existance of
hash arguments.

```
Handlebars.registerHelper('list', function(context, options) {
  var attrs = SC.keys(options.hash).map(function(key) {
    key + '="' + options.hash[key] + '"';
  }).join(" ");

  return "<ul " + attrs + ">" + context.map(function(item) {
    return "<li>" + options.fn(item) + "</li>";
  }).join("\n") + "</ul>";
});
```

Hash arguments provide a powerful way to offer a number of optional
parameters to a block helper without the complexity caused by
positional arguments.

Block helpers can also inject private variables into their child
templates. This can be useful to add extra information that is
not in the original context data.

For example, when iterating over a list, you may provide the
current index as a private variable.

```
{{#list array}}
  {{@index}}. {{title}}
{{/list}}
```

```
Handlebars.registerHelper('list', function(context, options) {
  var out = "<ul>", data;

  for (var i=0; i<context.length; i++) {
    if (options.data) {
      data = Handlebars.createFrame(options.data || {});
      data.index = i;
    }

    out += "<li>" + options.fn(context[i], { data: data }) + "</li>";
  }

  out += "</ul>";
  return out;
});
```

Private variables provided via the <code>data</code> option are
available in all descendent scopes.

Make sure you create a new data frame each time you invoke a block
with data. Otherwise, downstream helpers might unexpectedly mutate
upstream variables.

Also ensure that the <code>data</code> field is defined prior to attempting to
interact with an existing data object. The private variable behavior is condtionally
compiled and some templates might not create this field.