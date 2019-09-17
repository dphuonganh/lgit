# lgit


## Introduction

Version control is a powerful tool for collaborative teams. It might seem overly complicated for the purpose of just saving some code, and you might have already been traumatised by some wild git conflict on your own projects... every aspiring developer gets scared by git at some point! But imagine working in a team of a dozen developers on a project that has several features being developed in parallel: you need a smart process to ensure the tracking & integrity of the main project and a way to incorporate smoothly the new features.

Therefore, over the next two weeks, you will understand what version control is by looking under the hood and coding your own version of git. Understand how git works, and you will never waver in front of a conflict again!


## Your mission

Your task is to code a lightweight version of git. The goal is to understand how git works so that you can make sense of the magic that happens when you "git add / git commit / git push" - and save yourself of any git trouble you might encounter in the future as you work on large teams.

As the actual implementation of git is a bit complex, we'll work on a simplified version together, step by step. While simplified a little, this version still gives an accurate representation of how git works.

Your program will be called `lgit.py` and will implement versioning locally, but you don't have to handle remotes (so no pull, fetch or push).


## First: what is Git?

Git is a type of database: it records the state of the files in your working directory, which allows you to have a clear picture of what your directory contains at a given moment.

Git stores all its data in a hidden directory named `.git.` That directory is created when you type the command git init in a new directory, as tracking is initialised in that directory.

From there, any time you use a git command (`git add, git commit, git status...`), git will read and update the information stored in the .git directory. Destroy the .git directory, and you lose absolutely all tracking information!


```python
$ ls -al
total 0
drwxr-xr-x   2 laurie  wheel   64 Nov 10 17:46 .
drwxrwxrwt  25 root    wheel  800 Nov 10 17:46 ..

$ git init
Initialized empty Git repository in /private/tmp/test/.git/

$ ls -al
total 0
drwxr-xr-x   3 laurie  wheel   96 Nov 10 17:46 .
drwxrwxrwt  25 root    wheel  800 Nov 10 17:46 ..
drwxr-xr-x   9 laurie  wheel  288 Nov 10 17:46 .git

# the git init command has initialised all important files & directories in the .git directory
$ ls -l .git
total 24
-rw-r--r--   1 laurie  wheel   23 Nov 10 17:46 HEAD
-rw-r--r--   1 laurie  wheel  137 Nov 10 17:46 config
-rw-r--r--   1 laurie  wheel   73 Nov 10 17:46 description
drwxr-xr-x  13 laurie  wheel  416 Nov 10 17:46 hooks
drwxr-xr-x   3 laurie  wheel   96 Nov 10 17:46 info
drwxr-xr-x   4 laurie  wheel  128 Nov 10 17:46 objects
drwxr-xr-x   4 laurie  wheel  128 Nov 10 17:46 refs

```

## The key concepts

The main thing to understand is that, contrary to other version control systems, git records a full picture (a "snapshot") of your working directory every time you do a `git commit`. Every commit is a file listing that represents the exact state of your tracked files at one point.

