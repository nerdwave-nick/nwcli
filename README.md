# nwcli

An order-aware command-line grammar for Go, with help output and shell
completion derived from it. You declare a tree of commands, flags and operands
once; `nwcli` routes and parses `argv`, renders help pages, reports parse errors
as typed values and answers completion requests. It has no dependencies beyond
the Go standard library.

Distinctive feature: flags can begin an *item* that owns its own fields, so
`create --item A --note x --item B --note y` is valid while a second `--note`
inside one item is an error. Flag parsers that collect flags without order
cannot express this.

Supported shells for completion: bash 4.4+, zsh 5.8+ and fish 3.4+. PowerShell
is not supported.

## Usage

```go
package main

import (
	"fmt"
	"os"

	"github.com/nerdwave-nick/nwcli"
)

func main() {
	app := &nwcli.Command{Name: "app", Summary: "Do things", Commands: []*nwcli.Command{
		{Name: "create", Summary: "Create items", Flags: []nwcli.Flag{
			{Name: "item", Usage: "Item `TITLE`", Fields: []nwcli.Flag{
				{Name: "note", Usage: "Item `NOTE`"},
			}},
		}},
	}}
	if err := app.Validate(); err != nil {
		panic(err)
	}
	parsed, err := app.Parse(os.Args[1:])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2)
	}
	if parsed.Help { // -h, --help or the help command
		app.WriteHelp(os.Stdout, parsed.Path...)
		return
	}
	fmt.Printf("%+v\n", parsed)
}
```

Shell completion: serve `app.Complete(words, completers)` through
`nwcli.WriteCompletion` under a hidden command such as `__complete`, and print
the shell script with `nwcli.WriteScript(w, "bash", "app", "__complete")`.

## Provenance

`nwcli` was extracted, as a copy, from the `internal/nwcli` package of
[lit](https://github.com/nerdwave-nick/lit) (commit `3f2aed0`). lit's ADR 0001
explains why it exists: lit's argument grammar is order-dependent, and cobra
plus pflag collect flags without order, so lit ran two parsers that kept
disagreeing (repeatable-array flags, shadow copies of global flags, help-alias
rewrites, completion re-offering used flags). Replacing cobra with one grammar
that routing, help, error reporting and completion all read removed the
disagreement. The completion scripts are our own, so their behaviour (word
splitting at `:`, quoting, file fallback) is owned and tested in real
interactive shells (`shelltest`).

## Development

```sh
go vet ./... && go test -race ./...
```

The shell tests need bash, zsh and fish installed; without them they skip.
