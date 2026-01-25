+++
date = 2026-01-25T17:46:03+05:30
draft = false
title = "Jujutsu: Keeping a file untracked"

[taxonomies]
tags = ["git", "jujutsu"]
series = ["jujutsu-tips-and-tricks"]
+++

Coming from [Git](https://git-scm.com), I am used to a workflow where I can create a file in my repository *(maybe a temporary script, or a local configuration file)* and just never `git add` it. It sits there, listed as untracked in `git status`, invisible to my commits but available for my use. I never needed to add it to `.gitignore` since I simply never staged it.

When I started using [Jujutsu](https://github.com/jj-vcs/jj), I ran into a small friction point with this habit. Jujutsu is designed to be more automatic than Git. By default, it automatically tracks all files in your working copy that aren't ignored by `.gitignore`. This means as soon as I create that temporary file, it will be included in the next snapshot, and `jj status` shows it as added.

This behavior is controlled by the `snapshot.auto-track` configuration setting. You can customize this setting using [file patterns](https://docs.jj-vcs.dev/latest/filesets/#file-patterns) to exclude specific files. For example, to keep a file named `foo.txt` untracked, you can run:

```bash
jj config set --repo snapshot.auto-track "all() & ~foo.txt"
# Run the following if the file has already been snapshotted
jj file untrack foo.txt
```

You can still manually track it later if needed using `jj file track foo.txt`.

To untrack additional files later, you can append a new pattern to the existing configuration:

```bash
jj config set --repo snapshot.auto-track "$(jj config get snapshot.auto-track) & ~bar.txt"
```

If this is too much to type every time, you can define a Jujutsu alias in `~/.config/jj/config.toml`:

```toml
[aliases]
ignore = [
  'util', 'exec', '--', 'bash', '-c',
  'jj config set --repo snapshot.auto-track "$(jj config get snapshot.auto-track) & ~$0" && jj file untrack $0'
]
```

Then, simply run:

```bash
jj ignore foo.txt
jj ignore bar.txt
```

I've been using this approach with my [jj aliases](https://gist.github.com/pksunkara/622bc04242d402c4e43c7328234fd01c) for a while, and it works great!
