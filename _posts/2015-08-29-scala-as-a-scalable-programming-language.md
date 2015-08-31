---
layout:     post
title:      "Scala as a Scalable Programming Language"
date:       2015-08-29 11:09:00
summary:    "How I implemented an external domain-specific language with a parser, interpreter, and Java bytecode compiler."
categories: scala
---

I love the Scala programming language. For those who do not know much about the programming language, Scala is a statically typed, object-oriented and functional JVM language. Some of the popular tools and libraries built with Scala are Apache Spark, Apache Kafka, Apache Samza, and, a personal favourite, Akka.

Loving the programming language, I recently wanted to explore the scalability of the programming language itself in the context of metaprogramming as I loved the reflection capabilities introduced in Scala 2.10 while I was excited about the news about [scala.meta](http://scalameta.org/). Thus, I decided to work on a small project on a topic which I found little information: implementing an external domain-specific language,

For the small project, I choose to implement the WHILE programming language in Scala with a parser, an interpreter, and a Java bytecode compiler, all of which functioned during runtime. The reason why I decided to implement the WHILE programming language was its simplicity as a subject created for the book [*Principles of Program Analysis*](http://www.amazon.ca/Principles-Program-Analysis-Flemming-Nielson/dp/3540654100) (which I must address I have not read).

## What is WHILE?

The WHILE programming language is an imperative programming language which computes upon boolean and numerical values. An example of a WHILE program is the following:

```
BEGIN
variable1 := 1.0;
variable2 := 2.0;
IF variable2 > variable1 THEN
  WRITE true
ELSE
  WRITE false
END
```

This WHILE program would execute as the following steps:

1. Assign a variable of name `variable1` with a numerical value of `1.0`.
2. Assign a variable of name `variable2` with a numerical value of `2.0`.
3. Compare whether the value of `variable1` is greater than `variable2`.
4. If the value of `variable1` is greater than the value of `variable2`, print `true`.
5. If the value of `variable1` is not greater than the value of `variable2`, print `false`.

For whom it may interest, the Backus-Naur Form (BNF) grammar of the WHILE programming language is the following:

```
a   ::= x | n | - a | a opa a

b   ::= TRUE | FALSE | NOT b | b opb b | a opr a

opa ::= + | - | * | /

opb ::= AND | OR

opr ::= == | != | > | < | >= | <=

S   ::= x := a | SKIP | S1; S2 | IF b THEN S1 ELSE S2 | WHILE b DO S | WRITE a | WRITE b | READ x

P   ::= BEGIN S END
```

## How to implement WHILE in Scala?

As mentioned previously, I wanted to implement the WHILE programming language in Scala complete with a parser, an interpreted, and a Java bytecode compiler. The following sections will explore how achieved each part.

### Project Settings

For this project, I decided to utilize SBT to manage my dependencies. Accordingly, the project was configured with the following code:

```scala
object Dependencies {
  // Versions:
  val SCALA_PARSER_COMBINATOR_VERSION = "1.0.4"
  val SCALA_COMPILER_VERSION = "2.11.7"
  val SCALA_REFLECT_VERSION = "2.11.7"
  val SCALA_LIBRARY_VERSION = "2.11.7"
  val SCALATEST_VERSION = "2.2.4"

  // Libraries:
  val scalaParserCombinators = "org.scala-lang.modules" %% "scala-parser-combinators" % SCALA_PARSER_COMBINATOR_VERSION
  val scalaCompiler          = "org.scala-lang"          % "scala-compiler"           % SCALA_COMPILER_VERSION
  val scalaReflect           = "org.scala-lang"          % "scala-reflect"            % SCALA_REFLECT_VERSION
  val scalaLibrary           = "org.scala-lang"          % "scala-library"            % SCALA_LIBRARY_VERSION
  val scalatest              = "org.scalatest"          %% "scalatest"                % SCALATEST_VERSION % "test"

  // Project Dependencies:
  val projectDependencies = Seq(scalaCompiler, scalaReflect, scalaLibrary, scalaParserCombinators, scalatest)
}
```

```scala
object BlogScalaDslBuild extends Build {
  import Dependencies._

  lazy val project = Project(id = "BlogScalaDsl", base = file("."))
    .settings(organization := "com.github.jparkie")
    .settings(name := "BlogScalaDsl")
    .settings(scalaVersion := "2.11.7")
    .settings(libraryDependencies ++= projectDependencies)
}
```

This SBT project required four dependencies: `scala-parser-combinators`, `scala-compiler`, `scala-reflect`, and `scala-library`. All of these four dependencies were actually packaged with the official Scala SDK, yet since Scala 2.11, they have been separated into their own packages.

1. The `scala-parser-combinators` dependency was utilized to implement the parser for the WHILE programming language. 
2. The `scala-compiler` and `scala-reflect` dependencies were utilized to implement the compiler. 
3. The `scala-library` dependency was utilized to ensure the prior dependencies functioned appropriately.

The full project settings can be found at the following link: [GitHub Link](https://github.com/jparkie/WhileScalaDsl/tree/master/project).

### Establishing the Abstract Syntax Tree

Before I could implement the parser, the interpreter, and the compiler, I needed to model the WHILE programming language in Scala for them to function upon. As a result, I established the abstract syntax tree for the WHILE programming language based on the previously defined BNF grammar.

First, I created a trait named `WhileAbstractSyntaxTreeNode` as the base type for all the other nodes.

```scala
trait WhileAbstractSyntaxTreeNode[T] {
  def nodeName: String

  def evaluate()(implicit whileEnvironment: WhileEnvironment): Try[T]

  def asWhile: Try[String]

  def asScala: Try[String]
}
```

Because I wanted this base type to model the WHILE programming language for the parser, the interpreter, and the compiler, I exposed the following three methods: `evaluate()`, `asWhile()`, and `asScala()`.

- The `evaluate()` method was exposed to allow the interpreter to function upon the AST by traversing it through this method during interpretation.
- The `asWhile()`  method was exposed to allow the interoperability between the AST generated by the parser and the actual WHILE programming language.
- The `asScala()` method was exposed to allow the compiler to function upon the AST to create a Scala AST for compilation into Java bytecode.

An example of one of the grammars of the WHILE programming language modeled as a node of the AST is the following `NOT` operator.

```scala
case class WhileNotOperator(unaryArgument: WhileBooleanType) extends WhileBooleanType {
  override def nodeName: String = classOf[WhileNotOperator].getSimpleName

  override def evaluate()(implicit whileEnvironment: WhileEnvironment): Try[Boolean] = {
    val unaryIdentityTry = unaryArgument.evaluate()

    for {
      unaryIdentity <- unaryIdentityTry
    } yield !unaryIdentity
  }

  override def asWhile: Try[String] = {
    val unaryArgumentWhileTry = unaryArgument.asWhile

    for {
      unaryArgumentWhile <- unaryArgumentWhileTry
    } yield s"${WhileKeywords.`NOT`} $unaryArgumentWhile"
  }

  override def asScala: Try[String] = {
    val unaryArgumentScalaTry = unaryArgument.asScala

    for {
      unaryArgumentScala <- unaryArgumentScalaTry
    } yield s"!($unaryArgumentScala)"
  }
}
```

The full AST can be found at the following link: [GitHub Link](https://github.com/jparkie/WhileScalaDsl/tree/master/src/main/scala/com/github/jparkie/bsd).

### Creating the Parser

With the AST established, I created the parser utilizing the `scala-parser-combinators` dependency. Why I utilized this dependency was it ease of translating the previously defined BNF grammar of the WHILE programming language into Scala with a parallel, internal Scala DSL which operates upon String input. Information about the dependency can be found at the following link: [https://github.com/scala/scala-parser-combinators](https://github.com/scala/scala-parser-combinators).

For those who do not know what a parser combinator function is, I explain it as a higher-order function which operates on an input of parsers to output a new parser. Sadly, a tutorial regarding the use of this library is beyond the scope of this article. Nonetheless, I recommend reading [*Programming in Scala*](http://www.artima.com/shop/programming_in_scala_2ed)’s “33. Combinator Parsing”.
Accordingly, utilizing the `scala-parser-combinators` dependency, I created a `WhileParser` object as a composition of various orders of parsers representing the various elements of the BNF grammar of the WHILE programming language as depicted below.

```scala
object WhileParser extends RegexParsers {
  import ExpressionGrammar._
  import LexicalGrammar._
  import LiteralGrammar._
  import OperatorGrammar._
  import ProgramGrammar._
  import StatementGrammar._
  import VariableGrammar._
  
  val TrySuccess = scala.util.Success
  val TryFailure = scala.util.Failure

  def parseWhileScript(whileScript: String): Try[WhileProgram] = {
    parseAll(whileProgram, whileScript) match {
      case Success(actualWhileProgram, _) => TrySuccess(actualWhileProgram)
      case Failure(errorMessage, _) => TryFailure(WhileParseException(errorMessage))
    }
  }

  //
  // ...
  //
}
```

Nonetheless, for those of whom are familiar with parser combinator functions, the following explains how this parser was created.

First I utilized the `RegexParser` to define the lexical units of the WHILE programming language. These units included regular expressions to parse boolean values, numerical values, and variable identifiers.

```scala
object LexicalGrammar {
  lazy val lexicalIdentifier: Parser[String] = WhileRegex.IDENTIFIER ^^ (_.toString)
  
  lazy val lexicalBoolean: Parser[Boolean] = WhileRegex.BOOLEAN ^^ (_.toBoolean)
  
  lazy val lexicalNumber: Parser[Double] = WhileRegex.NUMBER ^^ (_.toDouble)
}
```

From this point onwards, it was a simple process of composing these lexical unit parsers by following the BNF grammar of the WHILE programming language whose `Parser`s were of types extending `WhileAbstractSyntaxTreeNode`.

A sample of this process is the grammar for an assignment statement as depicted below.

```scala
lazy val assignmentStatement: Parser[WhileStatementType] = lexicalIdentifier ~ WhileKeywords.`:=` ~ numberExpression ^^ {
  case variableName ~ WhileKeywords.`:=` ~ variableValue => WhileAssignmentStatement(variableName, variableValue)
}
```

Finally, for the WHILE programming language, all the combinator functions which I utilized for this parser were the following:

- `|`, the alternation combinator which parses successfully if either its left or its right operand parsers succeeds.
- `~`, the sequential combinator which parses successfully if its left and its subsequent right operand parsers succeeds.
- `chainl1`, this combinator successively applies a parser to the input with a left-associative function that combines the elements it separates.
- `rep1sep`, this combinator successively applies a parser to the input as it is separated by a parser that parses the elements the previous parser with one element of separation required.
- `^^`, the transformation combinator which parses successfully if its left operand parser succeeds to which it transforms by applying a function.

The full parser can be found at the following link: [GitHub Link](https://github.com/jparkie/WhileScalaDsl/blob/master/src/main/scala/com/github/jparkie/bsd/parser/WhileParser.scala).

### Creating the Interpreter

With the AST established and the parser created, the creating of an interpreter was fairly simple after the bootstrapping done by the `WhileAbstractSyntaxTreeNode`. To create the interpreter, I created a `WhileInterpreter` object which performed a traversal of the AST by successively calling each node’s `evaluate()` method with a `WhileEnvironment` case class as an implicit to manage a primitive `mutable.Map` to manage state.

```scala
case class WhileEnvironment(defaultValueMap: Map[String, Any] = Map.empty)
  extends mutable.HashMap[String, Any] {
  defaultValueMap foreach { currentKeyValuePair =>
    this.put(currentKeyValuePair._1, currentKeyValuePair._2)
  }
}
```

```scala
object WhileInterpreter {
  def run(whileProgram: WhileProgram): Unit = {
    val executionResult = whileProgram.evaluate()(WhileEnvironment())

    executionResult match {
      case Success(_) =>
        Console.out.println("Program exited upon completion.")
      case Failure(exception) =>
        Console.out.println(s"Program exited upon failure: ${exception.getMessage}.")
    }
  }
}
```

The full interpreter can be found at the following link: [GitHub Link](https://github.com/jparkie/WhileScalaDsl/tree/master/src/main/scala/com/github/jparkie/bsd/interpreter).

### Creating the Compiler

Again with the AST established and the parser created, the creating of the compiler was not too difficult with ease of reflection provided by Scala 2.11’s quasiquotes and `Toolbox`. Utilizing these two concepts, I was able to create a `WhileCompiler` object which flattened an AST to a Scala program, then parsed the Scala program to a Scala AST, then compile the Scala AST as Java bytecode, and then dynamically load it.

Quasiquotes in Scala are a great feature of its reflection and macro libraries to manipulate Scala ASTs with features such as interpolation and pattern matching. It is a topic sadly too large for the scope of this article, so I highly suggest reading more about it at this link: [http://docs.scala-lang.org/overviews/quasiquotes/intro.html](http://docs.scala-lang.org/overviews/quasiquotes/intro.html). Accordingly, quasiquotes was utilized in the compiler to take a flatten AST, a Scala program, and lift the `String` with quasiquotes exposed by a `Toolbox` into a Scala AST.

Meanwhile, a `Toolbox` is a feature of Scala Reflection which allows type-checking, parsing, compiling, and evaluating of Scala ASTs. Again this is a feature sadly too large for the scope of this article. I do know that Databricks loves Scala Reflection regarding quasiquotes and `Toolbox`es as they utilized it heavily to optimize Spark SQL as explained here if you want to learn more about `Toolbox`: [https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html).

```scala
object WhileCompiler extends WhileCompilerQuasiquotes {
  val whileTemplateType = typeOf[WhileTemplate]

  def getMirror: Mirror =
    runtimeMirror(Thread.currentThread().getContextClassLoader)

  val generateTemplateToolbox = getMirror.mkToolBox()

  def compileWhileTemplate(whileProgram: WhileProgram): Try[WhileTemplate] = {
    def generateClassString(whileScalaString: String): String = {
      val temporaryString = s"new $whileTemplateType { override def run(): Unit = { $whileScalaString } }"

      temporaryString
    }

    def generateClassAST(whileClassString: String): Tree = {
      val temporaryTree = generateTemplateToolbox.parse(whileClassString)

      temporaryTree
    }

    def generateWhileTemplate(whileClassTree: Tree): WhileTemplate = {
      val temporaryMethod = generateTemplateToolbox.compile(whileClassTree)

      temporaryMethod().asInstanceOf[WhileTemplate]
    }

    try {
      whileProgram.asScala
        .map(generateClassString(_))
        .map(generateClassAST(_))
        .map(generateWhileTemplate(_))
    } catch {
      case exception: Exception => Failure(exception)
    }
  }

  def run(whileTemplate: WhileTemplate): Unit = {
    val executionResult = Try(whileTemplate.run())

    executionResult match {
      case Success(_) =>
        Console.out.println("Program exited upon completion.")
      case Failure(exception) =>
        Console.out.println(s"Program exited upon failure: ${exception.getMessage}.")
    }
  }
}
```

## WHILE in Practice

With the AST, the parser, the interpreter, and the compiler all implemented in Scala, a WHILE program can finally be consumed. For the following WHILE program as an output, the outputs of the parser, the interpreter, and the compiler will be listed.

```
BEGIN
variable1 := 1.0;
variable2 := 2.0;
IF variable2 > variable1 THEN
  WRITE true
ELSE
  WRITE false
END
```

### The WHILE Parser

```scala
WhileProgram(
  WhileMultipleStatement(List(
    WhileAssignmentStatement(variable1,WhileNumberLiteral(1.0)),       
    WhileAssignmentStatement(variable2,WhileNumberLiteral(2.0)), 
    WhileIfThenElseStatement(
      WhileGreaterThanOperator(WhileNumberVariable(variable2),WhileNumberVariable(variable1)),
      WhileMultipleStatement(List(WhileWriteBooleanStatement(WhileBooleanLiteral(true)))),
      WhileMultipleStatement(List(WhileWriteBooleanStatement(WhileBooleanLiteral(false))))
    )
  ))
)
```

### The WHILE Interpreter

```
//
// WhileInterpreter
//
// Name: Jacob Park
// Date: Friday, August 28, 2015
//
       
true
Program exited upon completion.
```

### The WHILE Compiler

```
//
// WhileCompiler
//
// Name: Jacob Park
// Date: Friday, August 28, 2015
//
       
true
Program exited upon completion.
```

## Conclusion

With all of this, I implemented the WHILE programming language as an external DSL in Scala. I had a lot of fun with this small project. If you want to learn more about Scala Parser Combinators and Scala Reflection through a book, [*Programming Scala, 2nd Edition*](http://shop.oreilly.com/product/0636920033073.do) has the following great sections:

- Chapter 19: Dynamic Invocation in Scala,
- Chapter 20: Domain-Specific Languages in Scala,
- Chapter 24: Metaprogramming: Macros and Reflection.


## Notes

- The implementation of my dialect of the WHILE programming language in Scala is available at the following link: [https://github.com/jparkie/WhileScalaDsl](https://github.com/jparkie/WhileScalaDsl).
