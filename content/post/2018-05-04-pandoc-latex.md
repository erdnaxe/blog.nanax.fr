---
categories:
- Software
date: "2018-05-04"
title: "Write reports using Pandoc and LaTeX backend"
description: "Learn how to write a Makefile in order to generate professional looking documents from human-friendly Markdown sources."
keywords: ["xelatex", "pandoc", "xelatex", "latex", "reports", "tex", "pagella"]
aliases: ["/pandoc-latex/"]
---

LaTeX is a great markup language to write documents such as scientific articles or lessons,
but writing directly in LaTeX sometimes results in very complex documents with many packages and macros.
100% of the writing time isn't focused on content as you have to format with the markup language.

Pandoc is a powerful multi-format document converter, and it is able to convert **Markdown**[^1] to LaTeX.
So Pandoc is capable of writing the LaTeX corresponding to what you wrote as Markdown saving your time.
And if you need a complex LaTeX command that Pandoc doesn't support you can directly put LaTeX in the Markdown.

I am going to guide you through the creation of a **Makefile**[^2] to compile **Markdown documents to PDF** through **XeTeX**[^3].

## Do I really need a Makefile?

Coming soon...

## Basic Makefile

The idea is to create a directory in which you will be able to call `make` command to compile all Markdown documents inside.

The following is the content of a minimal `Makefile` file that builds `document1.md` and `document2.md`
to `build/document1.pdf` and `build/document2.pdf` using **XeTeX**.

```make
SOURCES = \
	document1 \
	document2
BUILDDIR = build/
DEP = $(wildcard *.sty *.tex *.jpg *.png)
TARGETS = $(addprefix $(BUILDDIR),$(addsuffix .pdf,$(SOURCES)))

# Change LaTeX engine
PARAMETERS = --pdf-engine=xelatex

all: $(TARGETS)

$(BUILDDIR)%.pdf : %.md $(DEP)
	@mkdir -p $(BUILDDIR) # Make sure build dir exists
	pandoc $(PARAMETERS) $< -o $@

clean:
	@rm -f $(TARGETS)
```

`DEP` contains all files that may be included in documents.
Changes to those files (here those who end in *.sty*, *.tex*, *.jpg* or *.png*) will imply a new compilation next time `make` is called.


## Add some custom LaTeX header commands

To modify Pandoc LaTeX output, you can modify Pandoc LaTeX template, or more simply just append commands in the header.
I personally prefer the second method which seems cleaner and doesn't touch Pandoc internal files.

You can append code in the header directly in a parameter or pass a file.

### Include a file in LaTeX header

The following shows an example creating a page footer using *fancyhdr* package.

First of all, create a file `header.tex` containing the custom LaTeX header:

```latex
\usepackage{fancyhdr}
\usepackage{lastpage}
\pagestyle{fancy}
\renewcommand{\headrulewidth}{0pt}
\renewcommand{\footrulewidth}{1pt}
\fancyhead{}
\fancyfoot[L]{My name!}
\fancyfoot[R]{
  \href{https://github.com/}{Go to GitHub}
}
\fancyfoot[C]{Page \thepage/\pageref{LastPage}}
```

Now pass the file to Pandoc by appending a parameter in the `Makefile`:

```make
#[...]

# Change LaTeX engine
PARAMETERS = --pdf-engine=xelatex

# Custom LaTeX header : Page footer
PARAMETERS += --include-in-header=header.tex

#[...]
```

### Directly add commands to LaTeX header

It's quite the same that previously except everything is done inside the `Makefile`:

```make
#[...]

# Change LaTeX engine
PARAMETERS = --pdf-engine=xelatex

# Custom LaTeX header : Watermark documents
PARAMETERS += -V 'header-includes:\usepackage{draftwatermark}\
	\SetWatermarkScale{0.8}\
	\SetWatermarkText{Perso}'

#[...]
```

## Change document font family

The following describes how to change the font to [TeX Gyre Pagella](http://www.gust.org.pl/projects/e-foundry/tex-gyre/pagella).

XeLaTeX uses OpenType fonts so you have to download your font in an OpenType format and place it in a *fonts* directory. You will have a structure like this one:

```
fonts/texgyrepagella-bold.otf
fonts/texgyrepagella-bolditalic.otf
fonts/texgyrepagella-italic.otf
fonts/texgyrepagella-math.otf
fonts/texgyrepagella-regular.otf
```

Then in the `Makefile` add four parameters to Pandoc:

```make
#[...]

# Change LaTeX engine
PARAMETERS = --pdf-engine=xelatex

# Set fonts
PARAMETERS += -V mainfont:"texgyrepagella.otf" \
	-V 'mainfontoptions: Path = fonts/, \
		UprightFont = *-regular, \
		BoldFont = *-bold, \
		ItalicFont = *-italic, \
		BoldItalicFont = *-bolditalic' \
	-V mathfont:"texgyrepagella-math.otf" \
	-V 'mathfontoptions: Path = fonts/'

#[...]
```

## Resulting Makefile

*TL;DR.* To summarise, the following `Makefile` groups all previous modifications. That is actually my daily driver: 

```make
SOURCES = \
	document1 \
	document2
BUILDDIR = build/
DEP = $(wildcard *.sty *.tex *.jpg *.png)
TARGETS = $(addprefix $(BUILDDIR),$(addsuffix .pdf,$(SOURCES)))

# Change LaTeX engine and use custom header
PARAMETERS = --pdf-engine=xelatex

# Custom LaTeX header : Page footer
PARAMETERS += --include-in-header=header.tex

# Custom LaTeX header : Watermark documents
PARAMETERS += -V 'header-includes:\usepackage{draftwatermark}\
	\SetWatermarkScale{0.8}\
	\SetWatermarkText{Perso}'

# Set fonts
PARAMETERS += -V mainfont:"texgyrepagella.otf" \
	-V 'mainfontoptions: Path = fonts/, \
		UprightFont = *-regular, \
		BoldFont = *-bold, \
		ItalicFont = *-italic, \
		BoldItalicFont = *-bolditalic' \
	-V mathfont:"texgyrepagella-math.otf" \
	-V 'mathfontoptions: Path = fonts/'

all: $(TARGETS)

$(BUILDDIR)%.pdf : %.md $(DEP)
	@mkdir -p $(BUILDDIR) # Make sure build dir exists
	pandoc $(PARAMETERS) $< -o $@

clean:
	@rm -f $(TARGETS)
```

## Bonus: convert your old documents to Markdown

Pandoc is able to convert many types of document such as Word (*.docx*) and LaTeX (*.tex*) to Markdown (*.md*).

Just try something like:

```bash
pandoc mydocument.docx -o mydocument.md
```

---

[^1]: According to Wikipedia: "**Markdown** is a lightweight markup language with plain text formatting syntax." All this website content was written in Markdown!
[^2]: According to Wikipedia: "A **Makefile** is a file containing a set of directives used with by make build automation tool to generate a target/goal."
[^3]: **XeTeX** is a TeX/LaTeX engine using Unicode and supporting OpenType font.
