---
title: Add new repos to GitHub without leaving the terminal
pubDate: 2023-12-19
description: It took me a surprisingly long time to explore the GitHub CLI.
tags: ['git', 'GitHub']
isDraft: true
slug: github-cli
---

When I started coding, using the terminal was intimidating and often hindered my progress. Overcoming that fear has brought significant benefits. Although I still instinctively reach for a GUI tool, I'm gradually breaking that habit. As a result, I'm getting things done faster and enjoying coding much more.

Many online coding courses try to shield students from the terminal for as long as possible. I personally believe this is a big mistake and it held me back during my first unsuccessful attempt to learn coding.

Of course you should read the [docs](https://cli.github.com/manual/), but below is my guided version of setting up and using the GitHub CLI. Hopefully, it can help you overcome your mental barriers and demonstrate why using the CLI is liberating!

## Getting started

To get started you will need to have `git` and `gh cli` installed, and be authenticated on your machine.

1. [Install git](https://github.com/git-guides/install-git) on your machine (already installed if you are on Mac)
1. Get a GitHub account if you don't have one
1. [Install the GitHub CLI](https://github.com/cli/cli?tab=readme-ov-file#installation). Using [homebrew](https://brew.sh/) on Mac that was simply running `brew install gh` in the terminal
1. Run `gh auth login` in your terminal and follow the prompts to authenticate on GitHub

## Create a remote repository with GitHub CLI

With the GitHub CLI we can now create a repo on GitHub and clone it locally without leaving our terminal.

```bash
# cd into the directory where you put your projects
cd repos/

# create a new private repo called 'my-project' and clone it into a new dir
gh repo create my-project --private --clone

# cd into your new project directory
cd my-project/
```

But what if you have have already created a new project locally, and even have git initialised already?

No problem, if you use `gh repo create` without any arguments, you will be taken through an interactive setup process. Instead of creating remote repo and cloning it locally, you can choose to create a remote repo from the content of your local project.

```bash
# cd into your project directory
cd repos/my-project/

# Walk through creating the repo interactively
gh repo create
	# Wizard steps:
	# - select 'Push an existing local repo...'
	# - select the current directory (.)
	# - confirm the repo name (defaults to directory name)
	# - add description if needed
	# - select visibility
	# - yes add a remote
	# - accept default remote name ('origin')
	# - yes push existing commits to origin

```

But adding a few arguments skips the interactive process and speeds things up considerably.

Here we can create a new public repo from the current directory and push any existing commits in one line! (Note that we set the repo name to be the current directory name thanks to the argument `--source=.`)

```bash
# cd into your project
cd repos/my-project

# create a new public repo and push commits
gh repo create --public --source=. --push
```

You should absolutely check out the docs for [`gh repo create`](https://cli.github.com/manual/gh_repo_create) to see all the possible options!

## What else can I do with the GitHub CLI?

There is of course much more that the GitHub CLI can do, like working with PRs, Issues and Projects, and all the basic admin you might need to do with GitHub.

### Renaming a repo

Have you ever had to rename a repo?

It's a simple job, but using github.com to change a repo name is a [multi-step process](https://docs.github.com/en/repositories/creating-and-managing-repositories/renaming-a-repository) that requires you to subsequently come back to your local repo and update the remote URL. For me that just sounds like an opportunity for mistakes to happen.

With your new CLI skills it is as simple as writing `gh repo rename [<new-name>]` from within your project directory. Everything is handled for you in an instant, and you never had to touch your mouse!

### Creating a gist

You often need to share code in slack/discord or online forums to get or give feedback. For lengthy code where an in-chat code block is not appropriate, a GitHub gist is a great option. But clicking all over the GitHub website adds a bunch of friction to this process.

Using the GitHub CLI you can generate a public gist from a local file and have the public URL to share in about 5 seconds with one command:

```bash
gh gist create myLocalFile.ts -p -w -d "My Typescript file that isn't working"
```

- The `-p` flag means it will be a public gist
- The `-w` flag means the url will open in your browser
- The `-d` flag and subsequent text adds that text description to the gist

Instead of sharing a whole file, you could share just some code from your clipboard. With the commands below we create a new temp file from your clipboard content, run the create gist command, and remove the temp file.

```bash
# create a temp file 'example.js' with the content from your clipboard
pbpaste > example.js

# create a new gist from the file, make it public, open it in your browser
gh gist create example.js -p -w

# remove the local temp file
rm example.js
```

In fact I have this saved as a one line snippet in [Raycast](https://www.raycast.com/) so all I have to do is type `create-gist` in my terminal and it will run the above in one command:

```bash
pbpaste > example.js && gh gist create example.js -p -w && rm example.js
```

## Conclusion

As you can see I have become a definite _terminal-enjoyer_ since breaking my mental barrier.

I think the `gh cli` is a great option to start leaning into the terminal and feel more in-control of your coding environment - give it a shot!
