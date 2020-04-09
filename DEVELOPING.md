# Developing

This document gives some pointers and tips for those interested in contributing to this project.

If you notice that this document is outdated, or if something isn't clear to you, please [open an issue](https://github.com/solidity-parser/parser/issues/new)!

## Architecture

This package is a wrapper around a parser generated by [ANTLR](https://www.antlr.org/). This makes it hard to describe the components, since there are two parsers: the one exported by this project, and the one generated by ANTLR. To avoid confusion, we will call "parser" to the one exported by this project, and "generated parser" to the one generated by ANTLR.

The grammar description used as input for ANTLR lives in [this repo](https://github.com/solidity-parser/antlr), which is included here as a git submodule. The grammar file is `Solidity.g4`. This means that some issues can be fixed just in this repository, while others depend on updates to the ANTLR grammar. In the latter case, the fix usually involves updating the grammar, updating the submodule in this repository, updating the generated parser and probably updating the parser to handle these changes.

For example, an issue like "the list of state variables in the generated AST code should be in the same order than in the solidity file" probably can be done by just updating the code in the parser. On the other hand, an issue like "support this new keyword" will need an update to the grammar.

The entry point of the project is [`src/index.js`](src/index.js), but most of the important logic is in [`src/ASTBuilder.js`](src/ASTBuilder.js). `ASTBuilder` is a [visitor](https://en.wikipedia.org/wiki/Visitor_pattern) object that traverses the AST returned by teh generated parser, and produces the AST that will be returned by the parser.

## Setup

```bash
# clone the repository
git clone git@github.com:solidity-parser/parser.git solidity-parser

# initialize the submodule
cd solidity-parser
git submodule update --init
```

## Updating the grammar

The [`scripts/antlr.sh`](scripts/antlr.sh) script should download ANTLR and generate the grammar. For this to work, you
need Java (1.6 or higher) installed.

## Updating the parser

Run `yarn build` to build the parser.

## Quick testing

Right now the easier way to test something is to start the node REPL and import the project:

```
yarn build
node
> parser = require('.')
> ast = parser.parse('contract Foo {}')
> console.log(ast.children[0].name)
```

## Understanding the generated parser

The AST returned by the generated parser is very low level and tightly coupled to the grammar definition, that's why
using this wrapper is much easier.  But if you want to contribute to this parser, you need to deal with this low level
representation. Let's see some examples of how to interpret it.

### Example 1: PragmaDirective

At the moment of writing this, the function that handles pragma directives in `ASTBuilder` is this one:

```javascript
  PragmaDirective(ctx) {
    return {
      name: toText(ctx.pragmaName()),
      value: toText(ctx.pragmaValue())
    }
  },
```

In this function, we are in the context of a pragma directive. In `Solidity.g4`, a pragma directive is defined like
this:

```
pragmaDirective : 'pragma' pragmaName pragmaValue ';' ;
```

This means that the context will have a `pragmaName` and a `pragmaValue` function. Since we are only interested in their
verbatim content, we use the `toText` helper to obtain their values.

### Example 2: FunctionCall

Let's look now at how `FunctionCall` is handled:

```javascript
  FunctionCall(ctx) {
    let args = []
    const names = []

    const ctxArgs = ctx.functionCallArguments()
    if (ctxArgs.expressionList()) {
      args = ctxArgs
        .expressionList()
        .expression()
        .map(exprCtx => this.visit(exprCtx))
    } else if (ctxArgs.nameValueList()) {
      for (const nameValue of ctxArgs.nameValueList().nameValue()) {
        args.push(this.visit(nameValue.expression()))
        names.push(toText(nameValue.identifier()))
      }
    }

    return {
      expression: this.visit(ctx.expression()),
      arguments: args,
      names
    }
  },
```

This function is a little more involved. The relevant parts of the grammar are:

```
functionCall : expression '(' functionCallArguments ')' ;

functionCallArguments
  : '{' nameValueList? '}'
  | expressionList? ;

nameValueList
  : nameValue (',' nameValue)* ','? ;

nameValue
  : identifier ':' expression ;
```

Let's focus on the return. Here we are returning our parser's representation of a `FunctionCall` node. We decide to
return the expression that represents the function called (and for this we recursively use `this.visit`), along with the
names and values of the arguments. `names` only makes sense when using the `f({a: 1})` syntax, otherwise it will be
empty. If the `f(1, 2)` syntax is used, then `ctxArgs.expressionList()` will be truthy, and then we visit each expression in
turn to generate the list of arguments. Otherwise, if `ctxArgs.nameValueList()` is truthy, we traverse the list of
`nameValue`s and use their `identifier`s as argument names and their `expression`s as values.

## Using ANTLR for visualizing the AST

Sometimes is helpful to visualize the AST generated by the ANTLR parser. For example, let's say we are interested in
this contract:

```
contract Example is Base1, Base2(0) {}
```

Go to the `antlr` directory and run:

```
./build.sh
echo 'contract Example is Base1, Base2(0) {}' > example.sol
java -classpath antlr4.jar:target/ org.antlr.v4.gui.TestRig Solidity sourceUnit -gui < example.sol
```

And you'll get a window where you can see an inspect the generated tree.
