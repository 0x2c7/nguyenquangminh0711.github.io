---
title:  "What is special inside rubocop?"
date:   2017-08-21
layout: post
draft: 1
description: "Rubocop is a popular linter and code analyzer for Ruby. It helps us prevent the code smells and unify the code styles in the whole code base. So, how does it produce such magic? Let's find out in my blog post: What is special inside rubocop?"
---

## Introduction
- What is Rubocop?
	+ Ruby static code analyzer. Enforce many of the guidelines outlined in the community Ruby Style Guide
	+ In other words, Rubocop analyze your code and point out the bad code formats, encourage best practices and alert obvious code problems.
	+ The `static code analyzer` part means that Rubocop works on the source code only, it doesn't requires the code to actually run to point out the problem.
	+ Demonstrate how Rubocop works with a simple example. Run and show the output
```ruby
def greet(message, language)
  puts "#{message}"
end
```
	- Point out the popularity of Rubocop
		- Used widely in many companies
		- Used as a check before merging into production
		- Even native supported in Rubymine
- Raise the question how Rubocop produce its magic from a dead simple source code
## Grammar and AST
- Raise the question of how Ruby is defined
- Similar to any other programming language, or even a real-world language, Ruby has a specific grammar. The compilers / interpreters depends on this grammar to "understand" and execute your source code.
- For example, this is a sample grammar
```
binary_operation: arg tPLUS arg
arg: | literal
     | string
     | array
literal: | numeric
         | symbol
numeric: tINTEGER
```
- Yes, perhaps the first expression when you look at this grammar is WTF is this?  Each line is a definition of something. `tPLUS` is a token. A token is a meaningful sequence of characters used in a programming language. In this case, it is `+` character. So, the first line tells us that a binary operation has the form `arg + arg`. The definition of arg is on the line 2. `arg` could be a literal, string or array. The definitions of string and array are omitted, we only mention the definition of literal in line 3. A literal could be numeric or symbol, and a numeric could be a `tINTERGER` token. `tINTEGER` indicates any valid integer, like `1`, `123`, or `12351235`.
- Based on the above grammar, the interpreters or compilers understand this little code: `1 + 1` is a binary operation.
- A little bit complicated right? Since a grammar is the back-bone of a programming language, it must be abstract enough to consist of all cases.
- Compilers / interpreters parses the source code into a big tree based on the grammar. However, it is too complicated and consists of a lot of overheads (Ain't everybody got time for that meme here)
-  So, people thinks of a grand new, but still powerful data structure, called Abstract Syntax Tree. For example, `1 + 1` operation could be demonstrated by a tree:
```
[Binary operation node]
          |
  -----------------
  |       |       |
[num]   [ops]   [num]
  |       |       |
 [1]     [+]     [1]
```
- It is much more simpler right? AST depends on the implementation of each compilers / interpreters, there won't be any standard for it. It could be much much more simpler or much more complicated depends on the intention of that compilers / interpreters.
- Usually, a grammar definition file consists of how a AST is built from the grammar. And to make everything easy, the grammar definition is usually human - friendly. A real parser is generated from this grammar file by a parser generator, for example: [Bison- GNU Project - Free Software Foundation](https://www.gnu.org/software/bison/). The generated parser is the one who takes responsibility to parse the source code.
- I won't go into the detail of this since it is endless. You could research more by reading any book about interpreters and compilers (recommend the dragon book)
## How does Rubocop parse your source code?
- Internally, Ruby has its own parser and grammar. But it is written in C and not encouraged to used outside of the Ruby MRI Core.
- Rubocop depends on this gem [GitHub - whitequark/parser: A Ruby parser.](https://github.com/whitequark/parser) of
- The gem author is a genius. I highly recommend to read the source code of this gem. This gem does a lot of things. It uses [GitHub - tenderlove/racc: Racc is an LALR(1) parser generator.  It is written in Ruby itself, and generates ruby programs.](https://github.com/tenderlove/racc) of Tender Love sensei as the main parser generator .
	- Grammars for Ruby from 1.8 to 2.4 (`lib/ruby18.y`, `lib/ruby19.y`, etc.)
	- Extensible and abstract AST builder
	- Powerful callback-based processor to traverse through the AST
	- Source code re-writing
	- Syntax dianostic
	...
- Rubocop extends the parser gem: add helpers to AST nodes depends on its types, implement a custom processor, etc.
## How does Rubocop use AST to analyze your source code?
- Rubocop uses AST to process its magics.
- Use this example.
	- This code snippet has two smells that the parameters should not have more than 5 parameters
	- `RuntimeError.new('Failed to check')` is not encourage, `raise 'Failed to check' is enough`.
```ruby
def check_something(a, b, c, d, e, f)
  result = a + b + c + d + e + f
  raise RuntimeError.new('Failed to check') if result > 0
end
```
- At the beginning before processing a file, Rubocop parses your code into a full flex AST tree.
```
s(:def, :check_something,
  s(:args,
    s(:arg, :a),
    s(:arg, :b),
    s(:arg, :c),
    s(:arg, :d),
    s(:arg, :e),
    s(:arg, :f)),
  s(:begin,
    s(:lvasgn, :result,
      s(:send,
        s(:send,
          s(:send,
            s(:send,
              s(:send,
                s(:lvar, :a), :+,
                s(:lvar, :b)), :+,
              s(:lvar, :c)), :+,
            s(:lvar, :d)), :+,
          s(:lvar, :e)), :+,
        s(:lvar, :f))),
    s(:if,
      s(:send,
        s(:lvar, :result), :>,
        s(:int, 0)),
      s(:send, nil, :raise,
        s(:send,
          s(:const, nil, :RuntimeError), :new,
          s(:str, "Failed to check"))), nil)))
```
- Each `s` method is a node, first argument is node type, the rest is its children. The above structure is a presentation of this tree: [insert diagram here]. From now on, I'll use that presentation.
- After that, Rubocop starts a processor. It traverses the generated AST recursively top-down and left-right
- Rubocop structures all the checks in units called `Cop`. Each cop is a listener of the AST traversing process based on node type. Whenever the processor reaches a node of type x, Rubocop finds all the Cop that listens to event `on_x`  and gets the checking results from that Cop.
- On above example, it is obvious that every method definition node has a child node with type `args` that manages the method arguments. This node has multiple children, each child is an argument of the method from left to right. We just simply observe the even `on_args` and checks for the number of children of the args node . The check will be something like this (the real implementation has many edge cases, I just put the simplest case)
```ruby
      class ParameterLists < Cop
        def on_args(node)
          argument_count = node.children.length
          if argumnet_count
            add_offense(node, "Number of arguments is too high")
          end
        end
      end
```
- Note that offense here means one style violation. It contains some meta data such as the violated node, messages, etc.
- The second check is more complicated. It is obvious that we have to catch `raise` method call, then check whether the argument of the `raise` call is an object allocation of RuntimeError node or node. So, the implementation is something like this:

```ruby
class RaiseArgs < Cop
  # All method call is presented by a send node
  def on_send(node)
    # Only process raise method call
    return unless node.children[1] != :raise
    raise_argument = node.children[2]
    if run_time_error_allocation?(raise_argument)
      add_offense(node, "Dont need to allocate a RuntimeError object")
    end
  end

  private

  def run_time_error_allocation?(raise_argument)
    first_child = raise_argumnet.children.first
    raise_argument.type == :send &&
      first_child.type == :const &&
      first_child.children[1] == :RuntimeError &&
      raise_argument.children[1] == :new
  end
end
```
- Yup, it actually works. Rubocop supports hundreds of Cops and many helpers to support the checking process. Whatever it does, it still depends on the AST. AST is a really powerful tool
## NodePattern
## How is Rubocop able to correct a violated style?
## Conclusion
