# Shell scripting style guide

A collection of best practices and patterns for common tasks.

## Which shell?
bash should be used as the script interpreter, with the shebang line: `#!/usr/bin/env bash`
Your script should be marked executable (`chmod +x`) when committed to git, and should have a `.sh` extension.

See [this discussion on the pros and cons of using env in the shebang](http://unix.stackexchange.com/a/29620). 


## shellcheck
Run the [shellcheck](https://github.com/koalaman/shellcheck) static analysis tool on all you shell scripts. Use a locally installed shellcheck, not the web-based one: `brew install shellcheck; shellcheck /path/to/my.sh`

shellcheck's defaults are great and should be heeded in nearly every case.

## Writing safer bash scripts
Most bash scripts will run "safer" with the following options set at the top, right after your shebang line:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

These options do the following:

1. `set -e`, or `set -o errexit`, tells bash to exit immediately if any command exits with a nonzero exit status.
With errexit, you don't need to fill your scripts with ampersands (&&) or conditional exits (|| exit 1) to handle errors
2. `set -u`, or `set -o errunset`, tells bash to exit with an error if you try to access any variable that isn't defined
Attempting to access an unset variable is often the result of capturing the output of a command that failed but still returned a 0 exit code
3. `set -o pipefail` tells bash to do the same thing as errexit, but even when one half of a pipe fails. For example `false | true` is OK with just errexit, but fails with pipefail set.

If you want to do any custom error handling in your script, or do special tricks with the [internal field separator](http://www.tldp.org/LDP/abs/html/internalvariables.html), you may omit these.

If you want to use these as safeties by temporarily disable them to do something on your own, you can unset any option with the `+` parameter to set, as in `set +o errexit` or `set +u` to disable errexit and errunset until the next `set +e` or `set +o errunset` respectively.

See [Unofficial Bash strict mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/) for someone else's words on this.

## Portability
Many common commands differ in functionality between their BSD versions, found on OS X hosts, and their GNU versions, found on most Linux hosts.

You can avoid a lot of this headache by [using Homebrew to install the GNU versions](http://apple.stackexchange.com/a/69332) or by testing your scripts in a container/GNU environment.

Some common ones include:

* tar, bsdtar – tarballs created on OS X or with bsdtar may error out when untarred by GNU tar
* date – has many more options in its GNU version, including the handy `date -d 'now + 2 days'` syntax
* awk – GNU awk has lots of handy functions like strftime that aren't available in BSD awk
* readlink – GNU readlink has the `-m` flag for resolving symlink loops. BSD does not.


## Finding files

* [Don't parse ls output](http://mywiki.wooledge.org/ParsingLs)
* Remember that filenames can include exotic characters, including newlines, tabs, UTF-8 chars, and anything short of a null byte.

Use a pattern like this to safely read matching files into a bash array and operate on them:

```bash
# fills JARS with an array of results*.txt filenames in /var/db/myapp
JARS=()
while read -r -d $'\0' jar; do
  JARS+=("$jar")
done < <(find /var/db/myapp -maxdepth 1 -regex ".*/results.*\.txt$" -type f -print0)

# echoing matched files
for jar in "${JARS[@]}"; do
  echo "$jar"
done
```
Adjust find parameters as needed, with particular attention to `-maxdepth`, `-regex`, and `-type`.

## Parsing JSON
Use [jq](https://stedolan.github.io/jq/), but consider using a language with a JSON parsing lib at this point.


## Checking running processes
Use [pgrep](http://manpages.ubuntu.com/manpages/man1/pgrep.1.html) instead of grepping ps output.


## Creating temporary files and directories
Use [mktemp](http://manpages.ubuntu.com/manpages/man1/mktemp.1.html) instead of rolling your own naming and logic

## Cleaning up after your script
Use [bash traps](https://linux.die.net/Bash-Beginners-Guide/sect_12_02.html) to clean up any temporary files created by your script, or operate on any remote resources that need to be destroyed when your script exits.

See this blog post: [How "Exit Traps" Can Make Your Bash Scripts Way More Robust And Reliable](http://redsymbol.net/articles/bash-exit-traps/)

## Secret storage and use
Read sensitive data and credentials from environment variables, or config files pointed to by environment variables, whenever possible.

## Use long options
In scripts, if a command has short and long options, such as `-s` versus `--silent`, use `--silent` in your script so it's more clear to someone unfamiliar with all the short options for a command.

## Retrieving remote files
`curl` tends to be more widely available by default, so use it in preference to `wget`

When running in a script, there are a few curl parameters you should consider using:

* `–-silent --show-errors` – this combination will suppress the progress bar from standard out but still print human readable messages when curl encounters and error
* `--max-time N` – how many seconds, total, to allow the entire curl operation to take. This is most appropriate when you are usng curl to hit an API endpoint that's expected to respond in a reasonable amount of time (<= 10 seconds), and not when you're using curl to retrieve a large (multi-megabyte) file.
* `--connect-timeout N` – how many seconds to wait to establish a TCP connection to the host. This can typically be very short. If a TCP connection isn't established after 2 seconds, something is usually very wrong.

## Set defaults
You can use [parameter substitution](http://www.tldp.org/LDP/abs/html/parameter-substitution.html) like this to allow a variable within your script to be overridden by the environment, but still use a sane default if not set externally.

```bash
#!/usr/bin/env bash
MYSCRIPT_TIMEOUT="${MYSCRIPT_TIMEOUT:-10}"
sleep $MYSCRIPT_TIMEOUT
```

If someone called this as `./defaults.sh` it would sleep for 10 seconds, and if someone called this as `MYSCRIPT_TIMEOUT=60 ./defaults.sh` it'd sleep for a minute.

## Checking whether a variable is defined, unset, or empty
Deviate from this only if you have a very particular requirement:

```bash
# use -n as a positive check for defined variables rather than the confusing ! -z negation
if [ -n "$var" ]; then
  echo "\$var is set and its value is $var"
else
  echo "\$var is unset or zero-length"
fi

# use -z as a positive check for unset/zero length instead of the confusing ! -n negation
if [ -z "$var" ]; then
  echo "\$var is unset or zero-length"
else
  echo "\$var is set and its value is $var"
fi
```

See [the test man page](http://manpages.ubuntu.com/manpages/man1/test.1plan9.html) for details on these letter flags

## Multiline strings
If you find yourself escaping newlines and tabs or setting variables to multiline strings, consider [using a here document instead](http://www.tldp.org/LDP/abs/html/here-docs.html).

## Miscellaneous
* [Avoid bash pitfalls](http://mywiki.wooledge.org/BashPitfalls)
