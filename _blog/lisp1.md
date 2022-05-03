---
layout: default
title: Write You a Lisp - Part 1
date: 2022-05-03
---

I've decided to learn more about how compilers work. I'm starting by trying to create
my own programming language - a simple Lisp. I'll be implementing it in Rust, but feel
free to follow along in whatever language you like.
<!--more-->

I've you haven't already read Ruslan Spivak's blog post series on building an interpeter,
I highly suggest you read it [here](https://ruslanspivak.com/lsbasi-part1/ "Ruslan Spivak - Let's Build a Simple Interpreter, Part 1")

(I'm still pretty new to Rust, so my code might not be as good as a veteran Rustacean's.)
## What is a Lisp, and Why should I write one?

Lisp is a very old family of Programming Languages; As of the time of writing, it is 64 years old; that's 14 years older than C! It's responsible for pioneering many features which we see today, such as dynamic typing, first-class functions, and even recursion and conditionals.

In addition to this, Lisps have a very simple grammar, that can be parsed easily. You can express it's grammar in [BNF notation](Ahttps://en.wikipedia.org/wiki/Backus–Naur_form) like so:

```bnf
; an expression can be either an atom (a fundamental type) or a list
expr ::= atom | list

; an atom can be either a number, a name (or symbol) or a string
atom ::= number | name | string

; a list consists of zero or more expressions between a pair of parentheses
list ::= '(' expr* ')'
```

## Writing the Lexer

### The Tokens

The lexer is the first part of our interpreter. It splits a string into "tokens", each of which has an assigned meaning. For now, our Lisp will have five token types:
 - `Number`: A positive integer made from consecutive digits
 - `Name`: Consecutive letters representing a variable or function
 - `OpenParen` and `CloseParen`: represent opening and closing parentheses (`(` and `)`)
 - `EOF`: Used to indicate that no more characters can be read from the input

I'll be implementing this as a Rust `enum`:
```rust
pub enum Token {
	Number(i32),
	Name(String),
	OpenParen,
	CloseParen,
	EOF
}
```

### Designing the Lexer

Now that we have our tokens, we need a way to split a string and generate this Tokens. I'll use a `struct` called `Lexer` that does this. I'll also implement a `new` method that takes a string as an argument and returns a `Lexer`.

```rust
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

```rust
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

### Getting the next token

Even though we have the basic structure, we still need a way to get the next token from our lexer. Let's implement a `next_token` method that:
 - Skips the current char if it's whitespace
 - Reads a number if it's a digit
 - Reads a name if it's a letter
 - Handles opening and closing parentheses
 - Throws an error if any other token is encountered

```rust
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
			Token::Name(self.read_name)
			
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

### Explanation

Some of this might be a bit confusing if you're unfamiliar with Rust. I'll try explaining a bit:

```rust
if let Some(c) = self.current_char() {
    ...
}
```

`if let` is a Rust feature that lets you do pattern matching in `if` expressions. Rust has an `Option` enum that can be one of two types: `Some(value)` and `None`. For example, `self.current_char`  returns a character wrapped in `Some`, if one exists, and `None` otherwise.\

Here we are checking if it returns a value of the form `Some(c)`. We only check for the various token types if a character exists. Otherwise we return the `EOF` token.

```rust
else {
	Token::EOF
}
```
The code also contains some statements without a terminating semicolon. Rust returns the value of these statements implicitly.

### Reading Numbers and Names

You might have noticed that I've used two undefined methods, `read_number` and `read_name` inside the `next_token` method. These are used to read a number and a name respectively from the input.

```rust
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

### Testing the Lexer

With that, our lexer is complete. Now it's time to test it. Here's some driver code I wrote to test it:
```rust
fn main() {
	let mut l = Lexer::new("(println 1 2)");
	
	for _ in 0..=5 {
		println!("{:?}", l.next_token());
	}
}
```

If you're following along in Rust, you'll need to derive the `Debug` trait for the Token enum like so:
```rust
#[derive(Debug)]
enum Token {
	...
}
```

And then, on compiling and running the program, you should see
```rust
OpenParen
Name("println")
Number(1)
Number(2)
CloseParen
EOF
```

If you get a similar output, congratulate yourself, because you've just built a working lexer! 🥳

---

Coming soon: ASTs and Parsing