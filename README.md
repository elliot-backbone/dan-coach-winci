# dan-coach-winci

Windows CI for Dan Coach. **This repository is public and holds nothing but workflow files.**

## Why a public repository exists for a private product

GitHub bills Actions minutes on private repositories and does not bill them on public ones.
Windows runners bill at twice the standard rate, so a Windows job on the private `dan-coach`
repository is the most expensive shape available. On 2026-08-27 one such run was refused
outright by GitHub with `the job was not started because recent account payments have failed
or your spending limit needs to be increased`, having executed zero steps.

The workaround is the one already proven on `bill-coach-winci`: keep the product, the code and
the data private, and keep only the *instructions for running them* here, where the minutes are
free.

## What is deliberately not here

- No database. Dan's record is 1.23 GB and never enters Git anywhere, in this repository or in
  the private one.
- No product code. The runtime is checked out at run time from the private repository at an
  exact pinned commit.
- No credentials. The read token is an Actions secret, which is not readable from a public
  repository by anyone who can see this text.

Anything that would make this repository worth reading for its contents rather than its
mechanism is a defect. If a file appears here that is not a workflow, it is in the wrong place.

## What runs

`dan-coach-interactive` gets a Windows machine, installs Claude Code, fetches the pinned runtime
and the promoted store from the private repository, registers Coach the way Dan's own machine
registers it, and then opens an SSH session so a person can sit in front of it and talk to it.

The session is limited to the GitHub account that started the run. The connection string appears
in this repository's public log and is useless to anyone else.

Everything fetched is destroyed with the runner.

## The one thing this cannot prove

That Coach answers well. The workflow proves it attaches, that the store opens, and that twelve
tools are present. Whether the answers are any good is a judgement a person makes in the session,
which is the entire reason the session exists.
