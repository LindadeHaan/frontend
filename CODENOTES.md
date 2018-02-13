## Woorden die ik niet begrijp:
1. Lexical

## Aantekeningen les
Scope is waar variable worden opgeslagen. Waar je ze kan vinden. "Waar leeft een variable". Een set afspraken.  
Stel je hebt een functie (a) in die functie maak je een variable (b) aan. Als je een variable in een functie declareert, leeft die variable in de scope van de functie (Function Scope/Lexical Scope).  
Als je een variable in een blok defineert dan zit deze in een global scope (window).  
Zodra er geen referentie gevonden kan worden naar een variable, krijg je een ReferenceError. Dit krijg je als je, je var in een functie zet, en deze buiten deze functie aanroept.  
Engine doet het werk. Zorgt ervoor dat het programma wordt ingelezen en worden doorgegeven aan de content.  
__LHS__ = waar is de declaratie van de variabele
__RHS__ = wat is de waarde van de variabele

# Aantekening van Chapter 1: What is scope?  
But the inclusion of variables into our program begets the most interesting questions we will now address: where do those variables live? In other words, where are they stored? And,
most importantly, how does our program find them when it needs them?

These questions speak to the need for a well-defined set of rules for storing variables in some location, and for finding those variables at a later time.
We'll call that set of rules: Scope.

## Compiler theory

It may be self-evident, or it may be surprising, depending on your level of interaction with various languages, but despite the fact that JavaScript falls under the general category 
of "dynamic" or "interpreted" languages, it is in fact a compiled language. It is not compiled well in advance, as are many traditionally-compiled languages, nor are the results of 
compilation portable among various distributed systems.

In a traditional compiled-language process, a chunck of source code, your program, will undergo typiclaly three steps *before* it is executed, roughly called "compilation":  
1. __Tokenizing/Lexing__:breaking up a string of characters into meaningful (to the language) chunks, called tokens. For instance, consider the program: var a = 2;. This program would 
likely be broken up into the following tokens: var, a, =, 2, and ;. Whitespace may or may not be persisted as a token, depending on whether it's meaningful or not.



## Review(TL;DR)
Scope is a set of rules that determines where and how a variable (identifier) can be looked-up. This look-up may be for the purpose of assigning to the variable,
which een LHS (left-hand-side) reference, or it may be for the purposes of retrieving its value, which is aan RHS (right-hand-side) reference.

RHS(right-hand-side) & LHS(left-hand-side)?

LHS references result from assignment operations. *Scope*-related assignments can occur either with the = operator or by passing arguments to (assign to) function parameters.  
The JavaScript *Engine* first compiles code before it executes, and is so doing, it splits up statements like var a=2; into twoo serperate steps:  
1. First, var a to declare it in that Scope. This is performed at the beginning, before code execution.

2. Later, a = 2 to look up the variable (LHS reference) and assign to it if found.

Both LHS and RHS reference look-ups start at the currently executing Scope, and if need be (that is, they don't find what they're looking for there), they work their way up the nested Scope, 
one scope (floor) at a time, looking for the identifier, until they get to the global (top floor) and stop, and either find it, or don't.

Unfulfilled RHS references result in ReferenceErrors being thrown. Unfulfilled LHS references result in an automatic, implicitly-created global of that name (if not in "Strict Mode" 
[^note-strictmode]), or a ReferenceError (if in "Strict Mode" [^note-strictmode]).
