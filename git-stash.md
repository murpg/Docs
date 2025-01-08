# Git Stash: A Comprehensive Guide

## What is Git Stash?
Git stash is a powerful feature that temporarily shelves (or stashes) changes you've made to your working copy so you can switch to another task, and then come back and re-apply them later. This is particularly useful when you need to quickly switch contexts or branches but aren't ready to commit your current changes.

## Basic Stash Commands

### Stashing Changes
```bash
git stash
```
This command saves your uncommitted changes (both staged and unstaged) and reverts your working directory to match the HEAD commit.

### Stashing with a Message
```bash
git stash save "your message here"
```
Adding a descriptive message makes it easier to identify stashes later.

### Viewing Stashes
```bash
git stash list
```
Shows all stashes in your stack. Each stash is identified by stash@{n}, where n is the index in the stack.

### Applying Stashes

1. Pop the latest stash (applies and removes it):
```bash
git stash pop
```

2. Apply the latest stash (keeps it in the stash list):
```bash
git stash apply
```

3. Apply a specific stash:
```bash
git stash apply stash@{n}
```

### Managing Stashes

1. Remove a specific stash:
```bash
git stash drop stash@{n}
```

2. Remove all stashes:
```bash
git stash clear
```

3. Viewing Stash Contents:
```bash
# Show files changed in latest stash
git stash show

# Show actual content changes in latest stash
git stash show -p

# Show files in a specific stash
git stash show stash@{n}

# Show content changes in a specific stash
git stash show -p stash@{n}
```

The `-p` flag (which stands for "patch") shows the actual line-by-line changes in the stash, similar to `git diff`.

## Advanced Usage

### Stashing Untracked Files
By default, `git stash` only stores tracked files. To include untracked files:
```bash
git stash -u
```

### Stashing Specific Files
To stash only specific files:
```bash
git stash push path/to/file1 path/to/file2
```

### Creating a Branch from a Stash
To create a new branch and apply stashed changes to it:
```bash
git stash branch new-branch-name stash@{n}
```

## Best Practices

1. **Always add a descriptive message** to your stashes to make them easily identifiable later.

2. **Clean up old stashes** regularly to avoid cluttering your repository.

3. **Be careful with pop vs. apply**
   - Use `pop` when you're sure you won't need the stash again
   - Use `apply` when you might need to apply the same stash to multiple branches

4. **Check your working directory** before stashing to ensure you're stashing the right changes.

## Common Scenarios

### Scenario 1: Quick Branch Switch
When you need to quickly switch branches but have uncommitted changes:
```bash
git stash
git checkout other-branch
# do your work
git checkout original-branch
git stash pop
```

### Scenario 2: Selective Stashing
When you want to stash only some of your changes:
```bash
git stash push -p

The command `git stash push -p` (or `git stash -p`) enters an interactive mode that lets you choose specific parts (chunks) of your changes to stash, rather than stashing all changes at once.

Here's what happens when you run it:

1. Git will show you each changed "chunk" of code one at a time
2. For each chunk, you'll see a prompt asking what you want to do, with these options:
   - `y` - yes, stash this chunk
   - `n` - no, don't stash this chunk
   - `q` - quit, don't stash this or any remaining chunks
   - `a` - stash this chunk and all remaining ones
   - `d` - don't stash this chunk or any remaining ones
   - `s` - split this chunk into smaller chunks
   - `e` - manually edit this chunk

For example, if you changed three different functions in a file, but only want to stash two of them, you can use `git stash -p` to choose exactly which changes to stash and which to keep in your working directory.

This is particularly useful when you've made multiple unrelated changes and want to stash some while keeping others to continue working on them.
```

### Scenario 3: Recovering Lost Changes
If you accidentally lost stashed changes, you can usually find them using:
```bash
git fsck --no-reflog | grep dangling
```

## Troubleshooting

### Merge Conflicts
When applying a stash results in conflicts:
1. Resolve the conflicts manually
2. Mark files as resolved using `git add`
3. Complete the stash application

### Lost Stashes
If you accidentally dropped a stash, you can usually recover it within the grace period:
```bash
git fsck --no-reflog | grep dangling | grep commit
```

## Tips for Teams

1. Communicate when using stashes in shared branches
2. Don't keep stashes for too long in a collaborative environment
3. Consider using feature branches instead of stashes for longer-term work

Remember: Stashes are meant to be temporary. For longer-term storage of changes, consider creating a branch instead.