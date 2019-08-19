---
date: 2019-08-18
status: public
tags: Algorithm
title: How to Build a Calculator
---

# 1 Overview
**Calculator** is a common tool for us to get the arithmetic expression's result. In *NIX OS, once you type `bc` command, then you get into `bc` environment. Feel free to input any legal arithmetic expressions. 
```
bc 1.06
Copyright 1991-1994, 1997, 1998, 2000 Free Software Foundation, Inc.
This is free software with ABSOLUTELY NO WARRANTY.
For details type `warranty'. 
```
Let's dive into what's going on when you type string(aka arithmetic expression) then it feeds back the result.  If you're familar with `Pyhton`, it provides a interactive console which you can type any Python's statements, including the above arithmetic expression. 
```
Python 2.7.15 (v2.7.15:ca079a3ea3, Apr 29 2018, 20:59:26) 
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> 1 * 2 + 3 - 4 / 2
3
>>> 
```
Now that it's legal Python code, it must be underwent the full compile phases.  Here are they:
- Tokenize
- Parse
- Evalute

Exactly right, they're all stages for calculate the arithmetic expression. In this article we'll complete above from scratch in `C#` language (also netstandard2.0). And you will learn from this article.
- Abstract syntax tree (aka: AST)
- Pratt parsing method.
- How to evaluate system.

# 2 Environment

