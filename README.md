parser-generator.js
===================

A dynamic recursive descent LL(k) parser for creating grammar parsing functions in JavaScript. This library is based on the work of Dan Yoder in the Cruiser.Parse library.

Note: The following was adapted from [this wiki page](https://code.google.com/p/cruiser/wiki/Parse)

Introduction
============

When you reach the limits of regular expressions in JavaScript, you have two choices:

1. Realize that what you are trying to do is probably not a good idea.
2. Write yourself a parser generator and keep on going.

The addition of CSS 3 selector support broke our regex-based parser for Behaviors. And there were so many cool things to do with them, too. So we went ahead with option 2.

What this library gives you is the ability to write parsers very easily. We don’t use the YACC-like approach of generating JavaScript code – instead we just build up a big function. It isn’t as fast as code generation, but it is faster than regex, at least for non-trivial parsers (i.e., those that require multiple passes with different regex).

Example
=======

Here's the grammar for CSS (probably not perfect, but good enough for our purposes):

```javascript
var g = Behaviors.Grammar, s = Behaviors.Stylesheet, t = Behaviors.Translator, o = Parser.Operators;

// basic tokens
g.lbrace = o.token("{");
g.rbrace = o.token("}");
g.lparen = o.token(/\(/);
g.rparen = o.token(/\)/);
g.colon = o.token(":");
g.semicolon = o.token(";");
// comments
g.inlineComment = o.token(/\x2F\x2F[^\n]*\n/);
g.multilineComment = o.token(/\x2F\x2A(.|\n)*?\x2A\x2F/);
g.comments = o.ignore(o.any(g.inlineComment, g.multilineComment));
// attributes
g.attrName = o.token(/[\w\-\d]+/);
g.attrValue = o.token(/[^;\}]+/);
g.attr = o.pair(g.attrName, g.attrValue, g.colon);
g.attrList = o.list(g.attr, g.semicolon, true);
g.style = o.process(o.between(g.lbrace, g.attrList, g.rbrace), t.style);
// style rules
g.selector = o.token(/[^\{]+/);
g.rule = o.each(g.selector, g.style);
g.rules = o.process(o.many(g.rule), t.rules);
// parser
s._parse = o.process(many(any(g.comments, g.rules)), t.parse);
```

Most of it is pretty self-explanatory. The `t.*` functions (`style`, `rules`, and `parse`) transform the output tree from associative arrays to hashes, since, by default, the parser simply puts each matched token into an array.

For example, consider the following translator functions:

```javascript
Behaviors.Translator = {
	style: function(attrs) {
		return attrs.inject({}, function(h, a) {
			h[a[0]] = a[1];
			return h;
		});
	},
	rules: function(rx) {
		return rx.inject({}, function(h, r) {
			if (r) h[r[0]] = r[1];
			return h;
		});			
	},
	parse: function(rx) {
		return rx.inject($H({}), function(h, r) {
			h.merge(r);
			return h;
		});
	}
}
```

Also, functions like `pair` and `between` give the parser some additional intelligence beyond just sequences of tokens and alternate paths. The details of each function are provided below.

Functions
=========

Each of the functions described below return a function that will match based on the given criteria. You can then call the returned function with a string to parse. If it parses, it will return an array with the first element being the result of the parsing and the second element being the remaining string. Otherwise, it will throw a `Parser.Exception`.

* `token( token )`
  * Matches the given string or regex.
  * Note that you don’t need to pre-pend your regex with ^ since that is done for you.
* `any( rule1, rule2, ... )`
  * Matches any of the given rules, attempting them in the order given.
* `each( rule1, rule2, ... )`
  * Maches all of the given rules, in the order given.
* `many( rule )`
  * Matches the given rule as many times as possible.
* `list( rule, [ delim, trailing ] )`
  * Matches a list whose elements are the given rule, delimited by the given delimiter (defaults to ’,’).
  * Also takes a boolean to indicate whether a trailing delimiter is allowed (defaults to false).
  * Only returns the elements matching the rule.
* `between( ldelim, rule, rdelim )`
  * Matches a rule between two delimiters.
  * Useful for things like matching a block between braces or an argument list between parenthesis.
  * Returns only the matched rule.
* `pair( rule1, rule2, [ delim ] )`
  * Matches two rules separated by a delimiter (defaults to ’,’).
  * Returns the two matched rules.
* `process( rule, fn )`
  * Matches the given rule and passes the result to the given function for processing.
  * Useful for transforming the parser output into something more semantically meaningful.
* `ignore( rule)`
  * Discard the matched input (returns an empty array on match).
  * Useful for matching things like comments.

Extensibility
=============

You can easily add additional operators. Just add them to `Parser.Operators`. We will provide more guidance on this topic in a future release. Also, please let us know if you have added any new operators! 