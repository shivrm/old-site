---
layout: default
title: Write You a Lisp - Part 1
date: 2022-05-03
updated: 2022-05-07
---
I've decided to learn more about how compilers work. I'm starting by trying to create
my own programming language - a simple Lisp.<!--more--> I'll be implementing it in Rust, but feel
free to follow along in whatever language you like.


I've you haven't already read [Ruslan Spivak's blog post series on building an interpeter](https://ruslanspivak.com/lsbasi-part1/  "Ruslan Spivak - Let's Build a Simple Interpreter, Part 1"), I highly suggest that you do. I'll be using his lexer and parser models in this tutorial.


# What is a Lisp, and Why should I write one?

Lisp is a very old family of Programming Languages; As of the time of writing, it is 64 years old; that's 14 years older than C! It's responsible for pioneering many features which we see today, such as dynamic typing, first-class functions, and even recursion and conditionals.

In addition to this, Lisps have a very simple grammar, that can be parsed easily. You can express it's grammar in [BNF notation](Ahttps://en.wikipedia.org/wiki/Backus–Naur_form) like so:

```
; an expression can be either an atom (a fundamental type) or a list
expr ::= atom | list

; an atom can be either a number, or a name (or symbol)
atom ::= number | name

; a list consists of zero or more expressions between a pair of parentheses
list ::= '(' expr* ')'
```

# Writing the Lexer

## The Tokens

The lexer is the first part of our interpreter. It splits a string into "tokens", each of which has an assigned meaning. For now, our Lisp will have five token types:
 - `Number`: A positive integer made from consecutive digits
 - `Name`: Consecutive letters representing a variable or function
 - `OpenParen` and `CloseParen`: represent opening and closing parentheses (`(` and `)`)
 - `EOF`: Used to indicate that no more characters can be read from the input

I'll be implementing this as a Rust `enum`:
```
pub enum Token {
	Number(i32),
	Name(String),
	OpenParen,
	CloseParen,
	EOF
}
```

## Designing the Lexer

Now that we have our tokens, we need a way to split a string and generate this Tokens. We'll use a `struct` called `Lexer` that does this. We'll also implement a `new` method that takes a string as an argument and returns a `Lexer`.

```
use std::str::Chars;

pub struct Lexer<'a> {
	chars: Chars<'a>	// `chars` is an iterator over the characters of a string.
}

impl<'a> Lexer<'a> {
	pub fn new(text: &'a str) -> Self {
		Lexer {
			chars: text.chars()
		}
	}
}
```

I'll also throw in a few methods to get the current character, and to advance to the next character.

```
// Inside the impl<'a> Lexer<'a>

#[inline]
fn current_char(&self) -> Option<char> {
	self.chars.clone().next()
}

#[inline]
fn advance(&mut self) -> Option<char> {
	self.chars.next()
}
```

## Getting the next token

Even though we have the basic structure, we still need a way to get the next token from our lexer. Let's implement a `next_token` method that:
 - Skips the current char if it's whitespace
 - Reads a number if it's a digit
 - Reads a name if it's a letter
 - Handles opening and closing parentheses
 - Throws an error if any other token is encountered

```
// Inside the impl<'a> Lexer<'a>

pub fn next_token(&mut self) -> Token {
	// Checks if there is a current character
	if let Some(c) = self.current_char() {
	
		if c.is_whitespace() {
			self.advance();
			self.next_token()

		} else if c.is_numeric() {
			Token::Number(self.read_number())
		
		} else if c.is_alphabetic() {
			Token::Name(self.read_name())
			
		} else {
			// Handles miscellaneous single-character tokens
			self.advance();
			match c {
				'(' => Token::OpenParen,
				')' => Token::CloseParen,
				_ => panic!("Unknown character")
			}
		}
	} else {
		// Return EOF if there is no current character
		Token::EOF
	}
}
```

Some of this might be a bit confusing if you're unfamiliar with Rust. I'll try explaining a bit:

```
if let Some(c) = self.current_char() {
    ...
}
```

`if let` is a Rust feature that lets you do pattern matching in `if` expressions. Rust has an `Option` enum that can be one of two types: `Some(value)` and `None`. For example, `self.current_char`  returns a character wrapped in `Some`, if one exists, and `None` otherwise.\

Here we are checking if it returns a value of the form `Some(c)`. We only check for the various token types if a character exists. Otherwise we return the `EOF` token.

```
else {
	Token::EOF
}
```
The code also contains some statements without a terminating semicolon. Rust returns the value of these statements implicitly.

## Reading Numbers and Names

 I've used two undefined methods, `read_number` and `read_name` inside the `next_token` method. These are used to read a number and a name respectively from the input.

*Note that generally a lexer does not convert the token to a specific type. This is generally done by the parser. I'll be changing this in the next post.*

```
// Inside the impl<'a> Lexer<'a>

fn read_number(&mut self) -> i32 {
	let mut num_as_string = String::new();

	// Push characters into num_as_string white they are digits
	while let c @ Some('0'..='9') = self.current_char() {
		num_as_string.push(c.unwrap());
		self.advance();
	}
	num_as_string.parse().unwrap()
}

fn read_name(&mut self) -> String{
	let mut name = String::new();

	// Push characters into name while they are alphabets
	while let c @ Some('A'..='Z' | 'a'..='z') = self.current_char() {
		name.push(c.unwrap());
		self.advance();
	}
	name
}
```

We use `while let` pattern-matching here to loop while the character matches a certain pattern.
`'0'..='9'` indicates the range of ASCII digits, and `'A'..='Z' | 'a'..='z'` matches when the character is an uppercase or lowercase alphabet.

The `parse` method is also used on `num_as_string` to convert it into a 32-bit signed integer (`i32`)

## Testing the Lexer

With that, our lexer is complete. Now it's time to test it. Here's some driver code I wrote to test it:
```
fn main() {
	let mut l = Lexer::new("(println 1 2)");
	
	for _ in 0..=5 {
		println!("{:?}", l.next_token());
	}
}
```

If you're following along in Rust, you'll need to derive the `Debug` trait for the `Token` enum like so:
```
#[derive(Debug)]
enum Token {
	...
}
```

And then, on compiling and running the program, you should see
```
OpenParen
Name("println")
Number(1)
Number(2)
CloseParen
EOF
```

If you get a similar output, congratulate yourself, because you've just built a working lexer! 🥳

---

# ASTs and Parsing

## The Abstract Syntax Tree

We have a way of converting the text to tokens. Next, we're going to parse our tokens into an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree), or AST for short. This will represent the structure of our program.

Our AST will have three kinds of nodes:
```
pub enum AstNode {
	Number(i32),
	Name(String),
	Expr(Vec<AstNode>)
}
```

You might have already guessed what each node does:
 - `Number`: Holds a number
 - `Name`: Holds a name
 - `Expr`: Holds the terms of a LISP expression

## Making a Parser

The parser is responsible for reading the tokens from the lexer, and generating an AST. Just like the lexer, this will be a `struct` too. I'll also add a `current_token` field to store the current lexer token.
```
struct Parser<'a> {
	lexer: Lexer<'a>,
	current_token: Token
}

impl<'a> Parser<'a> {
	pub fn new(mut lexer: Lexer<'a>) -> Self {
		Self {
			current_token: lexer.next_token(),
			lexer
		}
	}
}
```

We'll also need two other methods - one to advance the parser, and another to *expect a token type*.

```
/// Checks if two enum members have the same variant
fn variant_eq<T>(a: &T, b: &T) -> bool {
	std::mem::discriminant(a) == std::mem::discriminant(b)
}

// Inside the impl<'a> Parser<'a>

fn advance(&mut self) {
	self.current_token = self.lexer.next_token();
}

fn expect(&mut self, token_type: Token) {
	if !variant_eq(&self.current_token, &token_type) {
		panic!("Did not find expected token");
	}
	self.advance();
}
```

The `expect` method also uses a separate `variant_eq` function. `variant_eq` compares if two members of an `enum` have the same variant, irrespective of value. So `variant_eq(&Number(1), &Number(2))` returns `true`.

## Constructing ASTs

Remember [the grammar](#what-is-a-lisp-and-why-should-i-write-one) from earlier? This section is mainly about how to parse it. Let's start with the simplest type - an atom. This can hold either a `Number`, or a `Name`. Let's make a function to parse it:

```
// Inside the impl<'a> Parser<'a>

fn parse_atom(&mut self) -> AstNode {

	let node = match &self.current_token {
		Token::Number(num) => AstNode::Number(*num),
		Token::Name(name) => AstNode::Name(name.clone()),
		_ => panic!("Expected Number or Name") 
	};
	
	self.advance();
	return node;
}
```

That should be pretty simple - It returns a `Number` node if the token is a `Number`, a `Name` if it is a `Name`, and panics if it's any other kind.

Next, we have to handle  the `expr`. This isn't that tough either. An `expr` can contain either an atom or a list. Since we know that a `list` always starts with an opening parenthesis, we only need to check for list in the case that our expression begins with a `(`.

```
// Inside the impl<'a> Parser<'a>

pub fn parse_expr(&mut self) -> AstNode {
	match &self.current_token {
		Token::OpenParen => AstNode::Expr(self.parse_list()),
		_ => self.parse_atom()
	}
}
```

`parse_list` is a bit more complicated, We have to read tokens until we hit either EOF, or a closing parenthesis. This method is also intended to be called from other `Parser` methods, not directly. 

```
// Inside the impl<'a> Parser<'a>

fn parse_list(&mut self) -> Vec<AstNode> {
	self.expect(Token::OpenParen)
	let mut elements: Vec<AstNode> = Vec::new();

	
	while self.current_token != Token::EOF && self.current_token != Token::CloseParen {
		elements.push(self.parse_expr());
	}

	self.expect(Token::CloseParen);
	elements
}
```

Since we're comparing `Token`s in this method, we'll need to `derive` the `PartialEq` trait for our `Token` struct:

```
// We derived Debug earlier for testing
#[derive(Debug, PartialEq)]
struct Token {
	...
}
```

## Testing the Parser

That's it! The parser's done, and now we need to test it. First `derive` the `Debug` trait for your `AstNode`...
```
#derive(Debug)
pub enum AstNode {
	...
}
```
...then add this driver code in your `main` function:
```
fn  main() {
	let  lexer  =  Lexer::new("(println 1 2)");
	let  mut  parser  =  Parser::new(lexer);

	println!("{:?}", parser.parse_expr());
}
```

Compiling and running this should give you:
```
Expr([Name("println"), Number(1), Number(2)])
```

If you see something similar, that means your parser is working. Yet another thing to celebrate!

However, that's also the end of this post. Keep an eye out for the next one, where we'll optimize the lexer, and make an interpreter for our ASTs.