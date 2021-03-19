

# git  deep dive notes


git system contain three type file

blob

tree

commit


### step 1.  create working folder
```
~/Documents ➭ mkdir gitdeepdive
~/Documents ➭ cd gitdeepdive
~/Documents/gitdeepdive ➭ git init
Initialized empty Git repository in ~/Documents/gitdeepdive/.git/
~/Documents/gitdeepdive:master ✓ ➭


.git
├── HEAD
├── branches
├── config
├── description
├── hooks
│   ├── applypatch-msg.sample
│   ├── ......
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags
```

### step 2.  staging one file
one blob object created
```
gitdeepdive:master ✓ ➭ echo "version 1" > afile.txt

gitdeepdive:master ✗ ➭ git add afile.txt

├── objects
│   ├── 83
│   │   └── baae61804e65cc73a7201a7252750c76066a30


gitdeepdive:master ✗ ➭ git cat-file -p 83baae
version 1
gitdeepdive:master ✗ ➭ git cat-file -t 83baae
blob
```

### step 3.  commit the file
one tree object 5f670 created, list the blob object 83baae


one commit object bf1a4 created, list the tree object 5f670
```
gitdeepdive:master ✗ ➭ git commit -m "commit 1"

├── objects
│   ├── 5f
│   │   └── 6706d5e13f4e5e3e6fc459ecef6c7cb80d649d
│   ├── 83
│   │   └── baae61804e65cc73a7201a7252750c76066a30
│   ├── bf
│   │   └── 1a450530a9cb2587d9510f36dcc5cfadd9fda9


gitdeepdive:master ✓ ➭ git cat-file -p 5f670
100644 blob 83baae61804e65cc73a7201a7252750c76066a30    afile.txt
gitdeepdive:master ✓ ➭ git cat-file -t 5f670
tree
gitdeepdive:master ✓ ➭ git cat-file -p bf1a4
tree 5f6706d5e13f4e5e3e6fc459ecef6c7cb80d649d
author clairezhou  1514996248 +0800
committer clairezhou  1514996248 +0800

commit 1
{0:19}~/Documents/gitdeepdive:master ✓ ➭ git cat-file -t bf1a4
commit
```

### step 4.  change afile.txt
one new blob 6527a created

origin blob 83baae is unchanged

```
├── objects
│   ├── 65
│   │   └── 27a68f1f04cf9a0e95903df6d8fae0f865fc67


gitdeepdive:master ✗ ➭ git cat-file -p 6527a
version 1

version 2

gitdeepdive:master ✗ ➭ git cat-file -p 83baae
version 1

```

### step 5.  commit the change
one tree object c032a created, list the blob object 6527a


one commit object d2a74 created, list the tree object c032a and parent commit object bf1a45

```
├── objects
│   ├── c0
│   │   └── 32a96db7def0b1e55eddb230eaaca54754d448
│   ├── d2
│   │   └── a74f0b25798bf6e725f9dc35e4953ba5db749b


gitdeepdive:master ✓ ➭ git cat-file -p c032a
100644 blob 6527a68f1f04cf9a0e95903df6d8fae0f865fc67    afile.txt
gitdeepdive:master ✓ ➭ git cat-file -p d2a74
tree c032a96db7def0b1e55eddb230eaaca54754d448
parent bf1a450530a9cb2587d9510f36dcc5cfadd9fda9
author clairezhou  1514998492 +0800
committer clairezhou  1514998492 +0800

commit 2
```

### step 6.  staging another file
new blob fac58 created

```
├── objects
│   ├── fa
│   │   └── c580e980837fda4244a08ee543e0e0aa16f2e2

gitdeepdive:master ✗ ➭ git cat-file -p fac58
file 2
```

### step 7.  commit another file
one tree object b83aff created, list two blob object 6527a and fac580


one commit object d4006 created, list the tree object b83aff and parent commit object d2a74

```
├── objects
│   ├── b8
│   │   └── 3affa2384f74954b5cc0cb1c17b9a01bd8f082
│   ├── d4
│   │   └── 006c445927b218df899d1aef557f88a9f86add


gitdeepdive:master ✓ ➭ git cat-file -p b83aff
100644 blob 6527a68f1f04cf9a0e95903df6d8fae0f865fc67    afile.txt
100644 blob fac580e980837fda4244a08ee543e0e0aa16f2e2    bfile.txt
gitdeepdive:master ✓ ➭ git cat-file -p d4006
tree b83affa2384f74954b5cc0cb1c17b9a01bd8f082
parent d2a74f0b25798bf6e725f9dc35e4953ba5db749b
author clairezhou 1514999579 +0800
committer clairezhou 1514999579 +0800

file 2

```

### step 8.  create branch dev

refs/heads/branchname only record head commit in each branch

```
gitdeepdive:master ✓ ➭ git checkout -b dev

└── refs
    ├── heads
    │   ├── dev        //d4006c
    │   └── master     //d4006c
    └── tags
```

### step 9.  logs


```
gitdeepdive:dev ✓ ➭ cat .git/logs/HEAD
0000000000000000000000000000000000000000 bf1a450530a9cb2587d9510f36dcc5cfadd9fda9 clairezhou  1514996248 +0800    commit (initial): commit 1
bf1a450530a9cb2587d9510f36dcc5cfadd9fda9 d2a74f0b25798bf6e725f9dc35e4953ba5db749b clairezhou  1514998492 +0800    commit: commit 2
d2a74f0b25798bf6e725f9dc35e4953ba5db749b d4006c445927b218df899d1aef557f88a9f86add clairezhou  1514999579 +0800    commit: file 2
d4006c445927b218df899d1aef557f88a9f86add d4006c445927b218df899d1aef557f88a9f86add clairezhou  1515000268 +0800    checkout: moving from master to dev

```


```
gitdeepdive:dev ✓ ➭ cat .git/logs/refs/heads/dev
0000000000000000000000000000000000000000 d4006c445927b218df899d1aef557f88a9f86add clairezhou  1515000268 +0800    branch: Created from HEAD
gitdeepdive:dev ✓ ➭ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 bf1a450530a9cb2587d9510f36dcc5cfadd9fda9 clairezhou 1514996248 +0800    commit (initial): commit 1
bf1a450530a9cb2587d9510f36dcc5cfadd9fda9 d2a74f0b25798bf6e725f9dc35e4953ba5db749b clairezhou  1514998492 +0800    commit: commit 2
d2a74f0b25798bf6e725f9dc35e4953ba5db749b d4006c445927b218df899d1aef557f88a9f86add clairezhou  1514999579 +0800    commit: file 2

```


## notes
tree contain all file structure, once a file in a folder change, a new tree is need to create, thus his parent tree need to create new node too

when two branch merge, a new commit is created with two parent, a new tree is created with merged blob and subtree which from two branch, no blob been created

.git/index contain all file sha in current branch, include staged but uncommit files, we can use git ls-files  --stage to view index file content(The index is a binary file    https://stackoverflow.com/questions/4084921/what-does-the-git-index-contain-exactly)
