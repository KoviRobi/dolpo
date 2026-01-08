# DoLPo — Markdown Literate Programming

[ <<'```sh' ]: \]

<!--

```sh
set -eu
export SCRIPT="$(realpath "$0")"

# Put the extracted files into their own directory
mkdir -p build; cd build

# Extract scissor block at the end and run it
sed -n -E <"$SCRIPT" '/-{34}8<-{34}/,/-{34}8<-{34}/p' |
    awk -f - "$SCRIPT"
exit
```

-->

This is a simple markdown literate programming tool, almost certainly
too simple. There are more featureful alternatives out there, such as
[Entangled](https://entangled.github.io/), [Quarto](https://quarto.org/)
and I'm sure many others.

## Name

The name is of course from markDOwn Literate PrOgramming. Why the those
letters? Well initially it was going to be Marklit but that name is
already used. Downlit is also used. And DoLiPro is a medication. DoLiPo is
good, it sounds like solmization. But searching for it gave me Dolpo, the
wikipedia article of which has the name written in Devanagari "डोल्पो" and
Tibetan scripts "དོལ་པོ". And I like other scripts.

So DoLPo it is. The casing to disambiguate from the region, but using
it as `dolpo` is also acceptable (you should capitalise Dolpo for the
region).

## Implementation

It started out as a thought experiment, and was an interesting puzzle. The
question occurred to me, having an executable markdown file. Since you
can put any text in markdown, I also wanted it to read mostly like the
documentation rather than just piping a shell script through Markdown. And
vice versa, so that the code to be highlighted.

My immediate thought was frontmatter. But that isn't part of CommonMark.

### Silent Entrance

The trickier side to the thought experiment is how do you make a shell
(bash, busybox sh) script which is mostly ignored? The solution is
comments, of course. There are a few ways to have non-executing code in
shell, actual comments `# like so (until the end of the line)`, no-effect
commands such as `: colon until the semicolon;`, which have the limitation
that the text in between needs to not contain parentheses, dollar signs
or backticks (otherwise they can start being interpreted as shell again).

But the easiest for multiple lines is the here document, which will pipe
the text to the stdin of the program until the terminating heredocument,
for example:

```sh
: <<'EOF'
You can put pretty much anything here, except from "EOF" without quotes
on its own line.
EOF
```

Note, the quoting of the here document means we don't have any expansion
inside the here document. And we pass it to the no-effect command to
just throw it away. This is pretty good, it only ends up with one line
before the text begins.

Another trick is, because shell is parsed line-by-line, we can terminate
the shell script early, like so:

```sh
exit
Anything after this is just plain text. So we can do
<<<<<<<<<<<<<<<<<<<<<< and it won't complain
```

Now we have shell comments, we need markdown comments. The most obvious
one is probably the HTML comment `<!--` and `-->` pairs. But a sneaky
trick is using link reference definitions, and then not using them.

```md
[some text]: link-target "optional title"
```

So we can combine the here document with a link reference defintion,
meaning we can hide the start of the here document, and just have the
text (or comment the text using HTML comments). So we are almost there
with the start, the last trick is making the here document terminator
be the start of a code block.

~~~md
[ <<'```sh' ]: ]

<!--

Text goes `here`

```sh
# Shell code (with syntax highlighting) goes here. Though probably not
# in the rendered document as this is a doubly-nested markdown block.
exit
```

-->

And the rest is _markdown_ :)
~~~

### The Awk Ward Bits

We can now implement all of it in the header as shell, but we end up
with a lot of text, wouldn't it be better to put most of it at the end,
to avoid polluting the source code, as well as the rendered document?

Of course we can, we have a handle onto the script file as `$0`, so we can
just filter it. We can filter for the scissor markers easily using `sed`.

```sh
sed -n -E <"$0" '/-{34}8<-{34}/,/-{34}8<-{34}/p'
```

This selects all scissored ranges and outputs it to the standard
output. We can call this initial block the "preamble", and we also use it
to export the environment variable `SCRIPT` to point to the source file,
for later use.

### Bootstrapping Code

We are now ready for the bootstrapping code, which will be enough for
the initial implementation. Note, that all the scissor markered blocks
end up being joined together.

First, let's define the state. A reset function allows us to reset it
at various points, such as at the start and after a literate block also.

```awk
# ----------------------------------8<----------------------------------
function reset() {
    FILE="";
    APPEND=0;
    RUN=0;
    ON=0;
}

BEGIN { reset(); }
# ----------------------------------8<----------------------------------
```

We want to detect when we start needing to care, the format of DoLPo
uses block quoted code blocks, because we cannot attach a label to a
code block, but a block quote keeps it visually together. (I don't know
about accessibility such as screen readers, please leave a comment if
it causes a problem for accessibility.)

The format of the header is either "File `name in code quote`" to start
a file, "File `name` continued" to continue/append, or "Run" to execute
(if allowed).

```awk
# ----------------------------------8<----------------------------------
/^> File `.*`( continued)?$/ && !ON {
    match($0, /`.*`/);
    FILE=substr($0, RSTART + 1, RLENGTH - 2);

    if ($0 ~ /continued$/) {
        APPEND=1;
    }
    next;
}
/^> Run$/ && !ON {
    RUN=1;
    next;
}

# ----------------------------------8<----------------------------------
```

Note that we are using `next` because we only want to match on one line at
a time, e.g. to avoid the code block start line being output into the file

Inside block quotes, we use code blocks to provide syntax
highlighting. The bootstrapping code only supports exactly three
backticks, which is a subset of CommonMark.

For run blocks, we only execute at the end, to support multi-line
constructs.

```awk
# ----------------------------------8<----------------------------------
/^> `{3}/ {
    if (ON && RUN && RUN != 0) {
        code = system(RUN)
        if (code != 0) {
            exit code
        }
    }
    ON=!ON;
    next;
}
# ----------------------------------8<----------------------------------
```

Now all that's left is outputting the file contents, or stashing run
blocks for later evaluation. We are strict on other block quotes, and
reset on anything else, because that's going to be an end of block quote.

```awk
# ----------------------------------8<----------------------------------
/^>/ && FILE && !APPEND && ON { print substr($0, 3) >  FILE; next; }
/^>/ && FILE &&  APPEND && ON { print substr($0, 3) >> FILE; next; }
/^>/ && RUN && ON {
    if (RUN == 1) {
        RUN = substr($0, 3);
    } else {
        RUN = RUN ORS substr($0, 3);
    }
    next;
}

/^> ?/ { next; }
/^>/ { print "Error on line " NR; exit 1; }
{ reset(); }
# ----------------------------------8<----------------------------------
```

Probably unsurprisingly, this file itself is executable, and when run,
it builds DoLPo.

We can make the bootstrapping code be version one of DoLPo. Note, in
run blocks we use the exported `SCRIPT` environment variable from ,
rather than `$0` which is going to be the shell of `system(RUN)`.

> Run
>
> ```sh
> sed -n -E <"$SCRIPT" '/-{34}8<-{34}/,/-{34}8<-{34}/p' > dolpo1
> chmod +x dolpo1
> ln -sf dolpo1 dolpo
> ```
