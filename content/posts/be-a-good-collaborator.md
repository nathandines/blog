---
title: "Be a Good Collaborator"
date: 2019-06-26T07:55:14+10:00
description: "Always strive to develop content in such a way as you would like to consume it"
categories: []
tags: []
---

Being a professional developer is about more than just smashing out code at a
keyboard from 9 to 5; it's about ensuring that the quality of the work is as
high as possible, which means that you spend time on things beyond just writing
code. These include documentation, commit history, agile card content, and
cross-references(!!!).

When developing on a project, be it your own personal project, or a project
which you're working on as part of a team, documentation is one of the most
important parts of it. The code itself will tell you what the application is
meant to do, and how it does it, but documentation gives you context (the why),
and tells you how it fits into the grander scheme of things. I'm not personally
a fan of overbaked documentation, so I think it's worth trying to consider "what
questions would I have about this project if I were new to it?" when developing
your documentation. This will help ensure that new collaborators on the project
will be able to get up to speed on the project faster, and this will also act as
a future reference guide for anyone else (including yourself).

One practice which is far too commonly discarded is writing good commit
messages, and keeping your commit history clean. I'm of the school of thought
that each commit should be a complete unit, adding one (and only one) feature or
bugfix all in itself. Your code should be able to compile cleanly from each
commit; This means that before you submit a pull request, you should rebase your
code if you've got a lot of cruft in your commit history. Rebasing can allow you
to squash related commits together to comprise a full feature. This will result
in a cleaner commit history on the project, where any commit could theoretically
be compiled and deployed. It also allows the commit history to actually tell a
story and give further context.

Rebasing also provides you with the opportunity to write meaningful commit
messages, which can give further context to changes, and cross-reference other
sources of information around the changes being made. I don't need to go into
detail on how to write good commit messages, because it has already been
discussed many times in detail. I find a good reference for writing good commit
messages is [this
page](https://github.com/erlang/otp/wiki/writing-good-commit-messages) which
also has some nice references of its own at the bottom.

Just like commit messages, if you're working using a card based system such as
Kanban or Scrum, the kind thing to do is to add any context you gain around the
card which you're working on to the card itself. I'm not talking about war and
peace, I'm just saying a couple of dot-points which give the next person some
additional information around where it's up to, and URL-based references to
other information such as Pull Requests, Slack threads, wiki pages, etc.
(obviously the cross-references are more difficult on a physical system, but be
pragmatic about it)

Speaking of Pull Requests, Slack, and all things discussion, try to capture what
you can in the relevent platform. If you're performing a code review, keep the
discussion in the Pull Request, this ensures that all collaborators who look at
the PR in the future can understand context around why design decisions were
made, and what discussions were had around changes and concessions in the code,
and again **include cross-references**. If there are other related commits, PRs
or Issues around this PR, make sure you link to those so that others can see the
history of the change.

Being a good collaborator isn't just limited to the workplace; I highly
recommend that if you utilise an open-source tool or library and find the need
to develop a feature or bugfix for it, please contribute it back to the
community, as others would surely benefit from your contribution. If you're a
team lead who is reading this, please try to provide time to your team to make
these contributions during work hours; don't make your team members to sacrifice
their personal time for open-source.

If you look at my development history, you'll find that I haven't always adhered
to these practices, but that's what development is all about: the continual
process of learning and improving. I've learnt these things by working with
other developers, both professionally and on open-source projects.

All of these practices are hallmarks of a professional developer. When the
aforementioned development practices all add up, they provide a much nicer
developer experience, and shortcut your peers to gaining context and getting
moving much faster; not to mention they can also be useful when looking back at
your own work.
