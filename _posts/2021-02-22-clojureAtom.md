---
layout:     post
title:      Setting up Atom as an IDE for Clojure
date:       2021-02-21 23:39:00
summary:    The settings and files I use to program Clojure with Atom
categories: programming
---

My foray into functional programming began recently with Clojure. One of the first things you do when learning a new language is setup your environment to program effectively and efficiently. Emacs is the recommended IDE for Clojure as Emacs was built using Lisp and is therefore quite easy to setup. I'm not a fan of having to use keybinds to do basic functions like delete a line of code, which Emacs requires, and I love my regular IDE Atom too much.

Setting up Atom to program with Clojure effectively was quite the struggle, requiring configs from several sources. I'll go through the steps required below, fully tested and working at time of writing.

### Requirements

You'll need to install these first:

 1. Java Development Kit
 3. Atom
 2. Leiningen

Instructions to install the above can be found [here](https://github.com/mauricioszabo/atom-chlorine/blob/v0.10.2/docs/quickstart.md).

### Atom packages

A major part of Clojure is the REPL, Read Eval Print Loop. Chlorine is the up-to-date package that provides this inside Atom.

To install Chlorine, and other Atom packages for Clojure, run this in an elevated command prompt:

`apm install chlorine lisp-paredit parinfer platformio-ide-terminal`

Using Chlorine is most efficient with specific keymaps. I use the ones by Sean Corfield [here](https://github.com/seancorfield/atom-chlorine-setup). Download the repo as a zip (Code > Download zip) and extract it directly into your Atom folder (on Windows this is located at C:\Users\$USER\.atom). Then, open `keymap.cson` and change

`'atom-workspace atom-text-editor:not([mini])':`
to
 `'atom-text-editor[data-grammar="source clojure"]':`.

This last part I will largely copy from the Chlorine [quickstart](https://github.com/mauricioszabo/atom-chlorine/blob/master/docs/quickstart.md) guide as they explain it very concisely:

#### Connecting and running REPL

Chlorine needs to connect to a running REPL to evaluate code. The shortest and easiest way to do this is the following:

1.  Open up the terminal by pressing the + in the bottom left of //flight-manual.atom.io/hacking-atom/sections/debugging/

2. Change to a directory with a Clojure project or create one with and moves to its directory:

```bash
lein new app <APP_NAME>
cd <APP_NAME>
```

> You do not exactly _need_ to have a previous clj project or even a .clj file, but this step makes your life easier by handling the app's files

3.  Inside the app directory, start a REPL enviroment with an specific port to connect to it later

```bash
# Recomended, pure Socket REPL
JVM_OPTS='-Dclojure.server.myrepl={:port,5555,:accept,clojure.core.server/repl}' lein repl
# OR you can use nREPL
lein repl :start :port 5555
```

4.  In Atom, open the app folder or a single .clj file (or create one if you didn't do this before).

5.  Connect Chlorine to the running REPL, typing `ctrl+shift+p` and then, `Repl`, to open `Connect Socket Repl`. In the next window set host to `localhost` and port to `5555` (or any other port that you have specified).

6. That's it! Remember to open REPL through both the terminal and Atom anytime you want to program some Clojure.