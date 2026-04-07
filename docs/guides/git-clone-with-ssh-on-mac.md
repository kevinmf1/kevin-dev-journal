---
sidebar_position: 1
title: "Git Clone Using SSH on macOS"
description: "A step-by-step guide on how to use registered SSH keys on your Mac to clone Git repositories"
---

# Git Clone Using SSH on macOS

When working with multiple Git accounts (e.g., personal and work), you may have several SSH keys registered on your Mac. This guide walks you through how to use a specific SSH configuration to clone repositories.

## Prerequisites

- macOS with SSH configured
- Git installed
- At least one SSH key pair registered on your machine

## Step-by-Step Guide

### 1. Check Your SSH Configuration

Open your terminal and run the following command to view your registered SSH configurations:

```bash
cat ~/.ssh/config
```

This will display your SSH config file, which typically looks something like this:

```text
Host my-personal-github
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

Host my-work-bitbucket
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

### 2. Identify the Correct SSH Host

From the output, look for the SSH host alias that corresponds to the account you want to use.

For work-related repositories, you would typically use the host configured for your work account — for example, `my-work-bitbucket`.

### 3. Clone Using the SSH Host Alias

When cloning a repository, replace the default hostname in the SSH URL with your SSH host alias.

**Default SSH clone URL:**

```bash
git clone git@bitbucket.org:my-company/awesome-project.git
```

**Modified URL using your SSH host alias:**

```bash
git clone git@my-work-bitbucket:my-company/awesome-project.git
```

The general format is:

```
git clone git@<ssh-host-alias>:<organization>/<repository-name>.git
```

### 4. Wait for the Clone to Complete

Once you run the command, Git will use the SSH key associated with your chosen host alias to authenticate. Wait for the cloning process to finish:

```text
Cloning into 'awesome-project'...
remote: Enumerating objects: 5432, done.
remote: Counting objects: 100% (5432/5432), done.
remote: Compressing objects: 100% (3210/3210), done.
Receiving objects: 100% (5432/5432), 25.00 MiB | 5.00 MiB/s, done.
Resolving deltas: 100% (4321/4321), done.
```

### 5. Verify the Remote Configuration

After cloning, navigate into the cloned repository and verify that the remote URL is using the correct SSH host alias:

```bash
cd awesome-project
git remote -v
```

You should see output like:

```text
origin  git@my-work-bitbucket:my-company/awesome-project.git (fetch)
origin  git@my-work-bitbucket:my-company/awesome-project.git (push)
```

This confirms that all future `git pull`, `git push`, and `git fetch` operations will use the correct SSH key automatically.

## Troubleshooting

### Permission Denied (publickey)

If you encounter a `Permission denied (publickey)` error, make sure:

1. Your SSH key is added to the SSH agent:
   ```bash
   ssh-add ~/.ssh/your_private_key
   ```
2. Your public key is registered on the remote Git service (e.g., Bitbucket, GitHub).
3. The `IdentityFile` path in your `~/.ssh/config` is correct.

### Testing Your SSH Connection

You can test your SSH connection before cloning:

```bash
ssh -T git@my-work-bitbucket
```

A successful connection will display a welcome message from the Git service.

## Summary

| Step | Action |
|------|--------|
| 1 | Run `cat ~/.ssh/config` to view your SSH configurations |
| 2 | Identify the correct SSH host alias for your account |
| 3 | Replace the hostname in the clone URL with your SSH host alias |
| 4 | Run `git clone` and wait for it to complete |
| 5 | Verify the remote URL with `git remote -v` |

:::tip
Using SSH host aliases is especially useful when you have multiple accounts on the same Git hosting service. Each alias can point to a different SSH key, making it seamless to switch between personal and work repositories.
:::
