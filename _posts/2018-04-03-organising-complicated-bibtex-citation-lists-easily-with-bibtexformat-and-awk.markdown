---
layout: post
title: "Cleaning Up Your BibTex Files with Bibtexformat and Perl"
date: 2018-04-03 11:06:28 +1100
featured-img: "person-writing"
categories: Programming
tags:
- vim
- bibtex
- biblatex
- latex
- perl
- citations
- programming
- unix
- bibtexformat
---

Like many poor postgraduate students, I'm currently in the process of putting
together the giant monster that is my PhD dissertation. After learning my
lesson during my honours year, I've chosen to do away with Microsoft Word (Oh!,
the humanity) and instead opt for the more sensible option of LaTeX, written in
[Vim](https://www.vim.org/) and compiled using a full installation of
[TexLive](https://www.tug.org/texlive/). My setup is:

- Use Vim (cross-platform) for writing the raw document
- Use Git and BitBucket for version control
- Use TexLive/latexmk for compilation into a convenient format (e.g. PDF)
- Manage bibliography items using BibTeX

The last point caused some issues. While I love the ease-of-use and the
philosophy of using a plain-text format for storing bibliographies, getting
these libraries into a format that is suitable for theses and publications is
not trivial. For instance, many citations contain special glyphs (e.g.
diacritical marks such as á) that may or may not render appropriately depending
on the character and how the document has been set up. As things stand, I'm at
about 55,000 words and have more than 400 citations, and things are only going
to get more complicated. Yikes! While I could go through each citation with a
fine tooth comb (I probably still will!), it'd be nice to have most of the
bigger problems addressed programmatically to save me the headache. This post
covers my solution to the problem.

## Hold On There!

Before moving on, I'm going to make several assumptions about you, the reader. Namely, that you are:

1. Using a Unix-based operating system (e.g. Linux or OSX)
1. Competent using the command line and some basic tools (`ls`, `git`, `Vim`, etc)

You can probably get this solution going under Windows, but I don't have the
time or patience to bother. Experiment!

Many other resources cover these topics. Google them and visit later!

## The Rules

I had several requirements for a solution aimed at organising and correcting my
BibTeX library:

- Free (as in beer)
- Cross-platform (namely Linux, OSX, and Windows)
- Needs to have a CLI (command-line interface)
- Must have the following features:
  - Detects incorrectly formatted fields (e.g. author names where the prefix Jr
  is badly positioned relative to the surname)
  - Corrects wonky field indentation Allows switching between abbreviated and
  full journal names (e.g. Journal of Biological Chemistry vs. J Biol Chem vs
  J. Biol. Chem.) as I see fit
  - Sorting based upon either keys (labels) or by first author surname
  - Can be run directly from within Vim (avoids having to switch in and out)

Several tools have a reasonably good go at doing this, including
[bibclean](https://ctan.org/pkg/bibclean?lang=en),
[JabRef](http://www.jabref.org/),
[bibcleaner](https://github.com/sirrice/bibcleaner), and
[biber](http://biblatex-biber.sourceforge.net/). While each have their
strengths, no one solution catered to all of my needs. Enter
[bibtexformat](https://github.com/yfpeng/pengyifan-bibtexformat), a neat
cross-platform perl tool that allows for easy formatting of even
complicated bibtex libraries.

## bibtexformat

Download BibTexFormat from
[here](https://github.com/yfpeng/pengyifan-bibtexformat) using git. Many of the
requirements that I set out above are already covered by bibtexformat, which
make life a lot easier. Before we move on, be sure to download bibtexformat and
make sure that `bibtexformat` binary is in your `PATH`. For example, you could
append `PATH` with the location of the cloned repository in your `.bashrc` (if
using bash):

```bash
git clone https://github.com/yfpeng/pengyifan-bibtexformat

export PATH=/path/to/cloned/bibtexformat/repo:$PATH
```

Now we can apply `bibtexformat` to our BibTeX library of interest.

```bash
# take bibtex library as an argument
# -format -wrap 80 = wrap author lists and other fields to under 80 characters
# for readability
# -abb2 = use abbreviated version of journal names
# -sort = sort based upon entry key (e.g. Gordon:18 following Gordon:17)
bibtexformat /path/to/main.bib -format -wrap 80 -abb2 -sort -o -

# N.B this will print to stdout. Change the -o argument to a filename to output
# there
```

Run bibtexformat with the help flag (`bibtexformat -h`) for more options. Play
and see what works.

## Integrating With Vim

I can run this easily from the command line, but how can I integrate it into
Vim? Easy. I can get around this by mapping a new command in Vimscript. Getting
this set up is easy. Simply open up your vim startup file (typically
`$HOME/.vimrc` or `$HOME/.vim/vimrc`) and add the following function:

```vim
" use with call BibTexFormat
function BibTexFormat()
  " Don't capture sterr
  set shellredir=>
  " awk '/@/ {p=1}; / => Done/{p=0};p' --- remove header and footer garbage
  read! 
        \bibtexformat % -format -wrap 80 -abb2 -sort -o - |
        \awk '/@/ {p=1}; / => Done/{p=0};p'
endfunction
```

Now, when you have a bibtex file open (e.g. `file.bib`) in Vim, you can append
the open buffer with a pretty-fied version of the library using the Vim command
`:call BibTexFormat()`. Hey presto, you've got a consistently-formatted BibTex
library. I first like to purge the existing buffer before running this so that
it prints into a clean slate by using the Vim normal mode key combination
`ggdG`, followed by `:call BibTexFormat()`.

We can do better than this though. Many citations contain characters decorated
with special symbols that may muck up sort or, in the worst case, not render
properly when passed to `latexmk` or similar. I'll use `awk` to parse the
output of the `BibTexFormat` command to catch the most common of these.

```vim
" use with call BibTexFormat
function BibTexFormat()
  " Don't capture sterr
  set shellredir=>
  " awk '/@/ {p=1}; / => Done/{p=0};p' --- remove header and footer garbage
  " \s/á/{\\\x27{a}}/g; - correct acute diacritics
  " \s/à/{\\`{a}}/g; - correct grave diacritics
  " \s/ä/{\\"{a}}/g; - correct umlaut diacritics
  read! 
        \bibtexformat % -format -wrap 80 -abb2 -sort -o - |
        \awk '/@/ {p=1}; / => Done/{p=0};p' |
        \sed '
        \s/–/--/g;
        \s/—/--/g;
        \s/á/{\\\x27{a}}/g;
        \s/é/{\\\x27{e}}/g;
        \s/í/{\\\x27{i}}/g;
        \s/ó/{\\\x27{o}}/g;
        \s/ú/{\\\x27{u}}/g;
        \s/à/{\\`{a}}/g;
        \s/è/{\\`{e}}/g;
        \s/ì/{\\`{i}}/g;
        \s/ò/{\\`{o}}/g;
        \s/ù/{\\`{u}}/g;
        \s/ä/{\\"{a}}/g;
        \s/ë/{\\"{e}}/g;
        \s/ï/{\\"{i}}/g;
        \s/ö/{\\"{o}}/g;
        \s/ü/{\\"{u}}/g;
        \'
endfunction
```

But wait, there's more. Biblatex, which I use for handling bibliographies in my
documents, spits the dummy when I provide string-based month fields (Jan vs.
1). I'll catch these too while I'm at it, giving me the finished product.

```vim
" use with call BibTexFormat
function BibTexFormat()
  " Don't capture sterr
  set shellredir=>
  " awk '/@/ {p=1}; / => Done/{p=0};p' --- remove header and footer garbage
  " \s/á/{\\\x27{a}}/g; - correct acute diacritics
  " \s/à/{\\`{a}}/g; - correct grave diacritics
  " \s/ä/{\\"{a}}/g; - correct umlaut diacritics
  read! 
        \bibtexformat % -format -wrap 80 -abb2 -sort -o - |
        \awk '/@/ {p=1}; / => Done/{p=0};p' |
        \perl -pne '
        \s/–/--/g;
        \s/—/--/g;
        \s/á/{\\\x27{a}}/g;
        \s/é/{\\\x27{e}}/g;
        \s/í/{\\\x27{i}}/g;
        \s/ó/{\\\x27{o}}/g;
        \s/ú/{\\\x27{u}}/g;
        \s/à/{\\`{a}}/g;
        \s/è/{\\`{e}}/g;
        \s/ì/{\\`{i}}/g;
        \s/ò/{\\`{o}}/g;
        \s/ù/{\\`{u}}/g;
        \s/ä/{\\"{a}}/g;
        \s/ë/{\\"{e}}/g;
        \s/ï/{\\"{i}}/g;
        \s/ö/{\\"{o}}/g;
        \s/ü/{\\"{u}}/g;
        \s/{Jan}/{1}/g;
        \s/{Feb}/{2}/g;
        \s/{Mar}/{3}/g;
        \s/{Apr}/{4}/g;
        \s/{May}/{5}/g;
        \s/{Jun}/{6}/g;
        \s/{Jul}/{7}/g;
        \s/{Aug}/{8}/g;
        \s/{Sep}/{9}/g;
        \s/{Oct}/{10}/g;
        \s/{Nov}/{11}/g;
        \s/{Dec}/{12}/g
        \'
endfunction
```

## Extra Credit

Bonus points for adding it instead to a vim filetype plugin for bibtex files,
meaning that this function is only set for bibtex-format files in Vim. Neat!

```bash
# create directory when not already present
mkdir -p $HOME/.vim/ftplugin

# create bibtex filetype plugin file
touch $HOME/.vim/ftplugin/bib.vim

# now open the file in vim and add and doi and bibtexformat commands given above
vim $HOME/.vim/ftplugin/bib.vim
```

## Finishing Up

Enjoy your (mostly) corrected, nicely laid out bibtex libraries. I'll follow up
at some stage with a post at how to pull bibliographic entries from a central
repository from within Vim using the digital object identifier (DOI) alone.
Until then, happy writing.
