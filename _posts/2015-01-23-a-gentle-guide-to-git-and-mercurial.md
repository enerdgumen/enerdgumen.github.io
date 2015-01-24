title: A gentle introduction to Git and Mercurial
date: 2015-01-23 21:35
tags: tools
ignore: true

This is a little guide thought for who has never used Git or Mercurial in his life. I'll explain the essential use case useful in daily work.

## Setup the environment

After you have installed the versioning system a minimal configuration is required.
For Git create the file `.gitconfig` in your home with this content:

    ::ini
    [user]
        email = email@example.org
        name = Your Name
    [push]
        default = simple

For Mercurial create the file `.hgrc` with the following:

    ::ini
    [ui]
    username = Your Name <email@example.org>


## Beginning a project

Generally your contribute borns cloning the project in your host, this operation copies all files and history on your machine and initialize the local repository:

| Git | Mercurial |
|-----|-----|
| `git clone <repo>` | `hg clone <repo>` |
