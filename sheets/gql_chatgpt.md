# Asked ChatGPT
## what are terminal symbols

In the context of formal grammars and language theory, terminal symbols are the basic units of a language that cannot be further divided within the context of the grammar. They are the leaf nodes of the syntactic tree, representing the fundamental building blocks or tokens of a language.

Terminal symbols are contrasted with non-terminal symbols, which can be expanded into other symbols. Non-terminals are placeholders for patterns that can be further decomposed according to the rules of the grammar.

In formal grammar notations such as Backus-Naur Form (BNF) or Extended Backus-Naur Form (EBNF), terminal symbols are often represented by literals, characters, or regular expressions. Here's a simple example in EBNF:

```ebnf
expression ::= term ('+' term)*
term       ::= NUMBER
NUMBER     ::= [0-9]+
```

In this example:

 `NUMBER` is a terminal symbol, representing one or more digits.

In the context of programming languages:

- Keywords, identifiers, literals (such as numbers or strings), and punctuation symbols are common examples of terminal symbols.
- In a programming language grammar, a terminal symbol could represent a specific keyword (e.g., `if`, `else`), an identifier (variable or function names), a literal value (e.g., `42`, `"hello"`), or a punctuation symbol (e.g., `{`, `}`, `;`).

In the syntactic analysis of a program, the lexer (or lexical analyzer) identifies and categorizes the input characters into terminal symbols. These terminal symbols are then used by the parser to build the syntactic structure of the program according to the rules specified in the grammar.
