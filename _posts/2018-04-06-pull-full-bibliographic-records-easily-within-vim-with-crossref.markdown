---
layout: post
title: "Pull Full Bibliographic Records Easily Within Vim With CrossRef"
date: 2018-04-07 14:30:00 +1100
featured-img: computer-facing-code
comments: true
sharing: true
categories: Programming
tags:
- vim
- bibtex
- latex
- perl
- curl
- citations
- programming
- crossref
- unix
- bibtexformat
---

In a previous [post]({% post_url
2018-04-03-organising-complicated-bibtex-citation-lists-easily-with-bibtexformat-and-awk
%}) I covered how you can easily format and organise a bibtex bibliography from
within Vim using `bibtexformat`. Today, I'm going to cover how you can pull full
bibliographic entries from *within Vim* right into your open bibtex file. All
you need is one tiny piece of information: the digital object identifier, or DOI
for short.

Basically, most publications (or at least those published within the
last 20 or so years) are each assigned a unique numeric identifier that refers
to them, and them alone. You can think of them acting like a barcode of sorts.
An example of one might be **10.1371/journal.pcbi.1004811**
(**#ShamelessSelfPromotion**).

Thanks to the [CrossRef](https://www.crossref.org), we can conveniently obtain
bibliographic records from the command line with only knowledge of the DOI (see
[link](https://www.crossref.org/labs/citation-formatting-service/) for more
info). The basic syntax is:

```sh
$ curl -sLH "Accept: application/x-bibtex"\
  http://dx.doi.org/<my-doi-here>
```

Give it a try using the doi **10.1371/journal.pcbi.1004811**. You should get the following output:

```sh
$ curl -sLH "Accept: application/x-bibtex"\
  http://dx.doi.org/10.1371/journal.pcbi.1004811

@article{Gordon_2016,
	doi = {10.1371/journal.pcbi.1004811},
	url = {https://doi.org/10.1371%2Fjournal.pcbi.1004811},
	year = 2016,
	month = {mar},
	publisher = {Public Library of Science ({PLoS})},
	volume = {12},
	number = {3},
	pages = {e1004811},
	author = {Shane E. Gordon and Daniel K. Weber and Matthew T. Downton and John Wagner and Matthew A. Perugini},
	editor = {Bert L. de Groot},
	title = {Dynamic Modelling Reveals `Hotspots' on the Pathway to Enzyme-Substrate Complex Formation},
	journal = {PLOS Computational Biology}
}
```

Note that we are requesting that the bibliographic information be formatted as
`application/x-bibtex`, which translates to bibtex-styled output. You can also
trivially change this to something else of you like. For instance, you can also
curl bibliographic formation in json format by changing this bit to
`application/vnd.citationstyles.csl+json`. Neat. Info on other supported types
can be found [here](https://citation.crosscite.org/docs.html).

Now that we've got the basics, it's pretty trivial to integrate this step into
Vim by mapping it to a custom command. Let's call that Vim normal mode command
`Doi`. Insert the following into either your `$HOME/.vimrc` or your
`$HOME/.vim/ftplugin/bib.tex` (better).

```vim
" Simple command 'Doi' that takes single DOI as argument
command! -nargs=+ Doi r
      \!curl -sLH "Accept: application/x-bibtex"
      \http://dx.doi.org/<q-args>
```

Using the example given above (doi: 10.1371/journal.pcbi.1004811) and the Vim
normal mode command `:Doi 10.1371/journal.pcbi.1004811` gives us the output
that we observed in the shell before:

```bib
@article{Gordon_2016,
	doi = {10.1371/journal.pcbi.1004811},
	url = {https://doi.org/10.1371%2Fjournal.pcbi.1004811},
	year = 2016,
	month = {mar},
	publisher = {Public Library of Science ({PLoS})},
	volume = {12},
	number = {3},
	pages = {e1004811},
	author = {Shane E. Gordon and Daniel K. Weber and Matthew T. Downton and John Wagner and Matthew A. Perugini},
	editor = {Bert L. de Groot},
	title = {Dynamic Modelling Reveals `Hotspots' on the Pathway to Enzyme-Substrate Complex Formation},
	journal = {PLOS Computational Biology}
}
```

Not bad for a first try, but we can do better. We can pass the output of this
command through perl substition to improve the formatting and readability.

```perl
# replace em-dashes with latex equivalents, typically for number ranges
's/–/--/g'

# reformat citation key from typical Name_1990 to Name:90
's/_[1-2]\d(\d{2})/:$1/g'

# correct diacritics
's/á/{\\\x27{a}}/g'
's/à/{\\`{a}}/g'
's/ä/{\\"{a}}/g'

# change month from string to number for indexing
's/({Jan}|{jan})/{1}/g'
```

Finally, we can also alias `doi` to the `Doi` so that Vim tolerates both forms
of the term. Now, whenever you type `doi` followed by a space (or other
punctuation mark), the `doi` will "expand" to read as `Doi`. Try it out.

```vim
cab doi Doi
```

Whew! Putting that all together with the original command gives us the following.

```vim
" s/_[1-2]\d(\d{2})/:$1/g' - formats citation key from Name_1990 to Name:90
" s/({Jan}|{jan})/{1}/g - Replace months string with number
" \s/á/{\\\x27{a}}/g; - correct acute diacritics
" \s/à/{\\`{a}}/g; - correct grave diacritics
" \s/ä/{\\"{a}}/g; - correct umlaut diacritics
command! -nargs=+ Doi r
      \!curl -sLH "Accept: application/x-bibtex"
      \http://dx.doi.org/<q-args> |
      \perl -pne 
      \'
      \s/–/--/g;
      \s/_[1-2]\d(\d{2})/:$1/g;
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
      \s/({Jan}|{jan})/{1}/g;
      \s/({Feb}|{feb})/{2}/g;
      \s/({Mar}|{mar})/{3}/g;
      \s/({Apr}|{apr})/{4}/g;
      \s/({May}|{may})/{5}/g;
      \s/({Jun}|{jun})/{6}/g;
      \s/({Jul}|{jul})/{7}/g;
      \s/({Aug}|{aug})/{8}/g;
      \s/({Sep}|{sep})/{9}/g;
      \s/({Oct}|{oct})/{10}/g;
      \s/({Nov}|{nov})/{11}/g;
      \s/({Dec}|{dec})/{12}/g
      \'
cab doi Doi
```

Stick this into your `$HOME/.vimrc` or `$HOME/.vim/ftplugin/bib.vim` file. Now,
when you open up a bibtex file (e.g. `filename.bib`), Vim will define our new
function. Putting it to use on the function above using the Vim normal mode
command `:Doi 10.1371/journal.pcbi.1004811` now gives us a much nicer output:

```bib
@article{Gordon:16,
  doi = {10.1371/journal.pcbi.1004811},
  url = {https://doi.org/10.1371%2Fjournal.pcbi.1004811},
  year = 2016,
  month = {3},
  publisher = {Public Library of Science ({PLoS})},
  volume = {12},
  number = {3},
  pages = {e1004811},
  author = {Shane E. Gordon and Daniel K. Weber and Matthew T. Downton and John Wagner and Matthew A. Perugini},
  editor = {Bert L. de Groot},
  title = {Dynamic Modelling Reveals `Hotspots' on the Pathway to Enzyme-Substrate Complex Formation},
  journal = {PLOS Computational Biology}
}
```

Looks good, doesn't it? Now, just run it through the `:call BibTexFormat`
command that I defined in an earlier [post]({% post_url
2018-04-03-organising-complicated-bibtex-citation-lists-easily-with-bibtexformat-and-awk
%}) to apply the finishing touches.
