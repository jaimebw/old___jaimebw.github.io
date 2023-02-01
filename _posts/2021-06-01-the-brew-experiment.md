---
layout: post
title: The brew experiment
subtitle: Reinventing the homebrew wheel 
tags: [python, zsh, Homebrew ]
comments: true
---
## Introduction
Recently, my laptop has been experiencing performance issues, so I considered formatting my SSD to see if it would improve its performance. The problem with this process is that I would need to reinstall all my software, which can be tedious. Fortunately, I have Homebrew installed and some knowledge of zsh scripting and Python, so I decided to use these tools to automate the process.

## Experiment
My goal was to create a .py script that could generate a .zsh script from a .txt file containing the software I previously had installed. The first step was to get the software installed by Homebrew. I ran the following command in Terminal:
```bash
brew list
```
And we get the result on the first figure. The next step is to copy all the results inside a .txt and to code a way to parse the .txt into something that we can use with Python.

![brew_list](/assets/img/brew_list.png).

The .txt looks like this:  

```
Formulae
aom			libx11
asciidoc		libxau
bdw-gc			libxcb
berkeley-db		libxdmcp
boost			libxext
brotli			libxrender
c-ares			libyaml
cairo			lightgbm
cmake			little-cms2
dav1d			lua
docbook			lzo
emacs			m4
ffmpeg			mpdecimal
flac			mpfr
fontconfig		ncurses
```
And it looks the same with the casks.  

This is a little bit trivial. The next snippet lets us parse the .txt:
```python
path_txt = "brew_packages.txt"

with open(path_txt,"r") as f:
    lines = f.readlines()

lines = "\n".join(lines)
lines = re.sub("\s+",",",lines.strip())
lines = lines.split(",")

formulae_n = lines.index("Formulae")
cask_n = lines.index("Casks")

# Create list with casks
casks = lines[cask_n+1:]
casks = ("\n".join('"'+item+'"'for item in casks))

# Create lists with formulaes
formulae = lines[formulae_n+1:ocask_n]
formulae = ("\n".join('"'+item+'"'for item in formulae))
```
Now we have two list containing the different names of the packages, now we need to create a .zsh script that can be executed directly through the terminal with Python. 
The [zsh](https://es.wikipedia.org/wiki/Zsh) is the default shell in Mac Os and its scripting language its quite simple to use. A common "hello_world" script looks like this:
```bash
echo "Hello_world"

```
What we want to generate is Python script capable of creating a zsh script every time our txt with the packages changes.

### Zsh scripting
The zsh scripts will use two arrays and two different for loops. Arrays are declared this way:
```bash
declare -a CASKS=(
"blablabla"
)
declare -a FORMULAE=(
"blebleble"
)
```
And the for loops will use the ```brew install``` and ```brew install --cask```commands. The loops will look like this:
```bash
echo "Installing casks"
for i in "${CASKS[@]}"; do
  echo "Installing $i"
  brew install --cask $i
done
```
After having the zsh structure defined, we shall add this to a Python script to do all this process automatically every time we change ```brew_packages.txt```

### Python script
The Python script will concatenate a couple of different string and parameters to create a zsh script.
```python
# Declare arrays in .zsh
cask_final = "CASKS=(\n"+casks+"\n)"
formulae_final = "FORMULAE=(\n"+formulae+"\n)"

# Add the declares
text_cask = "declare -a "+ cask_final
text_formulae = "declare -a "+formulae_final

# Create the final script
zsh_script ="#!/bin/zsh\n"+ text_cask +'\n'+text_formulae+'\n'+ """
echo "Installing casks"
for i in "${CASKS[@]}"; do
echo "Installing $i"
brew install --cask $i
done
echo "Casks installed"
echo "Installing formulae"
for i in "${FORMULAE[@]}"; do
echo "Installing $i"
brew install $i
done
echo "Finish Install"
"""
#print(zsh_script)
# Create the zsh script
f = open("install_brew_confing.zsh","x")
f.write(zsh_script)
f.close()
```
Afterwards, we can run the ```install_brew_confing.zsh``` directly from the terminal using ```zsh install_brew_confing.zsh``` and see the end results.
![end_result](/assets/img/end_zsh.png)
It works! The full .py script is available [here](https://gist.github.com/jaimebw/12b0ef20157d7771ecffc86508e0459a).

## Final thoughts
Anyway, after all this long process(which was fun to do), I found out that there is a command available with Homebrew that already does this automatically, creating a Brewfille that can be saved for future recoverys 🤪 . The command is:
```bash
brew bundle dump
# this creates the Brewfile
brew bundle
# this install whatever the Brewfile has 
```