- Editor: [Visual Studio Code](https://code.visualstudio.com/)
- Language Runtime: [Dotnet Core](https://dotnet.microsoft.com/download)

After installing `Visual Studio Code`, you could install `C#` plugin to help you write a better code. 
Now you can run command `dotnet new console CalculatorCLI` to create a console application. And open the folder with `visual studio code`. At the begining the project, we just complete the prototype program that we want. 

```csharp:n
// program.cs
using System;
namespace CalculatorCLI
{
    class Program
    {
        static void Main(string[] args)
        {
            const string WELCOME = "Welcome to Calculator. Feel free to type any expression you want.";
            const string PROMPT = ">> ";
            Console.Out.Write(WELCOME + Environment.NewLine);
            while(true)
            {
                Console.Out.Write(PROMPT);
                try
                {
                    var input = Console.In.ReadLine();
                    if(string.Compare(input, "exit", StringComparison.OrdinalIgnoreCase) == 0)
                    {
                        break;
                    }
                    Console.Out.Write(input);
                    Console.Out.Write(Environment.NewLine);
                }
                catch(Exception e)
                {
                    Console.Out.Write("Oops! Something seems to go wrong." + Environment.NewLine);
                    Console.Out.Write(e.Message);
                    Console.Out.Write(Environment.NewLine);
                }
   
            }
        }
    }
}
```
command: `dotnet run -p Calculator/CalculatorCLI`
![](./_image/2019-08-18-14-02-16.jpg)There are nothing mysterious in above pieces of code. Read code from input then write back and what's our job? The lines between #16 and #21 will be filled by our calculator. Here we go.

# 3 Calculator Implementation
Take simplicity into consideration, our calculator only supports integer number, excluding float or complex number. And supports add(`+`), minus(`-`), multiply(`*`), divison(`/`) and power (`^`) operators. Parenthesises to group expression is also supported.

## 3.1 Tokenize
`Token` is the basic concept of complie technology. We re-create the source codes into a sequence of tokens. The `Lexer` will take the responsibility for it. First of all, how many kinds of token are in our calculator?
```csharp:n
//Token.cs
namespace CalculatorCLI.Token
{
    /// <summary>
    /// TokenType
    /// </summary>
    public enum TokenType
    {
        /// <summary>
        /// Integer
        /// </summary>
        INT,

        /// <summary>
        /// Plus +
        /// </summary>
        PLUS,

        /// <summary>
        /// Minus -
        /// </summary>
        MINUS,

        /// <summary>
        /// Multiply *
        /// </summary>
        MULTIPLY,

        /// <summary>
        /// Divide /
        /// </summary>
        DIVIDE,

        /// <summary>
        /// Power ^
        /// </summary>
        POWER,

        /// <summary>
        /// left parenthesis "("
        /// </summary>
        LPAREN,

        /// <summary>
        /// right parenthesis ")"
        /// </summary>
        RPAREN,

        /// <summary>
        /// Eof (end of file)
        /// </summary>
        EOF,

        /// <summary>
        /// Illegal token
        /// </summary>
        ILLEGAL,
    }
}
```
Nothing strange, except `EOF` and `ILLEGAL`, they are just talked above section. 
```csharp:n
//Token.cs
namespace CalculatorCLI.Token
{
    //[...]
    /// <summary>
    /// Token 
    /// </summary>
    public class Token
    {
        /// <summary>
        /// TokenType.
        /// </summary>
        public TokenType Type { get; set; }

        /// <summary>
        /// Token's literal value.
        /// </summary>
        public string Literal { get; set; }
    }
}
```
`Token` class is quite simple, including two properties. `Literal` including the token literal value in the source code.
The next step is to build our lexer, which just only includes one feature: `NextToken`. Every time we call `NextToken`, it returns a `Token` instance to represent the current source code.  
Before we move forward, we should expect what we'll get. Here are unit tests for the `Lexer` implementation, just like TDD(`Test Drive Development`). Use `dotnet new nunit CalculatorTest` command to create `NUnit` test framwork then `dotnet add CalculatorTest.csproj reference CalculatorCLI.csproj`。
```csharp:n
// LexerTest.cs
namespace CalculatorTest.Lexer
{
    using NUnit.Framework;

    using CalculatorCLI.Lexer;

    using CalculatorCLI.Token;

    [TestFixture]
    public class LexerTest
    {
       [Test]
       public void TestLexerNextToken()
       {
            string input = @"1 + 2 * (12 - 6) / 3 + 2 ^ 3
+ ((4+3) *   4)
/ 4
";

            var tests = new[]
            {
                new Token {Type = TokenType.INT, Literal="1"},
                new Token {Type = TokenType.PLUS, Literal="+"},
                new Token {Type = TokenType.INT, Literal="2"},
                new Token {Type = TokenType.MULTIPLY, Literal="*"},
                new Token {Type = TokenType.LPAREN, Literal="("},
                new Token {Type = TokenType.INT, Literal="12"},
                new Token {Type = TokenType.MINUS, Literal="-"},
                new Token {Type = TokenType.INT, Literal="6"},
                new Token {Type = TokenType.RPAREN, Literal=")"},
                new Token {Type = TokenType.DIVIDE, Literal="/"},
                new Token {Type = TokenType.INT, Literal="3"},
                new Token {Type = TokenType.PLUS, Literal="+"},
                new Token {Type = TokenType.INT, Literal="2"},
                new Token {Type = TokenType.POWER, Literal="^"},
                new Token {Type = TokenType.INT, Literal="3"},
                new Token {Type = TokenType.PLUS, Literal="+"},
                new Token {Type = TokenType.LPAREN, Literal="("},
                new Token {Type = TokenType.LPAREN, Literal="("},
                new Token {Type = TokenType.INT, Literal="4"},
                new Token {Type = TokenType.PLUS, Literal="+"},
                new Token {Type = TokenType.INT, Literal="3"},
                new Token {Type = TokenType.RPAREN, Literal=")"},
                new Token {Type = TokenType.MULTIPLY, Literal="*"},
                new Token {Type = TokenType.INT, Literal="4"},
                new Token {Type = TokenType.RPAREN, Literal=")"},
                new Token {Type = TokenType.DIVIDE, Literal="/"},
                new Token {Type = TokenType.INT, Literal="4"},
                new Token {Type = TokenType.EOF, Literal=""},
            };
            Lexer lexer = new Lexer(input);
            foreach (var tt in tests)
            {
                var actualToken = lexer.NextToken();
                Assert.AreEqual(tt.Type, actualToken.Type, $"expect TokenType {tt.Type}, got {actualToken.Type}");
                Assert.AreEqual(tt.Literal, actualToken.Literal, $"expect Token's Literal {tt.Literal}, got {actualToken.Literal}");
            }
       }
    }
}
```
It chooses `table-driven` unit test pattern. The `Lexer` takes in a string input and invokes `NextToken` once a time to generate a token. Compre this token with expect one. 
```csharp:n
// Lexer.cs
namespace CalculatorCLI.Lexer
{
    using Token;
    public class Lexer
    {
        private readonly string input;
        private int position;

        public Lexer(string input)
        {
            this.input = input;
            this.position = 0;
        }

        public Token NextToken()
        {
            this.SkipWhitespace();
            if (this.position >= this.input.Length)
            {
                return new Token { Type = TokenType.EOF, Literal=""};
            }
            Token token = null;
            char character = this.input[this.position];
            switch (character)
            {
                case '+':
                    token = new Token { Type = TokenType.PLUS, Literal = "+" };
                    break;
                case '-':
                    token = new Token { Type = TokenType.MINUS, Literal = "-" };
                    break;
                case '*':
                    token = new Token { Type = TokenType.MULTIPLY, Literal = "*" };
                    break;
                case '/':
                    token = new Token { Type = TokenType.DIVIDE, Literal = "/" };
                    break;
                case '^':
                    token = new Token { Type = TokenType.POWER, Literal = "^" };
                    break;
                case '(':
                    token = new Token { Type = TokenType.LPAREN, Literal = "(" };
                    break;
                case ')':
                    token = new Token { Type = TokenType.RPAREN, Literal = ")"};
                    break;
                default:
                    if(IsDigit(character))
                    {
                        token = new Token { Type = TokenType.INT };
                        token.Literal = this.ReadNumber();
                    }
                    else
                    {
                        token = new Token { Type = TokenType.ILLEGAL, Literal=""};
                    }
                    break;
            }
            this.position++;
            return token;
        }
        private void SkipWhitespace()
        {
            while(this.position < this.input.Length && IsWhitespace(this.input[this.position]))
            {
                this.position++;
            }
        }
        private string ReadNumber()
        {
            int position = this.position;
            while(this.position < this.input.Length && IsDigit(this.input[this.position]))
            {
                this.position++;
            }
            string number = this.input.Substring(position, this.position - position);
            this.position--;
            return number;
        }
        private static bool IsDigit(char character)
        {
            return character >= '0' && character <= '9';
        }
        private static bool IsWhitespace(char character)
        {
            return ' ' == character || '\t' == character || '\n' == character || '\r' == character;
        }
    }
}
```
The `Lexer` has only two fileds: `input` and `position`. The `position` points to the current character in the `input`. According to the pointing character, it returns the corresponding `Token`. It is pretty straight, isn't it?