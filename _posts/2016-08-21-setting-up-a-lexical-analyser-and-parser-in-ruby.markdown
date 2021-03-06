---
layout: post
title: "Setting up a lexical analyser and parser in Ruby"
date: 2016-08-21 12:36:05 +0530
comments: true
categories: 
---

I wrote this post as I was setting up the lexer and parser for Rubex, a new superset of Ruby that I'm developing.

Let's demonstrate the basic working of a lexical analyser and parser in action with a demonstration of a very simple addition program. Before you start, please make sure [rake](https://github.com/ruby/rake), [oedipus_lex](https://github.com/seattlerb/oedipus_lex) and [racc](https://github.com/tenderlove/racc) are installed on your computer.

### Configuring the lexical analyser

The most fundamental need of any parser is that it needs string tokens to work with, which we will provide by way of lexical analysis by using the [oedipus_lex](https://github.com/seattlerb/oedipus_lex) gem (the logical successor of [rexical](https://github.com/tenderlove/rexical)). Go ahead and create a file `lexer.rex` with the following code:

``` ruby
class AddLexer
macro
  DIGIT         /\d+/
rule
  /#{DIGIT}/    { [:DIGIT, text.to_i] }
  /.|\n/        { [text, text] }
inner
  def do_parse; end # this is a stub.
end # AddLexer
```

In the above code, we have defined the lexical analyser using Oedipus Lex's syntax inside the `AddLexer` class. Let's go over each element of the lexer one by one:

**macro**

The macro keyword lets you define macros for certain regular expressions
that you might need to write repeatedly. In the above lexer, the macro `DIGIT` is a regular expression (`\d+`) for detecting one or more integers. We place the regular expression inside forward slashes (`/../`) because oedipus_lex requires it that way. The lexer can handle any valid Ruby regular expression. See the Ruby docs for details on Ruby regexps.

**rule**

The section under the `rule` keyword defines your rules for the lexical analysis. Now it so happens that we've defined a macro for detecting digits, and in order to use that macro in the rules, it must be inside a Ruby string interpolation (`#{..}`). The line to the right of the `/#{DIGIT}/` states the action that must be taken if such a regular expression is encountered. Thus the lexer will return a Ruby Array that contains the first element as `:DIGIT`. The second element uses the `text` variable. This is a reserved variable in lex that holds the text that the lexer has matched. Similar the second rule will match any character (`.`) or a newline (`/n`) and return an `Array` with `[text, text]` inside it.

**inner**

Under the `inner` keyword you can specify any code that you want to occur inside your lexer class. This can be any logic that you want your lexer to execute. The Ruby code under the `inner` section is copied as-is into the final lexer class. In the above example, we've written an empty method called `do_parse` inside this section. This method is mandatory if you want your lexer to sucessfully execute. We'll be coupling the lexer with `racc` shortly, so unless you want to write your own parsing logic, you should leave this method empty.

### Configuring the parser

In order for our addition program to be successful, it needs to know what to do with the tokens that are generated by the lexer. For this purpose, we need [racc](), an LALR(1) parser generator for Ruby. It is similar to yacc or bison and let's you specify grammars easily.

Go ahead and create a file called `parser.racc` in the same folder as the previous `lexer.rex` and `Rakefile`, and put the following code inside it:

``` ruby
class AddParser
rule
  target: exp { result = 0 }
  
  exp: exp '+' exp { result += val[2]; puts result }
     | DIGIT
end

---- header
require_relative 'lexer.rex.rb'

---- inner
def next_token
  @lexer.next_token
end

def prepare_parser file_name
  @lexer = AddLexer.new
  @lexer.parse_file file_name
end
```

As you can see, we've put the logic for the parser inside the `AddParser` class. Yacc's `$$` is the `result`; `$0`, `$1`... is an array called `val`, and `$-1`, `$-2`... is an array called `_values`. Notice that in racc, only the parsing logic exists inside the class and everything else (i.e under `header` and `inner`) exists _outside_ the class. Let's go over each part of the parser one by one:

**class AddParser**

This is the core class that contains the parsing logic for the addition parser. Similar to `oedipus_lex`, it contains a `rule` section that specifies the grammar. The parser expects tokens in the form of `[:TOKEN_NAME, matched_text]`. The `:TOKEN_NAME` must be a symbol. This token name is matched to literal characters in the grammar (`DIGIT` in the above case). `token` and `expr` are varibles. Have a look at [this introduction to LALR(1) grammars](https://en.wikipedia.org/wiki/LALR_parser) for further information.

**header**

The `header` keyword tells racc what code should be put at the top of the parser that it generates. You usually put your `require` statements here. In this case, we load the lexer class so that the parser can use it for accessing the tokens generated by the lexer. Notice that `header` has 4 hyphens (`-`) and a space before it. This is mandatory if your program is to not malfunction.

**inner**

The `inner` keyword tells racc what should be put _inside_ the generated parser class. As you can see there are two methods in the above example - `next_token` and `prepare_parser`. The `next_token` method is mandatory for the parser to function and you must include it in your code. It should contain logic that will return the next token for the parser to consider. Moving on the `prepare_parser` method, it takes a file name that is to be parsed as an argument (how we pass that argument in will be seen later), and initialzes the lexer. It then calls the `parse_file` method, which is present in the lexer class by default.

The `next_token` method in turn uses the `@lexer` object's `next_token` method to get a token generated by the lexer so that it can be used by the parser.

### Putting it all together

Our lexical analyser and parser are now coupled to work with each other, and we now use them in a Ruby program to parse a file. Create a new file called `adder.rb` and put the following code in it:

``` ruby
require_relative 'parser.racc.rb'

file_name = ARGV[0]
parser = AddParser.new
parser.prepare_parser(file_name)
parser.do_parse
```

The `prepare_parser` is the same one that was defined in the `inner` section of the `parser.racc` above. The `do_parse` method called on the parser will signal the parser to start doing it's job.

In a separate file called `text.txt` put the following text:

```
2+2
```

Oedipus Lex does not have a command line tool like rexical for generating a lexer from the logic specified, but rather has a bunch of rake tasks defined for doing this job.
So now create a `Rakefile` in the same folder and put this code inside it:

``` ruby
require 'oedipus_lex'

Rake.application.rake_require "oedipus_lex"

desc "Generate Lexer"
task :lexer  => "lexer.rex.rb"

desc "Generate Parser"
task :parser => :lexer do
  `racc parser.racc -o parser.racc.rb`
end
```

Running `rake parser` will generate a two new files - `lexer.rex.rb` and `parser.racc.rb` - which will house the classes and logic for the lexer and parser, respectively. You can use your newly written lexer + parser with a `ruby adder.rb text.txt` command. It should output `4` as the answer.

You can find all the code in this blogpost [here](https://github.com/v0dro/scratch/tree/master/lexer_parser).