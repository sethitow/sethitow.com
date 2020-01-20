---
title: "Git's Commad Line Interface"
date: 2019-12-24T23:02:36-08:00
draft: true
---

Git has a reputation for having an obtuse commnad line interface, however it recendly donned on me how disjoint it can be. For example, creating or deleting an object. 

## Ways to Create Things
- git remote add <origin name>
- git commit -m <message>
- git tag <tag name>
- Branches
	- git branch <branch name>
	- git checkout -b <branch name>
	- git switch -c <branch name>

## Ways to Delete Things
- git remote remove <origin name>
- git reset HEAD~1
- git tag -d <tag name>
- Branches
	- git branch -D <branch name>
	- git push <remote name>  :<branch name>