What happens when you update a file? Git doesn't bother recording the difference between the new version and the old version, it actually just stores the new version of the file. This makes retrieving a version of a file extremely easy, as you just have to fetch it from the git database. Let's repeat it: git stores full files, and not just "updates" to a file. (This is actually not an absolute rule, but it's enough for our project today.)

Now the last thing to understand is what is the index. The index is a "temporary snapshot" that contains what will effectively be stored if you do a `git commit`. The index is a listing of all the files you have `git add` ed previously, and when you type in `git commit` what you effectively do is make a permanent copy of that snapshot. That index is also called the staging area, because it's the area where you make changes, add files, remove files (you "stage" your files... like a theater director!); and when you are perfectly content with what it looks like, you commit that snapshot permanently in your git history. Basically, a `git commit` says "I want to remember my directory in exactly that state, please archive a picture of my staging area, kthkbay."


## Your .lgit directory structure

For the purpose of simplification, we will use the following directory structure:

* a directory `objects` will store the files you lgit add
* a directory `commits` will store the commit objects: those are not the actual file listings but some information about the commit (author, date & commit message)
* a directory `snapshots` will store the actual file listings
* a file `index` will host the staging area & other information
* a file `config` will store the name of the author, initialised from the environment variable LOGNAME

The command `lgit init` will create the directory structure.

If a lgit command is typed in a directory which doesn't have (nor its parent directories) a `.lgit` directory, lgit will exit with a fatal error.

```python
# let's start with an empty directory
$ ls -la
total 0
drwxr-xr-x   3 laurie  staff   96 Nov 10 18:16 .
drwxr-xr-x  11 laurie  staff  352 Nov 10 18:16 ..

# oops, we haven't initialised the lgit directory yet!
$ ../lgit.py status
fatal: not a git repository (or any of the parent directories)

$ ../lgit.py init

$ ls -la
total 0
drwxr-xr-x   3 laurie  staff   96 Nov 10 18:16 .
drwxr-xr-x  11 laurie  staff  352 Nov 10 18:16 ..
drwxr-xr-x   9 laurie  staff  288 Nov 10 18:16 .lgit

$ ls -lR .lgit
total 8
drwxr-xr-x  2 laurie  staff  64 Nov 10 18:39 commits
-rw-r--r--  1 laurie  staff   7 Nov 10 18:39 config
-rw-r--r--  1 laurie  staff   0 Nov 10 18:39 index
drwxr-xr-x  2 laurie  staff  64 Nov 10 18:39 objects
drwxr-xr-x  2 laurie  staff  64 Nov 10 18:39 snapshots

.lgit/commits:

.lgit/objects:

.lgit/snapshots:

# the content of the config file has been initialised to the LOGNAME environment variable
$ env | grep LOGNAME
LOGNAME=laurie
$ cat .lgit/config
laurie
```

## Structure of the "objects" directory

When you `lgit add` a file, what you do is store a copy of the file content in the lgit database.

File contents will be stored in the lgit database with their SHA1 hash value. It means that you have to calculate the SHA1 value of the file content, and you will use the SHA1 value to retrieve the file content at any time.

Each file will be stocked in the following way:

* the first two characters of the SHA1 will be the directory name
* the last 38 characters will be the file name

Let's look at an example!

```python
# let's create some test file and add it
$ echo "test content" > test
$ ../lgit.py add test

# we can check that the object has been created in the lgit database
$ ls -lR .lgit/objects/
total 0
drwxr-xr-x  3 laurie  staff  96 Nov 10 18:46 4f

.lgit/objects//4f:
total 8
-rw-r--r--  1 laurie  staff  13 Nov 10 18:46 e2b8dd12cd9cd6a413ea960cd8c09c25f19527

# and yup, the object has our original file content
$ cat .lgit/objects/4f/e2b8dd12cd9cd6a413ea960cd8c09c25f19527
test content
```

## Structure of the "index" file

If you remember, the "index" file is our staging area: it's where we keep all current file information before we commit anything to the permanent "commits"/"snapshots" directories.

Each line in the index file will correspond to a tracked file and will contain 5 fields.

* 1: the timestamp of the file in the working directory
* 2: the SHA1 of the content in the working directory
* 3: the SHA1 of the file content after you `lgit add`'ed it
* 4: the SHA1 of the file content after you `lgit commit`'ed it
* 5: the file pathname
So basically the index has information about the content of a given file (as identified by its name) at different stages.

Let's go back to our example. I just added a file called test, let's look at the index. Since I just added the file, the content is the same in the staging as in the working directory. We have never done a commit so there's no hash in the fourth field.

```python
$ cat .lgit/index
20181110184140 4fe2b8dd12cd9cd6a413ea960cd8c09c25f19527 4fe2b8dd12cd9cd6a413ea960cd8c09c25f19527                                          test
```

## lgit status

Let's modify again the `test` file. We then run `lgit status` to update the index and... surprise, the file appears both as "to be committed" and "not staged for commit"! How is it possible?

Well, look at the index file. We had already once `lgit add`'ed the file, so lgit already has a version ready to be committed. You can check, the SHA1 in the middle didn't change.

But we also modified that file in the current directory, so the index has been updated with the new file timestamp and the SHA1 representing the new file content in the working directory.

And you can check that the first SHA1 **isn't** present in the objects directory. That's because it's just the state of the file in the current directory, but we have never `lgit add`'ed it!

```python
$ ../lgit.py status
On branch master

No commits yet

Changes to be committed:
  (use "./lgit.py reset HEAD ..." to unstage)

     modified: test

Changes not staged for commit:
  (use "./lgit.py add ..." to update what will be committed)
  (use "./lgit.py checkout -- ..." to discard changes in working directory)

     modified: test

# the hashes for the working directory and the staging area are different!
$ cat .lgit/index
20181110190018 48b67a26dc1b5897622a1e332bba3160c7b1f5bb 4fe2b8dd12cd9cd6a413ea960cd8c09c25f19527                                          test

# and we can check that the first hash doesn't correspond to a file in the database
$ ls -R .lgit/objects/
4f

.lgit/objects//4f:
e2b8dd12cd9cd6a413ea960cd8c09c25f19527
```

## lgit status: untracked files

`lgit status` not only updates the index but also reports on the files that are "untracked" by lgit. Those are the files that are present in the working directory, but have never been `lgit add`'ed.

```python
# let's init a fresh lgit repository
$ ../lgit.py init

# our directory has a couple of files that have never been added
$ ls -R
dir1	file1

./dir1:
nested_file

# we can check that lgit is currently not tracking those files
$ ../lgit.py ls-files

$ ../lgit.py status
On branch master

No commits yet

Untracked files:
  (use "./lgit.py add <file>..." to include in what will be committed)

	 file1
	 dir1/nested_file

nothing added to commit but untracked files present (use "./lgit.py add" to track)
```

## lgit commit, the commit object & the snapshot

Okay, let's actually commit the changes (that is, tell lgit to store the snapshot permanently).

The commit object and the snapshot are simplified here compared to what git actually does, but the principle is the same.

Both objects have the same name (here it's the timestamp with milliseconds, you can use another way to name your files if you want) and are in their respective folders.

Your commit object will contain the following information:

* author name
* time of the commit
* an empty line
* the commit message

The snapshot is a simplified version of the index with just the SHA1 of the staged content and the filename.

Let's walk through an example!

```python
$ ../lgit.py commit -m "my first commit"

# after the commit command, we check that the proper commit object and snapshot have been created
$ ls .lgit/commits
20181110190825.781983

$ ls .lgit/snapshots/
20181110190825.781983

$ cat .lgit/commits/20181110190825.781983
laurie
20181110190825
```

## lgit log

Now that you have worked hard, you might want to see all those commits you did! The `lgit log` command should display all commits, sorted in descending order, with the following information:

* your commit identifier (for lgit we will use the commit filename, the real git uses the SHA1 of the commit)
* a field with the author of the commit
* a field with a human-readable date
* and the commit message

```python
# after adding your first files, you do your first commit
$ ../lgit.py commit -m "1st commit"

# ... you add some more files...
$ ../lgit.py commit -m "2nd commit"

$ ../lgit.py log
commit 20181113171124.703280
Author: laurie
Date: Tue Nov 13 17:11:24 2018

	 2nd commit


commit 20181113171110.826441
Author: laurie
Date: Tue Nov 13 17:11:10 2018

	 1st commit
```

## Directions

Now hopefully after that slow walk-through, you have a good idea of how your lgit program works and interacts with the .lgit directory.

It's time for you to code your own!

Here are the commands that must be present in your project:

* lgit init: initialises version control in the current directory (until this has been done, any lgit command should return a fatal error)
* lgit add: stages changes, should work with files and recursively with directories
* lgit rm: removes a file from the working directory and the index
* lgit config --author: sets a user for authoring the commits
* lgit commit -m: creates a commit with the changes currently staged (if the config file is empty, this should not be possible!)
* lgit status: updates the index with the content of the working directory and displays the status of tracked/untracked files
* lgit ls-files: lists all the files currently tracked in the index, relative to the current directory
* lgit log: shows the commit history

You must respect the directory structure & files indicated in the subject.


## BONUS:

### Branches & merging

Branches are one of git's most powerful features. It allows developers to develop concurrent code, from one single point of code in time. This allows teams to work on different features separately.

Later on, when they are stable, those "branches" are merged into the main program.

For this bonus, you will implement the following commands:

* lgit branch: creates a new branch
* lgit checkout: replaces the working directory with the content of the head commit of the specified branch
* lgit merge: merges one branch onto another
* lgit stash: puts away in a "dirty working directory" the current changes, which allows to switch to a new branch without working files

To implement branches, you might have to modify a bit the structure of the .lgit directory. For example, it might be relevant to have a **HEAD** file pointing at the current branch. And it might be relevant to have a directory listing all existing branches. (For reference, git keeps track of those information in a directory called **refs/heads**. Look it up!)

==Note:== If you add some information to your commit objects (to include a reference to one or several parent commit objects maybe...?), be sure to respect the following structure:

* author name
* time of the commit
* 1 or several lines with additional information
* an empty line
* the commit message
