---
title: "trk: A Git Wrapper for Managing Dotfiles with Encryption"
date: 2025-10-01T16:52:00+02:00
description: "Manage your dotfiles and encrypt sensitive files seamlessly with trk, a lightweight Git wrapper inspired by yadm and transcrypt"
---

Managing dotfiles across machines while keeping sensitive configuration files secure is a common challenge. Meet **trk** (track) - a Git wrapper that combines repository management with transparent file encryption.

![trk](/trk.jpg)

## What is trk?

Trk is a lightweight Git wrapper inspired by [yadm](https://github.com/yadm-dev/yadm) and [transcrypt](https://github.com/elasticdog/transcrypt). It solves two key problems:

1. **Dotfile management**: Track configuration files in your home directory without cluttering it with a `.git` folder
2. **Transparent encryption**: Automatically encrypt sensitive files using OpenSSL with Git clean/smudge filters

## Quick Start

Install trk with a single command:

```bash
wget https://raw.githubusercontent.com/TheoBrigitte/trk/refs/heads/main/trk
install -D -m 755 trk ~/.local/bin/trk
```

Initialize a new repository:

```bash
trk init
```

Or clone an existing one:

```bash
trk clone <url>
```

Set up tracking from an existing repository:

```bash
trk setup
```

## Global Repository Mode

The killer feature of trk is its global repository mode. Instead of creating a Git repository directly in your home directory (which could lead to accidentally committing files from other projects), trk stores the repository in a separate location while tracking files in your home directory:

```bash
trk init --worktree $HOME
trk clone --worktree $HOME <url>
```

Now you can use `trk` commands just like `git`, but without worrying about polluting your home directory with Git metadata.

## Transparent File Encryption

Trk leverages [Git clean/smudge filters](https://git-scm.com/book/ms/v2/Customizing-Git-Git-Attributes#filters_a) to encrypt files automatically. Files are stored encrypted in the repository but decrypted on the fly when checked out.

Mark files or patterns for encryption:

```bash
trk mark ~/.ssh/config
trk mark '*.key'  # Don't forget the quotes!
```

The encryption uses OpenSSL with AES-256-CBC and PBKDF2 key derivation, with a unique salt per file for enhanced security.

View and modify encryption settings:

```bash
trk openssl get-args
trk openssl set-args <args>
```

List all encrypted files:

```bash
trk list encrypted
```

Re-encrypt all files (e.g., after changing the passphrase):

```bash
trk reencrypt
```

Verify that files are actually encrypted in Git:

```bash
trk rev-list --objects -g --no-walk --all
trk cat-file -p <hash>
```

## Key Features (v0.1.0)

**Encryption**:
- Pattern-based encryption marking
- Automatic encryption/decryption via Git filters
- OpenSSL AES-256-CBC with PBKDF2
- Custom merge driver for encrypted file conflicts

**Passphrase Management**:
- Generate secure passphrases: `trk passphrase generate`
- Import from file: `trk passphrase import`
- Retrieve current passphrase: `trk passphrase get`

**Configuration**:
- Export/import repository config
- Customizable OpenSSL arguments
- Optional file permission tracking

**Git Integration**:
- Full Git command passthrough
- Support for global repositories via `--worktree`
- Compatible with Git versions pre and post 2.46

## Use Cases

Trk excels at:

- **Dotfile management**: Track `.bashrc`, `.vimrc`, `.gitconfig`, etc. across machines
- **Secure configuration**: Encrypt SSH configs, API keys, and other sensitive files
- **Team dotfiles**: Share common configurations while encrypting personal credentials
- **Multi-machine sync**: Keep your development environment consistent across devices

## How It Works

Under the hood, trk is a Bash script that:

1. Wraps Git commands and passes them through transparently
2. Manages a separate Git directory (when using `--worktree`)
3. Configures Git filters to encrypt/decrypt files on the fly
4. Stores encryption keys and configuration in `.git/config`

The encryption happens seamlessly - when you `git add` a marked file, it's encrypted before being stored. When you check it out, it's automatically decrypted.

## Getting Started

For a typical dotfiles setup:

```bash
# Initialize with home directory as worktree
trk init --worktree $HOME

# Mark sensitive files for encryption
trk mark ~/.ssh/config
trk mark ~/.aws/credentials

# Add and commit files normally
trk add ~/.bashrc ~/.vimrc
trk commit -m "Initial dotfiles"

# Push to remote
trk remote add origin <your-repo-url>
trk push -u origin main
```

On a new machine:

```bash
# Clone your dotfiles
trk clone --worktree $HOME <your-repo-url>

# Enter your passphrase when prompted
# Your files are now tracked and encrypted files are automatically decrypted
```

## Conclusion

Trk provides a elegant solution for managing dotfiles with encryption. Its lightweight design (a single Bash script), transparent encryption, and global repository mode make it a powerful tool for anyone who wants to version control their configuration files securely.

Check out the project on [GitHub](https://github.com/TheoBrigitte/trk) and give it a try!
