---
author: fzb
type: idea
---
# Overview

## Table of Content
	
1. [[#References]]
	
2. [[#Definition]]
	
3. [[#Related Subjects]]
	
4. [[#Pillar Problems]]
	
5. [[#Inductive Synthesis]]
	

## References
- wikipedia: [https://en.wikipedia.org/wiki/Program_synthesis](https://en.wikipedia.org/wiki/Program_synthesis)
- MIT tutorials: [https://people.csail.mit.edu/asolar/SynthesisCourse/index.htm](https://people.csail.mit.edu/asolar/SynthesisCourse/index.htm)

## Definition

```
Program Synthesis correspond to a class of techniques that are able to generate a program from a collection of artifacts that establish semantic and syntactic requirements for the generated code.
```

## Related Subjects
	
- **Compilation**: the search techniques in compiler and synthesizer.
	
- **Declarative programming**: express the requirements of their computation in a logical form, and when given an input, the runtime system would derive an output that satisfied the logical constraints through a combination of search and deduction.
	
- **Machine Learning**: seek general function specified by input-output pairs dataset, where the ML algorithm itself become program synthesizer.
	

## Pillar Problems
	
- **Intention**: how user should tell their specification;
	
- **Invention**: discover the desired code piece for requirements (specification);
	
- **Adaption**: how to work with existing software system.
	

## Inductive Synthesis

The literature makes a distinction between **Programming by Example** (**PBE**), and **Programming by Demonstration** (**PBD**). In Programming by example, the goal is to infer a function given only a set of inputs and outputs, whereas in Programming by Demonstration, the user also provides a trace of how the output was computed.

For example, in Programming-by-example, if I want to convey to the system that I want it to synthesize the factorial function I may give it an example:

```
factorial(6)=720
```

As one can see, this is a highly under-specified problem, since there is an enormous number of possible functions that may return 720 given the input 6. By contrast, with programming-by-demonstration, one may provide a more detailed trace of the computation:

```
factorial(6)=6∗(5∗(4∗(3∗(2∗1))))=720
```

## Program By Demonstration

Cases:
	
- **Pygmalion** was framed as an "interactive 'remembering' editor for iconic data structures"; the high-level idea was that a program would move icons around and establish relationships between them, but the user would not write the program itself; instead, the user would manipulate the icons directly and the editor would remember the manipulations and be able to apply them in other contexts.

```
a program is a series of EDITING CHANGES to a DISPLAY DOCUMENT. Input to a program is an initial display document, i.e. a display screen containing images. Programming consists of editing the document. The result of a computation is a modified document containing the desired information.
```

## FP Language Style Notation

- **No side-effect**;
- **Concise and expressive**: Think about a program in Java that reverses a list. It's a relatively long program that involves a class declaration, some method declarations, some loops, some constructors; maybe it would look something like this:
	
>[!reverse-list-in-java] reverse-list-in-java
>```java
>class ListReverser{
>  static List reverseList(List myList) {
 >       List output = new ArrayList();
 >       for (int i = 0; i < myList.size(); i++) {
 >           output.add(myList.get(myList.size() - 1 - i));
 >       }
 >       return output;
 >   }
>}
>```
	
>[!reverse-list-in-haskell] reverse-list-in-haskell
>```haskell
>reverse lst = case lst of
>[] -> []
>head:rest -> (reverse rest) ++ [head]
>```
