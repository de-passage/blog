---
layout: post
title:  "Interactive edition in Bash scripts"
tags: bash script shell linux
---

I've long been interested in mimicking Git's interactive edition feature in my own scripts. I write a lot of Bash at work, to automate the tedious parts of the job. Sometimes, however, I find myself reaching the limits of command edition in the shell. I may have to fill a large number of entries, or type in some structured data, which is unwieldy in a command line edition.

Git has an elegant solution to this problem: it allows you to edit parts of what will form the final instructions in your favorite editor. For example it will open a window with instructions on how to perform an interactive rebase or let you fill in your commit messages in an editor. In this article we'll reproduce this behavior in a custom Bash script and deal the with the problem that arise when using everybody's favorite editor: Vim.

## The script

To demonstrate the idea, we'll write a simple script that prompts the user to fill various fields of a cURL command. We'll show a window with instructions in an editor and let the user fill in the details of the command. When the editor is exited, we'll parse the input and run cURL. We'll start with nano as our editor and make the choice of the editor configurable one we have a working foundation. 

The first thing we need is a file. Interactive text editors typically work with files, and while some may be able to stream data back and forth, we want to be as agnostic as possible with regard to the user's tooling. We'll simply write the template we want our user to fill in a file, wait for the editor process to exit then parse the content. Of course, we don't want to pollute the working directory, nor keep files that we don't need anymore: our file shall be temporary.
Linux distributions typically come with a tool called `mktemp` which creates a file in the temporary directory (defined by the variable `$TMPDIR`, or `/tmp` if it isn't defined). `mktemp` can be configured further but in this case the defaults are enough. Temporary files aren't immediately deleted however, so to avoid multiplying temporary files on the disk in case the machine isn't rebooted often, we'll manually delete our file when we're done with it. This is done with the `trap` instruction to handle cases when the script doesn't exit properly (for example if the user hits `Ctrl+C` during the execution).

```bash
#!/usr/bin/env bash

# mktemp prints the name of the file it creates on stdout
tmp_file=$(mktemp)

# Write the desired information on the file
echo '# Fill the following fields to complete the HTTP request. body must be the last field
target: https://
method: POST
content type: application/json
ignore SSL certificate: false
body: {
  "key": "value"
}
' > "$tmp_file"

# When the script exits, delete the temporary file
trap "rm $tmp_file" EXIT

# Open the editor to let the user do their thing
nano "$tmp_file"

# Building the arguments with sed. Obviously you may use whatever you want to parse the result
sed -n -i "$tmp_file" -e '

# Remove all the comments from the file
/^#/ d

# The target of the HTTP request is not prefixed on the command line
s/target:\s*//

# The option for the HTTP method is -X. If its empty well leave curl take the default value (GET)
s/^method:\s*\(\w\+\)/-X \1/

# I often work with custom servers that dont give proper SSL certificates, the option to ignore them is -k
/^ignore SSL certificate:/ {
  /:\s*true\b/ s/.*/-k/
  /:\s*true\b/! d
}

# Headers are prefixed with -H, I only handle Content-Type in this example
s/content type:\s*\(\S*\)/-H "Content-Type: \1"/

/body:/,$ {
  # Data is prefixed with -D. After that is a single quotation mark. Bash is weird.
  s/^body:/-D '"'"'/
  s/^\s*//
  $ s/$/'"'"'/
}

# Build a single line from the file. Not strictly necessary but will look good in the output
$! {
  s/$/ /
  H
}

$ {
  H
  x
  s/\n//gp
}
'

# This is an exemple, so I'll only output the command to prove that it's working.
echo "curl $(cat "$tmp_file")"
```

And that's it, fairly simple. All that's left to do is to select an editor based on the user's preference. Again, it's very simple, we'll check whether the user has defined the `$EDITOR` variable, and if not we'll fall back to `nano`.
```bash
...

# -x checks that the parameter is executable
if ! [[ -x "$EDITOR" ]]; then
  EDITOR=nano
fi
"$EDITOR" "$tmp_file"

...
```

And that's it, you may use the editor you want by changing the appropriate environment variable.

Quick note for VSCode users, in order for the editor to do what you want, you should define the variable as follow: `export EDITOR='code --wait'`

## The curious case of Vim

This all work well, until you try to redirect the output of your script with Vim as your editor. Try for example:
```bash
# Assuming you named your file interactive_curl.bash
EDITOR=vim ./interactive_curl.bash > output.txt
```

Best case scenario, you'll be met with a dry comment saying "Warning: Output is not to a terminal", and your terminal will be broken when you exit vim. Worst case scenario your shell will hang until forceful termination...  
As the warning states, this is because vim expects its input and output file descriptors to be a terminal, and in our case we redirected the output of all the commands in our script to a file...  
Fortunately, Linux systems have a pseudo descriptor that represents the terminal: `/dev/tty`. All we have to do is to redirect both inputs and outputs: 
```bash
"$EDITOR" "$tmp_file" >/dev/tty </dev/tty 
```

And we're done! We can now redirect safely everything to/from our script and edit our commands from within Vim. 
