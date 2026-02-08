+++
date = 2026-02-08T19:09:42+05:30
draft = false
title = "Jujutsu: Managing workspaces"

[taxonomies]
tags = ["git", "jujutsu", "worktrees"]
series = ["jujutsu-tips-and-tricks"]
+++

One of my favorite features in [Jujutsu][jj] is its native support for workspaces (similar to [Git](https://git-scm.com/) worktrees). They let you have multiple working copies of the same repo, which is super handy for running tests in the background while you keep coding, or for juggling multiple features without constantly switching contexts.

While the built-in `jj workspace` commands are great, they can be a bit verbose. I use a few simple conventions and aliases to speed things up and make my workflow smoother.

> Your repository must have been initialized with [Jujutsu][jj] version **0.38.0** or later. Older repositories will not have the necessary workspace metadata for the **default** workspace.[^1]

#### Creating a workspace

This usually requires specifying a name and a path. I prefer a convention where my secondary workspaces are siblings of my default workspace directory, named with a suffix. Assuming your **default** workspace is at `~/project`, I prefer the new workspace named `foo` to be at `~/project.foo`.

```bash
# Get the path of the default workspace
repo_dir=$(jj workspace root --name default)
# Create the new workspace at the same level with a suffix
jj workspace add --name foo $repo_dir.foo
```

To simplify this, you can define a Jujutsu alias in `~/.config/jj/config.toml`:

```toml
[aliases]
wa = [
  'util', 'exec', '--', 'bash', '-c',
  'jj workspace add --name $0 $(jj workspace root --name default).$0'
]
```

Then, you can simply run:

```bash
jj wa foo
```

#### Switching between workspaces

If you have several active workspaces, it can be hard to remember their names or paths.

```bash
# List all workspaces to find the one you want to switch to
jj workspace list
# Get the path of the workspace
ws_dir=$(jj workspace root --name foo)
# Change directory to the workspace
cd $ws_dir
```

You can combine the above with [fzf](https://github.com/junegunn/fzf) to interactively select a workspace from the list.

To simplify this, you can define a Jujutsu alias in `~/.config/jj/config.toml`:

```toml
[aliases]
wo = [
  'util', 'exec', '--', 'bash', '-c',
  'jj workspace root --name $(jj workspace list | fzf | cut -d":" -f1)'
]
```

Then, you can explore and switch workspaces with a single command:

```bash
cd $(jj wo)
```

#### Deleting a workspace

When you're done with a workspace, `jj workspace forget` stops tracking it in the repository. However, it leaves the files on disk. To clean up, you can combine it with `rm -rf` to delete the workspace directory.

```bash
# Forget the workspace
jj workspace forget --name foo
# Get the path of the default workspace
repo_dir=$(jj workspace root --name default)
# Remove the workspace directory
rm -rf $repo_dir.foo
```

To simplify this, you can define a Jujutsu alias in `~/.config/jj/config.toml`:

```toml
[aliases]
wff = [
  'util', 'exec', '--', 'bash', '-c',
  'jj workspace forget $0 && rm -rf $(jj workspace root --name default).$0'
]
```

Then, you can simply run:

```bash
jj wff foo
```

#### Conclusion

Using a simple naming convention coupled with these aliases makes workspaces feel like a natural part of my workflow, not a chore. It lets me switch tasks instantly, helping me stay focused while keeping my work organized.

I've been using this approach with my [jj aliases](https://gist.github.com/pksunkara/622bc04242d402c4e43c7328234fd01c) for a while, and it works great!

---

[^1]: I am the one who contributed that to [Jujutsu][jj] in [#6858](https://github.com/jj-vcs/jj/pull/6858)

[jj]: https://github.com/jj-vcs/jj
