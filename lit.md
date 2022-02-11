# The Bootstrap

	sed -n '0,/^# Utils/d;s/^\t//p' "$0"|sh -s "$0" $*;exit

This line will execute every following tabulated line in this
file as a shell script.
It seems linux will use standard shell if shebang is missing,
but you can pass this file to `sh` manually as well.

# Utils

Bootstrap is not terribly convinient by itself, we need more features.
First, put name of this file into a separate variable.

	file="$1"; shift

Then, for debugging purposes, add option for straight dump to stdout
of executable lines.  This is achived by testing first argument `$1`
against `-d` string, printing all tabulated lines and quitting.

	[[ "$1" = "-d" ]] && {
		sed -n '0,/^# Actual Code/d;s/^\t//p' "$file"
		exit
	}

Exit is needed to stop further execution.

# Actual Code

Bootstrap itself doesn't impose any formatting rules beyond tabulation for code blocks,
but I am using [Markdown](https://www.markdownguide.org), because that's what I am
familiar with, because it also uses tabs for code blocks and because it uses `#` for
headers, allowing me to conviniently place header at the top of file, which will be
treated as a comment by shell during execution.

Markdown is not cool, though, therefore this file shall be converted to LaTeX.

## Conversion

Parsing markdown for real is surprisingly non-trivial,
but if we focus only on small subset of what we use,
it might be doable with just `awk`.

	awk '
	BEGIN {
		print "\\documentclass[a4paper,12pt]{article}"
		print "\\begin{document}"
		iscodeblock = 0
	}

Inline code needs `verb` command.

	!/^\t/{
		x = 1
		while ( x != 0 ) {
			x = sub("`", "\\verb|", $0)
			x = sub("`", "|", $0)
		}
	}

Section headers are marked by `section` command.

	/^# / {
		sub("# ", "")
		printf "\\section{%s}\n", $0
		next
	}
	/^## / {
		sub("## ", "")
		printf "\\subsection{%s}\n", $0
		next
	}

Code-blocks need to be wrapped in `verbatim` environment.
Closing statement is broken apart in source, because escaping it in
code is rather complicated.

	/^\t/ {
		if (iscodeblock == 0) print "\\begin{verbatim}"
		iscodeblock = 1
	}
	!/^\t/ {
		if (iscodeblock == 1) printf "\\%s\n", "end{verbatim}"
		iscodeblock = 0
	}

URLs are converted into footnotes.

	!/^t/&&/\[.+\]\(.+\)/{
		rg = "\\[.+\\]\\(.+\\)"
		x = match($0, rg)
		s = substr($0, RSTART+1, RLENGTH-2)
		split(s, a, "\\]\\(")
		# S = a[1]+"\\footnote{%"+a[2]+"}"
		S = sprintf("%s\\footnote{%s}", a[1], a[2])
		sub(rg, S, $0)
	}

First tab is stripped away, and the rest is converted to spaces (yuck!)

	{
		sub("^\t", "", $0)
		gsub("\t", "        ", $0)
		print $0
	}
	END {
		print "\\end{document}"
	}
	' "$file" > "$file".latex

## Getting PDF

First, make sure `LaTeX` is installed.

	command -v pdflatex || {
		echo LaTeX not installed
		exit
	}

And if it is, then things are simple.

	pdflatex $file.latex

Resulting pdf should be available as `./lit.md.pdf` if all went well.
