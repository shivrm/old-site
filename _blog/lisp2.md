---
layout: default
title: Write You a Lisp, Part 2 - Crafting an Interpreter
date: 2022-05-13
---
Welcome back to *'Write You A Lisp'*, where I try to make an interpreted LISP dialect in Rust. In the last article, we built a lexer, and a parser. Now, we're going to improve the lexer, as well make an interpreter.<!--more-->

*The source code for the LISP is available on [GitHub Gist](https://gist.github.com/shivrm/e5ecb7a1e6c8e9678d6eb948e48e5a07).*

# A Faster Lexer

I'd like to thank [soemone](https://github.com/dquat) for [refactoring the lexer](https://github.com/shivrm/risp/pull/1). He did a lot of changes, which I'll document here.

## The `pos` attribute

Firstly, the lexer now has a new `pos` attribute which specifies it's current position in the source code.
```
pub struct Lexer<'a> {
    chars: Chars<'a>, // `chars` is an iterator over the characters of a string.
	pos: usize
}
```
```
// Inside the impl<'a> Lexer<'a>

pub fn new(text: &'a str) -> Self {
	Lexer {
		chars: text.chars(),
		pos: 0
	}
}

#[inline]
fn advance(&mut self) {
	self.pos += match self.chars.next() {
		Some(c) => c.len_utf8(),
		None => return,
	}
}
```

## New `Token`

We have a new struct that represents a token. It only stores its `kind`, as well as it's position in the source code. This new token will be parsed to the correct type and AST node by the parser.[^1]
```
#[derive(Debug)]
pub struct Token {
	pub kind: Kind,
	pub span: Span,
}
```

`Kind` is a enum indicating what kind of token it is...
```
#[derive(PartialEq, Debug)]
pub enum Kind {
	Name,
	Number,
	OpenParen,
	CloseParen,
	EOF
}
```

...and `Span` is a helper struct that lets us represent a range of characters in the source code. It lets us retrieve the start and end points of tokens. You can check it out in the [GitHub repo](https://github.com/shivrm/risp/blob/b0797f7b3a4776b05ce38170539ddf978f186d57/src/risp/utils.rs)

## `take_while`

In the last article, we used patterns such as `while let c @ Some( 'A'..='Z' | 'a'..='z' ) = self.current_char()` to read tokens; We also had separate methods for reading numbers and names.

We're going to combine these into a single method - `take_while` - which will read characters from the source while a `predicate` is met:
```
// Inside the impl<'a> Lexer<'a>

fn take_while(&mut self, mut predicate: impl FnMut(char) -> bool) -> Span {
	let start = self.pos;

	loop {
		match self.current_char() {
			Some(c) if predicate(c) => self.advance(),
			_ => return Span::new(start, self.pos),
		}
	}
}
```

`predicate` is called with the current character as an argument, and the returned boolean determines whether the lexer should advance to the next character.

Now that we've generalized this part, we can rewrite the `next_token` method. The method now reads characters from the lexer while the conditions (earlier defined in the `read_name` and `read_number` method) are met. The `Span` returned from this method is used to construct a `Token`.

```
// Inside the impl<'a> Lexer<'a>

pub fn next_token(&mut self) -> Token {
	let start = self.pos;

	// Checks if there is a current character
	if let Some(c) = self.current_char() {
		match c {
			c if c.is_whitespace() => {
				self.take_while(|c| c.is_whitespace());
				self.next_token()
			}

			'0'..='9' => {
				let span = self.take_while(|c| matches!(c, '0'..='9'));
				Token { span, kind: Kind::Number }
			}

			'a'..='z' | 'A'..='Z' | '_' => {
				let span =
					self.take_while(|c| matches!(c, 'a'..='z' | 'A'..='Z' | '_' | '0'..='9'));
				Token { span, kind: Kind::Name }
			}

			c => {
				self.advance();
				let span = Span::new(start, self.pos);
				let kind = match c {
					'(' => Kind::OpenParen,
					')' => Kind::CloseParen,
					_ => panic!("Unknown character")
				};
				Token { span, kind }
			}
		}
	} else {
		// Return EOF if there is no current character
		Token {
			kind: Kind::EOF,
			span: Span::new(self.pos, self.pos),
		}
	}
}
```

Since we've moved the predicate functionality to `next_token`, we don't need the `read_name` and `read_number` methods anymore!

## Parser Modifications

In the parser, the token comparisons have been replaces with token `Kind` comparisons. Some methods have been updated to use the new token struct. This mostly involves `match`ing `token.kind` instead of `token`, and using `Kind::*` instead of `Token::*`

 The `variant_eq` function is no longer required as `Kind` does not contain any data other than the variant.
Therefore, we can update the `expect` method:
```
fn expect(&mut self, token_type: Kind) {
	if self.current_token.type != token_type {
		panic!("Did not find expected token");
	}
	
	self.advance();
}
```

Notably, we have added a `src` property to the parser, which stores the source code that it's parsing. This is necessary because we want to retrieve string slices of the source code for parsing.
```
pub struct Parser<'a> {
	lexer: Lexer<'a>,
	current_token: Token,
	src: &'a str,
}

impl<'a> Parser<'a> {
	pub fn new(mut lexer: Lexer<'a>, src: &'a str) -> Self {
		Self {
			current_token: lexer.next_token(),
			src,
			lexer,
		}
	}
}
```

The `parse_atom` method has also been modified to convert the token to the correct type:
```
// Inside the impl<'a> Parser<'a>

fn parse_atom(&mut self) -> AstNode {
	let content = &self.src[self.current_token.span.range()];

		let node = match &self.current_token.kind {
		Kind::Number => AstNode::Number(content.parse().unwrap()),
		Kind::Name => AstNode::Name(content.to_owned()),
		_ => panic!("Expected Number or Name")
	};

	self.advance();
	node
}
```

That's pretty much all the changes. soemone and I benchmarked the lexer using the [`lex_speed` function](https://github.com/shivrm/risp/blob/b0797f7b3a4776b05ce38170539ddf978f186d57/src/main.rs#L53=L88) in the `main.rs` file. We got some great results:
```
Total elapsed: 1.3163127s
[max: 43ns, min: 23ns, avg: 26ns]
Average lex speed: 500.000 MB/s => 476.837 MiB/s
```

In comparison, the lexer made in the last post had a lex speed of 40-50 MB/s.

# Crafting an Interpreter

In the last post, we created a parser that parses tokens into AST nodes. Now, we're going to interpret the AST nodes using an interpreter. 

## The Type System

 First, we need to make a [type system](https://en.wikipedia.org/wiki/Type_system) that will store the different kinds of values. Just like AST nodes, types will be represented by an `enum`:
```
#[derive(Debug)]
pub enum Type {
	Number(i32),
	List(Vec<Type>),
	BuiltinFn(&'static dyn Fn(Vec<Type>) -> Vec<Type>),
	Null
}
```

Here:
- A `Number` is used to represent an integer (can be positive, negative or 0)
- A `List` is just a vector[^2] of `Type`s
- A `BuiltinFn` is a function that we'll define in Rust itself (or whatever language you're making it in). 
- `Null` is a null type.

`&'static dyn Fn(Vec<Type>) -> Vec<Type>` indicates a static function, which accepts a vector of `Type`s as an argument, and returns vector of `Type`s.
We use this to implement variadic functions[^3] which can accept and return any number of arguments.

## The Interpreter

The interpreter is yet another `struct`. However, we don't need to store any attributes in this struct right now. So I'll just create an empty struct:
```
struct Interpreter {}

impl Interpreter {
	fn new() -> Self {
		Self {}
	}
}
```

In the interpreter, we need a function that parses our AST nodes. Here's an overview of what it should do:

 - `Number` - return the value of the number.
 - `Name` - return a value associated with the name
 - `Expr` - The first element of an expression is the function to call. Call the function with the remaining elements as parameters.
 - 
```
// Inside the impl Interpreter

pub fn eval(&mut self, node: AstNode) -> Type {
	match node {
		AstNode::Name(name) => self.get_name(name.to_owned()),
		
		AstNode::Number(num) => Type::Number(num),

		AstNode::Expr(mut nodes) => {
			// Get the function (first element) and evaluate it
			let func = self.eval(nodes.remove(0));
			
			// Evaluate the rest of the elements to get the parameters for the function
			let mut params = Vec::new();
			for node in nodes.iter() {
				params.push(self.eval(node.clone()));
			}

			// Make sure that the function is a callable BuiltinFn
			if let Type::BuiltinFn(f) = func {
			
				// Call the function
				let mut result = f(params).clone();

				// Return Null if nothing was returned, the bare value if one was returned, and a List if more than one
				let value = match result.len() {
					0 => Type::Null,
					1 => result.pop().unwrap(),
					_ => Type::List(result),
				};

				value
			} else {
				panic!("Function is not callable");
			}
		}
	}
}
```

You'll also need to derive the `Clone` trait for AstNode:
```
#[derive(Debug, Clone)] // Debug was already derived
pub enum AstNode {
	...
}
```

The logic for `Expr` is a bit complicated, so I've added a few code comments to make it clearer. We also have a mystery `get_name` method, to get the value associated with a name.

Before I tell you what `get_name` does, Let's first create a function that we can associate with a name - a simple `println`:

```
fn println(elems: Vec<Type>) -> Vec<Type> {
	let mut iter = elems.iter();

	// Prints first element
	match iter.next() {
		Some(v) => print!("{v}"),
		None => (())
	}

	// Prints rest of the elements
	for el in iter {
		print!(" {el}");
	}

	print!("\n");

	Vec::new()
}
```

This fancy function just prints elements with spaces in between. We'll also need to implement the `Display` trait for our `Type` enum, so that they can be printed how we want them to be:

```
use std::fmt;

impl fmt::Display for Type {
	fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
		match self {
			Type::Number(n) => write!(f, "{n}"),
			
			Type::BuiltinFn(_) => write!(f, "<builtin function>"),
			
			Type::Null => write!(f, "Null"),
			
			Type::List(elems) => {
				// Again, the same fancy stuff as the println
				let mut iter = elems.iter();

				// Prints first element
				match iter.next() {
					Some(el) => write!(f, "({el}")?,
					None => write!(f, "(")?
				}

				// Prints rest of the elements
				for el in iter {
					write!(f, ", {el}")?;
				}
				
				// Closing brace
				write!(f, ")")
			}
		}
	}
}
```

Now, let's implement the `get_name` method for the parser. We only have a single name, so let's add a manual check for that:
```
// Inside the impl Interpreter

fn get_name(&self, name: String) -> Type {
	if name == "println" {
		Type::BuiltinFn(&println)
	} else {
		panic!("Unknown name {}", name);
	}
}
```

And with that, the interpreter is complete. Now, it's time to test it.

## Testing the Interpreter

Add this code to your `main` function:
```
fn main() {
	let text = "(println 1 2)"
	let lexer = Lexer::new(text);
	let ast = Parser::new(lexer, text).parse_expr();
	Interpreter::new().eval(ast);
}
```

And, on running, you should see
```
1 2
```

This means your interpreter is working, and you should congratulate yourself 👏👏👏. Stay tuned for the next article, where we implement error handling and dynamic libraries.

---

[^1]: Many lexers, including [Rust's own lexer](https://github.com/rust-lang/rust/tree/master/compiler/rustc_lexer) do lexing and parsing like this. Tokens are emitted from the lexer as a strings, and are parsed into the correct type by the parser.

[^2]: A vector (`Vec`) is simply a grow-able array. You'd use `std::vector` for this in C++, and `list` in Python.

[^3]: Our function technically accepts and returns only one argument, but since this is a `Vec<Type>` that can hold any number of elements, it's almost similar to the variadic functions seen in most interpreted languages.