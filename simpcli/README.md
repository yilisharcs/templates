# NAME

{{project-name}}

# SYNOPSIS

TBD

# DESCRIPTION

{{project-description}}

# MASKFILE RULES

This section contains commands for the task runner **mask**. If you're not a
developer, you may skip to the next section. Otherwise, please install the
following dev-dependencies:

* **entr**
* **mask**
* **nushell**
* **pandoc**

NOTE: The scripts below may deviate slighty from the maskfile source due to the
way **pandoc** handles conversions to the \*roff format.

## build

> Builds the program

**OPTIONS**
* release
  * flags: --release
  * desc: Build with release mode

```nu
if $env.release? == "true" {
	cargo build --release
} else {
	cargo build
}
```

## run

> Runs the program

**OPTIONS**
* release
  * flags: --release
  * desc: Build with release mode

```nu
if $env.release? == "true" {
	cargo run --release
} else {
	cargo run
}
```

## man

> Builds the manpage from the maskfile.md source file

```nu
let root = (cargo locate-project | from json | get root | path dirname)
let mandir = ([$root "docs/man"] | path join)

if ($mandir | path exists) {
	ls $mandir
	| each { rm $in.name }
	| ignore
} else {
	mkdir $mandir
}

mut pkg = ""
mut semver = ""

cargo pkgid
| path basename
| if ($in | str contains "@") {
  $pkg = ($in | str replace --regex ".*#(.*)@.*" "$1")
  $semver = ($in | str replace --regex ".*@" "")
} else {
  $pkg = ($in | str replace --regex "#.*" "")
  $semver = ($in | str replace --regex ".*#" "")
}

let pkgname = ($pkg | str title-case)
let pkgbin = ($pkg | str replace --regex " .*" "" | str downcase)
let date = (date now | to text | split row " " | get 1 2 3 | str join " ")
let mansection = 1 # Executables or shell commands

open maskfile.md
| lines
| insert 0 $"% ($pkgbin)\(($mansection)\) ($pkgbin) ($semver) | ($pkgname) Manual"
| insert 1 "%"
| insert 2 $"% ($date)"
| insert 3 ""
| each {
	# NOTE: Tab char (^I)
	$in | str replace --all "	" "  "
}
| each {
	if ($in | str starts-with "# ") {
		prepend (char hamburger)
	} else {
		return $in
	}
}
| flatten
| to text
| split row (char hamburger)
| do {
	let mask_index = (
		$in
		| enumerate
		| where $it.item =~ "# MASKFILE RULES"
		| first
		| get index
	)

	$in
	| update $mask_index {
		lines
		| enumerate
		| do {
			let subheader = (
				$in
				| skip until { $in.item | str starts-with "## " }
				| first
				| get index
			)

			$in
			| each {
				if ($in.item | is-empty) or ($in.index < $subheader) {
					return $in.item
				} else if ($in.item | str starts-with "> ") {
					$in.item
					| str replace -r "^" "|"
					| prepend ""
					| append ""
				} else if ($in.item | str starts-with "~") and not ($in.item | str ends-with "~") {
					$in.item
					| str replace -r "^" "| "
					| prepend ""
				} else if ($in.item | str starts-with "~") and ($in.item | str ends-with "~") {
					$in.item
					| str replace -r "^" "| "
					| append ""
				} else {
					$in.item
					| str replace -r "^" "| "
				}
			}
		} $in
		| to text
	}
}
| to text
| pandoc --standalone --from markdown-smart-tex_math_dollars --to man
| zstd --compress --force -19 -o $"($mandir)/($pkgbin).($mansection).zst"

 # man $"($mandir)/($pkgbin).($mansection).zst"
```

## entr

### entr man

> Automatically rebuilds the manpage on save

NOTE: only needed for testing purposes

```sh
fd maskfile.md | entr -cs "mask man > testman.1"
```

# ACKNOWLEDGEMENTS

- TBD

# BUGS

Report issues at: <https://github.com/yilisharcs/{{project-name}}/issues>

# AUTHOR

yilisharcs <yilisharcs@gmail.com>

# SEE ALSO

Website: <https://github.com/yilisharcs/{{project-name}}>

How to make manpages with markdown:

* <https://www.dlab.ninja/2021/10/how-to-write-manpages-with-markdown-and.html>
* <https://pandoc.org/MANUAL.html#metadata-blocks>

# LICENSE

Copyright (C) 2025 yilisharcs <yilisharcs@gmail.com>

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later
version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with
this program. If not, see <https://www.gnu.org/licenses/>.
