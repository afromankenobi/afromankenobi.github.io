---
layout: post
title: Managing Secrets Securely on MacOS Keychain and ZSH
tags: [dev, mac, zsh]
---

Security is a big deal, especially in the context of software development. Although I haven't paid much attention to it in the past, my new job has thrust me into a world where security isn't just a buzzword—it's a requirement. Today, I'd like to share a useful tip for managing secrets securely on MacOS, specifically using ZSH.

## The Problem: Storing Github Packages Tokens

Let's say you need to access a private NPM repository, for which you have a Github packages token. The conventional way to do this is to copy and paste the token into your `~/.npmrc` file so that your JavaScript packages work. However, storing passwords in plaintext is a major security risk. This brings us to the solution: MacOS Keychain.

## Storing the Token in MacOS Keychain

Firstly, add the token to your MacOS Keychain by running the following command:

```zsh
security add-generic-password -a "$USER" -s 'gh_packages_token' -w 'TOKEN_VALUE'
```

Replace `TOKEN_VALUE` with your actual token.

**Pro Tip**: Prepend the command with a space to avoid storing it in your `.zsh_history`.

## Fetching the Token When Needed

Suppose your code is stored in `~/src/`. You'd want the token to be available as an environment variable, say `GH_PACKAGES_TOKEN`. To achieve this, add the following code to your `~/.zshrc`:

```zsh
autoload -U add-zsh-hook

load_token() {
  local work_folder_pattern="$HOME/src/*" # Change this to the path where your work projects reside
  if [[ "$PWD" =~ $work_folder_pattern ]]; then
    export GH_PACKAGES_TOKEN=$(security find-generic-password -a "$USER" -s "gh_packages_token" -w)
  fi
}

add-zsh-hook chpwd load_token
load_token
```

## Using the Token in `~/.npmrc`

Finally, incorporate this environment variable in your `.npmrc` as follows:

```zsh
//npm.pkg.github.com/:_authToken=${GH_PACKAGES_TOKEN}
```

This simple trick sets the `GH_PACKAGES_TOKEN` only in directories where it's needed—in my case, within `~/src`. Now you can manage your tokens more securely without compromising functionality.
