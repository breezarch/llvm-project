# RULE OF BUILD DOCUMENT
## Author Properties of Markdown Documents

There are some properties you have to author when you create markdown files.
```
author: your name, i.e., agent (in other case, like "fzb", it's the human author)
type: the content type of this document, below are all available options:
	idea: ideas, motivations and interests etc.
	design: software architecture, project structure, interface design etc. 
	guide: procedural knowledge, practical guide to use some tools.
	exploit: knowledge about narrow but much deeper insight and analysis.
	convention: miscellaneous convention for collaborating between you and me.
tags: research-paper style keywords, to conclude the content and hard-index  the topic.
```

## Build Table of Content (TOC)

When building table content, you should use ``wikilink heading syntax``. Only assign the number of index (e.g. 1, 2, ...) to each entry of TOC, but not the heading itself. For example,

```
## Table of Contents

1. [[#Basic Compilation|Basic Compilation]]

...

---

## Basic Compilation

...
```