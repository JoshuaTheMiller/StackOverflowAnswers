There is now a file in this directory besides the README.

Another line.

# Hide and Seek

There should be another file in this directory. You remember it was a `.txt` file, but nothing else. Since we are using `git`, it should exist somewhere in the commit history, but where? Checking the commit history doesn't help, as there is not commit that said it was removing the file. The removal must have been a mistake...

Thankfully, `git` has some commands that can help us here: `git rev-list`. Looking through [the documentation](https://git-scm.com/docs/git-rev-list), we can some additional options that can help in finding the missing file.

## Attempt 1

```sh
# input
git rev-list HEAD -- .txt
# output
```

Unfortunately, nothing appears. Thinking about it, this is probably because the file we are trying to find was in a subdirectory. We know it was in the `undeleteProblem` subdirectory, so lets add it to the search pattern.

## Attempt 2

```sh
# input
git rev-list HEAD -- undeleteProblem/*.txt
# output
7e4bfe3a10b36de7805147de970c35aef5e8cc82
af24fe839b0349a43e20e5a506e48bb6f6676958
```

Awesome! We have some output, though it doesn't tell us much. By reading the documentation for `git rev-list`, we understand that commits are listed in reverse chronological order for any modifications to the object in question (the "modification" in this case being a deletion). Theoretically, that means that if we checked out the commit immediately prior to the first commit hash in this list, we would be able to make a copy of the file that was accidentally deleted. Let's try that now:

```sh
git checkout 7e4bfe3a10b36de7805147de970c35aef5e8cc82^
```

Looking at the files in our directory, we can now see the missing file is back!

![The missing file is now present in our directory](https://i.imgur.com/aV0nYHg.png)

Our work is not yet done, however, as the file does not exist in the latest commit of this repository. To solve this the quick and dirty way, perform the following steps:

1. Copy the file (`Right Click` + `Copy`, or however you prefer)
2. Switch back to the latest commit: `git checkout HEAD` (HEAD typically being the latest)
3. Paste the file where you want it
4. Stage the file: `git add undeleteProblem/somefile.txt`
5. Commit the staged changes: `git commit -m "Recover accidentally deleted file"`
6. Be happy that your file is back

## Attempt 3

Let us say you don't remember the file extension of the file you are trying to find. You just know it was committed at some point, but it is deleted now. While this may be a bit more work, you could start your search with the following:

```sh
git log --pretty=oneline --diff-filter=D
```

The command above will list all commits that contain *deletions of files*. At this point, you can go about a similar process as `Attempt 2` where you `git checkout` the commit prior to the one(s) shown by the command. Once you find the file, the rest of the process is the same as `Attempt 2`.

## Additional niceties

```sh
git rev-list --pretty=oneline --abbrev-commit --max-count=5 HEAD -- **.txt
```

`git rev-list` has many additional options (check the documentation for more). Above is a command that could be used for this excercise that adds some additional, arguably helpful, detail.

* `--pretty=oneline`= instead of just the raw commit hash, also output the commit message all in one line
* `--abbrev-commit`= shorten the commit hash while still making it unique
* `--max-count=5`= show only the last 5 commits that match the glob pattern
    * Instead of writing `--max-count=5`, one could also use `-n 5` or simply `-5` **for this command**