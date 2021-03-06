## Location

The config file used by dotdrop is
[config.yaml](https://github.com/deadc0de6/dotdrop/blob/master/config.yaml).

Unless specified dotdrop will look in following places for its config file
and use the first one found

* current/working directory or the directory where [dotdrop.sh](https://github.com/deadc0de6/dotdrop/blob/master/dotdrop.sh) is located if used
* `${XDG_CONFIG_HOME}/dotdrop/`
* `~/.config/dotdrop/`
* `/etc/xdg/dotdrop/`
* `/etc/dotdrop/`

You can force dotdrop to use a different file either by using the `-c --cfg` cli switch
or by defining the `DOTDROP_CONFIG` environment variable.

## Variables

Multiple variables can be used within the config file to
parametrize following elements of the config:

* dotfiles `src` and `dst` paths (see [Dynamic dotfile paths](#dynamic-dotfile-paths))
* external path specifications
  * `import_variables`
  * `import_actions`
  * `import_configs`
  * profiles's `import`

`actions` and `transformations` also support the use of variables
but those are resolved when the action/transformation is executed
(see [Dynamic actions](#dynamic-actions),
[Dynamic transformations](#dynamic-transformations) and [Templating](templating.md)).

Following variables are available in the config files:

* [variables defined in the config](config-details.md#entry-variables)
* [interpreted variables defined in the config](config-details.md#entry-dynvariables)
* [profile variables defined in the config](config-details.md#entry-profile-variables)
* environment variables: `{{@@ env['MY_VAR'] @@}}`
* dotdrop header: `{{@@ header() @@}}` (see [Dotdrop header](templating.md#dotdrop-header))

As well as all template methods (see [Available methods](templating.md#template-methods))

## Symlink dotfiles

Dotdrop is able to install dotfiles in three different ways
which are controlled by the `link` config attribute of each dotfile:

* `link: nolink`: the dotfile (file or directory) is copied to its destination
* `link: link`: the dotfile (file or directory) is symlinked to its destination
* `link: link_children`: the files/directories found under the dotfile (directory) are symlinked to their destination

For more see [this how-to](howto/symlink-dotfiles.md)

## Dynamic dotfile paths

Dotfile source (`src`) and destination (`dst`) can be dynamically constructed using
defined variables ([variables and dynvariables](#variables)).

For example to have a dotfile deployed on the unique firefox profile where the
profile path is dynamically found using a shell oneliner stored in a dynvariable:
```yaml
dynvariables:
  mozpath: find ~/.mozilla/firefox -name '*.default'
dotfiles:
  f_somefile:
    dst: "{{@@ mozpath @@}}/somefile"
    src: firefox/somefile
profiles:
  home:
    dotfiles:
    - f_somefile
```

Make sure to quote the path in the config file.

## Dynamic actions

Variables ([config variables and dynvariables](#variables)
and [template variables](templating.md#template-variables)) can be used
in actions for more advanced use-cases.

```yaml
dotfiles:
  f_test:
    dst: ~/.test
    src: test
    actions:
      - cookie_mv_somewhere "/tmp/moved-cookie"
variables:
  cookie_dir_available: (test -d /tmp/cookiedir || mkdir -p /tmp/cookiedir)
  cookie_header: "{{@@ cookie_dir_available @@}} && echo 'header' > /tmp/cookiedir/cookie"
  cookie_mv: "{{@@ cookie_header @@}} && mv /tmp/cookiedir/cookie"
actions:
  cookie_mv_somewhere: "{{@@ cookie_mv @@}} {0}"
```

or even something like this:
```yaml
actions:
  log: "echo {0} >> {1}"
config:
  default_actions:
  - preaction '{{@@ _dotfile_key @@}} installed' "/tmp/log"
...
```

Make sure to quote the actions using variables.

## Dynamic transformations

As for [dynamic actions](#dynamic-actions), transformations support
the use of variables ([variables and dynvariables](#variables)
and [template variables](templating.md#template-variables)).

A very dumb example:
```yaml
trans_read:
  r_echo_abs_src: echo "{0}: {{@@ _dotfile_abs_src @@}}" > {1}
  r_echo_var: echo "{0}: {{@@ r_var @@}}" > {1}
trans_write:
  w_echo_key: echo "{0}: {{@@ _dotfile_key @@}}" > {1}
  w_echo_var: echo "{0}: {{@@ w_var @@}}" > {1}
variables:
  r_var: readvar
  w_var: writevar
dotfiles:
  f_abc:
    dst: ${tmpd}/abc
    src: abc
    trans_read: r_echo_abs_src
    trans_write: w_echo_key
  f_def:
    dst: ${tmpd}/def
    src: def
    trans_read: r_echo_var
    trans_write: w_echo_var
```

## All dotfiles for a profile

To use all defined dotfiles for a profile, simply use
the keyword `ALL`.

For example:
```yaml
dotfiles:
  f_xinitrc:
    dst: ~/.xinitrc
    src: xinitrc
  f_vimrc:
    dst: ~/.vimrc
    src: vimrc
profiles:
  host1:
    dotfiles:
    - ALL
  host2:
    dotfiles:
    - f_vimrc
```


## Ignore patterns

It is possible to ignore specific patterns when using dotdrop. For example for `compare` when temporary
files don't need to appear in the output.

* for [install](usage.md#install-dotfiles)
    * using `instignore` in the config file
* for [compare](usage.md#compare-dotfiles)
    * using `cmpignore` in the config file
    * using the command line switch `-i --ignore`
* for [update](usage.md#update-dotfiles)
    * using `upignore` in the config file
    * using the command line switch `-i --ignore`

The ignore pattern must follow Unix shell-style wildcards like for example `*/path/to/file`.
Make sure to quote those when using wildcards in the config file.

Patterns used on a specific dotfile can be specified relative to the dotfile destination (`dst`).

```yaml
config:
  cmpignore:
  - '*/README.md'
  upignore:
  - '*/README.md'
  instignore:
  - '*/README.md'
...
dotfiles:
  d_vim
    dst: ~/.vim
    src: vim
    upignore:
    - "*/undo-dir"
    - "*/plugged"
...
```

To completely ignore comparison of a specific dotfile:
```yaml
dotfiles:
  d_vim
    dst: ~/.vim
    src: vim
    cmpignore:
    - "*"
```

To ignore specific directory when updating
```yaml
dotfiles:
  d_colorpicker:
    src: config/some_directory
    dst: ~/.config/some_directory
    upignore:
      - '*sub_directory_to_ignore'
```

