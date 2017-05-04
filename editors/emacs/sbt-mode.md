---
layout: section
order: 7
title: sbt-mode
---

The mode provides basic functionality required for successfully interacting with [sbt](http://www.scala-sbt.org/) inside emacs. The core functionality includes:

- interacting with sbt shell and scala console
- compiling code and navigating to errors

This mode can be used independently of ENSIME.

## Installation

This mode will be installed automatically if you use ENSIME.

However, if you want to install separately, the preferred mechanism is via MELPA and `use-package` as per our [Learning Emacs](/editors/emacs/learning) guide:

```elisp
(use-package sbt-mode
  :commands sbt-start sbt-command
  :config
  ;; WORKAROUND: https://github.com/ensime/emacs-sbt-mode/issues/31
  ;; allows using SPACE when in the minibuffer
  (substitute-key-definition
   'minibuffer-complete-word
   'self-insert-command
   minibuffer-local-completion-map))
```

Start an sbt session with `M-x sbt-start` or send a command to the current sbt process with `M-x sbt-command`.

You might also want to customize some `sbt-mode` variables (run `M-x customize-mode RET sbt-mode`).

For example you can play with variable `sbt:scroll-to-bottom-on-output`. If set to `t` (which is a default value) sbt output buffer scrolls automatically to the last line if there is some new output. Setting this variable to `nil` will cause point remain on it's current position in a buffer when there is some new output. If set to `nil` and point is on the last line, it scrolls automaticaly with new output, but if there is compilation error (line beginning `[error] - ...`) it will stop on that line.

## Related Customizations

To work efficiently with `sbt-mode`, you may wish to customise your workflow with the built-in Emacs compile support:

- customise `compilation-skip-threshold`
- locally bind a key to `comint-bol`
- locally bind a key to `comint-accumulate`
- locally bind a key to `next-error`

## Tasks

### Searching

Most users may prefer to use [projectile](https://github.com/bbatsov/projectile) for these features and [we may remove them](https://github.com/ensime/emacs-sbt-mode/issues/53).

A special version of the `rgrep` command is available with `sbt-mode`.

- `M-x sbt-grep` greps files in the sbt project
- `M-x sbt-find-definitions` will search for the definition of the thing at point from the project `.scala` and `.java` files.
- `M-x sbt-find-usages` will search for occurrences of the id at point from the project `.scala` and `.java` files.

### Scala console

ENSIME users have direct access to a scala REPL for the project, but it is also possible to drop into an sbt `console` session using `sbt-mode`.

It is advised when typing multi-line code snippets, use `comint-accumulate` instead of `RET` for newlines. This way, if you need to modify the code, you can use `M-p` to recall the whole snippet for reworking.

You can also send a region of code from an other buffer in the same project. First set the mark to the other end of the region to send and the point (cursor) the other. Then run the `M-x sbt-send-region` command. `sbt-paste-region` will enter `:paste` mode of Scala REPL, so that multiline statement/expression will work as expected.

### Triggered execution

Triggered execution in sbt, the `~compile` or `~test` commands (when sbt is waiting with the message "press enter to interrupt"), can be interrupted by typing `C-c C-j` in `sbt-mode` (and `C-c C-b C-j` anywhere else).  Hitting just `RET` in an sbt buffer doesn't interrupt sbt because there isn't a recognized sbt prompt in `sbt-mode`.  Instead, hitting `RET` in an `sbt-mode` buffer will cause Emacs to complain that "Text is read-only".

However, you should consider using the more granular and lightweight commands of ENSIME to trigger actions interactively or with Emacs save hooks.  Triggered execution in sbt uses filesystem polling which is shown to be inefficient and expensive to use, especially on a large codebase. 

## Hydra

`sbt-mode` also offers [hydra](https://github.com/abo-abo/hydra) to speed up sbt interaction. This is focused mainly on running usual sbt commands `compile`, `test:compile`, `test`, `run` etc. on multiproject build. Hydra allows to execute these commands on per project basis.

### Basics

Run the `M-x sbt-hydra` command. This will start new sbt process in sbt buffer (if there is already sbt process running it will switch to it's buffer).

You will see message:
```
Please wait! sbt projects are loading. sbt Hydra will be ready soon.
```
while sbt is launching.

To generate hydra we need to get information about available projects. This is done by running `projects` command in sbt (this is done automatically as soon as sbt is ready). After this is done, you will see message:
```
Success hydra for projects (projectA, projectB, ...) created.
```

After you will see this message run `M-x sbt-hydra` command again and you will see hydra appearing at the bottom of the screen. By pressing `h` you can invoke help where you can learn more detais about Hydra features.

We recommend to bind `sbt-hydra` command to some convenient shortcut.

### Recommended usage

Hydra remembers last command which was executed and it can run this command automatically again on a save of an edited file. Of course it would be undesirable to run 10 commands when saving 10 edited files by `save-some-buffers` command so for this reason last sbt command will be run again only on a save of last edited file.

To enable this feature add `sbt-hydra:check-modified-buffers` into `before-save-hook`. For example:

```elisp
(add-hook 'sbt-mode-hook (lambda ()
                           (add-hook 'before-save-hook 'sbt-hydra:check-modified-buffers)))
```

To eliminate undesirable execution of last sbt command, there is customizable variable `allowed-files-regexp` containing regular expressions to detect files which when edited will cause executing last sbt command again. By default this variable contains two records: `.*.scala$` and `^routes$`.


### Customization

There exists 3 [Directory Local Variables](https://www.gnu.org/software/emacs/manual/html_node/emacs/Directory-Variables.html) which provides more customization for Hydra. They must be specified in `.dir-locals.el` file in the root of your sbt project.

#### 1. sbt-hydra:projects

This variable allows to specify list of projects to include in the Hydra. By default Hydra includes all sbt projects, which in case there is lots of them make working with Hydra cumbersome. This variable, when set, allows to specify subset of all projects.

```elisp
((sbt-mode
  (sbt-hydra:projects "api" "core" "ensime" "jerky" "server" "swanky" "util")))
```

#### 2. sbt-hydra:command-line-arguments

When using `run` sbt command we usually want to provide some command line arguments to the main method. This variable allows us to specify these arguments. We can provide different arguments for every projects. Example configuration looks like this:

```elisp
((sbt-mode
  (sbt-hydra:command-line-arguments . (
    ("restApi" . "myconf.conf development")
    ("jobs" . "delivery_flag")
    ("identityManager" . "myconf.conf 9001")))))
```

Here we have 3 sbt projects `restApi`, `jobs` and `identityManager`. For example when we execute `run` command through `r` hydra key we provide `myconf.conf development` arguments to `restApi` project main method.

#### 3. sbt-hydra:system-properties

When using `run` sbt command we may to set system variables for our program to use while running. This variable allows us to specify system variables without need to manipule environment variables. When set they will be provided to our main method by forking JVM and putting them into `javaOptions` before executing `run` itself.

```elisp
((sbt-mode
  (sbt-hydra:system-properties . (
    ("macwire" . ("-Dconf.file=myconf.conf" "-Xmx1G"))
    ("reader" . ("-Dconf.file=myconf.conf" "-Xmx1G"))))))
```

Here we have 2 sbt projects `macwire` and `reader`. For example for `macwire` project there will be set two system variables `-Dconf.file=myconf.conf` and `-Xmx1G`.
