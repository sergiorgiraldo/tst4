# git-flow-next — Hands-on Learning Lab

Target: your playground repo on GitHub, already cloned locally. Everything below runs against it directly.

Each phase is: **run → inspect → answer the checkpoint.** The checkpoints are the point; the commands are just how you get the evidence.

---

## 0. Aliases

| Written here | Runs |
|---|---|
| `g` | `git` |
| `g ft …` | `git flow feature …` |
| `g rel …` | `git flow release …` |
| `git flow …` | spelled out — `init`, `config`, `overview`, `hotfix`, `support`, shorthands |

### Settle what `next` is first

Your alias is `next = flow next`. I couldn't confirm a `next` subcommand in the CLI reference — the documented set is `init`, `config`, `overview`, `version`, `completion`, the per-type commands (`start finish publish list update delete rename checkout track`), and the shorthands (`finish update rebase rename publish delete`). No `next`. But git-flow.sh's workflow pages do print `git flow next update` in prose.

```bash
git flow next --help
git flow next update --help
```

- **Works** → `g next` is your generic entry point; use it wherever this doc writes `git flow`.
- **Errors** → it was marketing copy. Add real aliases:

```bash
g config --global alias.fl flow
g config --global alias.hf 'flow hotfix'
g config --global alias.sup 'flow support'
```

Plus the one you'll lean on constantly:

```bash
g config --global alias.lg "log --graph --oneline --decorate --all"
```

---

## 1. Setup

```bash
cd ~/path/to/playground
g remote -v
g branch -a
g symbolic-ref --short refs/remotes/origin/HEAD    # main or master?
```

Commit helper — use plain `git` inside, shell aliases don't expand in functions:

```bash
mk() { echo "$1" >> notes.md; git add -A; git commit -qm "$1"; git --no-pager log -1 --oneline; }
```

One practical note: if `main` has branch protection on, release and hotfix finishes will be rejected when they try to merge. Turn it off for the duration:

```bash
gh api -X DELETE repos/{owner}/{repo}/branches/main/protection 2>/dev/null || true
```

Since it's on real GitHub, you get a bonus the sandbox wouldn't give you — `publish`, `track`, remote branch deletion, and tags are all observable in the web UI as you go. Keep a browser tab open on the branches page.

---

## 2. `init` — what a workflow actually *is*

git-flow-next hardcodes no model. A workflow is config: **base branches** (long-lived) + **topic branch types** (short-lived), each with a parent and merge strategies.

```bash
git flow init --preset=classic --defaults
# if your default branch is master:
# git flow init --preset=classic --defaults --main=master
```

Three views of the same truth:

```bash
git flow config list                   # branch hierarchy
git flow overview                      # + live state: ahead/behind, health
git flow overview --verbose
g config --get-regexp '^gitflow\.'     # raw storage
sed -n '/\[gitflow/,$p' .git/config
```

Expect roughly:

```
[gitflow "branch.main"]      type = base   upstreamstrategy = none
[gitflow "branch.develop"]   type = base   parent = main   autoupdate = true
[gitflow "branch.feature"]   type = topic  parent = develop  prefix = feature/
[gitflow "branch.release"]   type = topic  parent = main  startpoint = develop  tag = true
```

`init` created `develop` locally if it didn't exist. Push it so the remote matches:

```bash
g push -u origin develop
```

Worth reading now, used later:

```bash
git flow init --help
# --preset=classic|github|gitlab  --custom  --defaults  --force  --no-create-branches
# --main= --develop= --feature= --release= --hotfix= --tag=
# scope: --local (default) | --global | --system | --file=path
```

### Checkpoint
1. Difference between a **base** branch and a **topic** branch type?
2. Why does `release` have both a `parent` (`main`) and a `startpoint` (`develop`)?
3. What does `autoupdate = true` on `develop` mean, and when does it fire?

---

## 3. Feature lifecycle

### 3.1 start / list / checkout

```bash
g ft start login
g branch --show-current            # feature/login
mk "feat: login form"
mk "feat: password validation"

g ft list
g ft list "log*"
g ft start search
g ft checkout log                  # partial-name matching
g branch --show-current
```

Start from an arbitrary point:

```bash
g ft start experiment main         # third arg = base commit/tag/branch
git flow config list               # note: still parented to develop for finish
g ft delete experiment --force
```

### 3.2 publish / track

```bash
g ft checkout login
g ft publish
g branch -vv
g ls-remote --heads origin
```

Now check GitHub — the branch is there. Simulate a teammate with a second clone of the same repo:

```bash
cd .. && git clone <same-github-url> playground-teammate && cd playground-teammate
git flow init --preset=classic --defaults
g ft track login
g branch -vv
cd ../playground
```

Push options, if you ever use GitLab/Gerrit:

```bash
g ft publish login -o ci.skip
g ft publish login -o merge_request.create -o merge_request.target=main
g ft publish login --no-push-option
```

### 3.3 update

Put something new on `develop` first:

```bash
g ft checkout search
mk "feat: search box"
g ft finish search
g lg
```

Then pull it down:

```bash
g ft checkout login
git flow update                    # shorthand, current branch
g ft update login --rebase         # force rebase over configured strategy
git flow rebase                    # alias for: update --rebase
```

### 3.4 rename / delete / finish

```bash
g ft rename login user-login
g ft list

g ft finish user-login
g lg
g branch -a                        # local gone
g ls-remote --heads origin         # remote gone too — confirm on GitHub
```

Variants worth trying on later features:

```bash
g ft finish X --keep --keeplocal --keepremote
g ft finish X --no-fetch
g ft finish X --force              # bypasses the remote-sync check
git flow finish                    # shorthand, current branch
```

### Checkpoint
1. Two merge directions — which config key governs each? (`upstreamstrategy` vs `downstreamstrategy`)
2. Why does `finish` fetch by default, and what does it refuse if your branch is *behind* its remote?
3. `git flow update` vs `git flow rebase`?

---

## 4. Adjusting commits — three distinct layers

Knowing which layer you're operating at is most of the skill here.

### Layer A — plain git, before finishing

A topic branch is a normal branch. Reshape freely:

```bash
g ft start cleanup
mk "wip"; mk "wip 2"; mk "wip 3"

g commit --amend -m "feat: cleanup, take one"       # reword the tip
g rebase -i develop                                  # squash/reorder/reword
g reset --soft develop && g commit -m "feat: cleanup"  # collapse to one
g lg
```

Deliberately hit the wall — rewrite *after* publishing:

```bash
g ft publish
mk "feat: extra"
g commit --amend -m "feat: extra (reworded)"
g ft finish cleanup                # expect a diverged/sync complaint
g push --force-with-lease          # the correct fix
# g ft finish cleanup --force      # the knowing bypass
```

### Layer B — finish-time flags

```bash
g ft start squashme && mk "wip a" && mk "wip b" && mk "wip c"
g ft finish squashme --squash --squash-message "feat: the whole thing, one commit"

g ft start linear && mk "step 1" && mk "step 2"
g ft finish linear --rebase        # replay onto develop, no merge commit

g ft start explicit && mk "one commit"
g ft finish explicit --no-ff --merge-message "feat: merge %b into %p"
g log -1 --format=%B develop
```

Placeholders: `%b` branch · `%B` full refname · `%p` parent · `%P` full parent refname · `%%` literal `%`.

Also available: `--preserve-merges` / `--no-preserve-merges`, `--ff` / `--no-ff`, `--no-verify`, `--update-message "chore: sync %b from %p"`.

### Layer C — config defaults

```bash
g config gitflow.branch.feature.upstreamstrategy squash
g config gitflow.branch.feature.downstreamstrategy rebase

g config gitflow.feature.finish.squash true
g config gitflow.feature.finish.mergemessage "feat: merge %b into %p"
g config gitflow.feature.start.fetch true

git flow config list
git flow config edit topic feature --upstream-strategy=merge   # same thing, via the command
```

Precedence, high → low: **CLI flags** → `gitflow.<type>.<command>.*` → `gitflow.branch.<type>.*`. Prove it rather than trusting it:

```bash
g config gitflow.feature.finish.squash true
g ft start prove && mk "x" && mk "y"
g ft finish prove --no-squash      # flag wins
g config --unset gitflow.feature.finish.squash
```

### Checkpoint
1. Which strategy loses individual commits; which loses the integration point?
2. Where does `--squash-message` sit in the precedence chain? (Trick — it's CLI-only, no config equivalent.)
3. Why is `--force-with-lease` better than `finish --force`?

---

## 5. Releases

Release branches are special twice over: `startpoint` ≠ `parent`, and `tag = true`.

```bash
git flow config list | grep -A5 release
g config gitflow.branch.release.tagprefix v
```

```bash
g ft start feat-for-1.0 && mk "feat: shipping thing" && g ft finish feat-for-1.0

g rel start 1.0.0
g branch --show-current            # release/1.0.0, branched off develop
mk "chore: bump version to 1.0.0"
g rel publish
g rel list
g rel finish 1.0.0 -m "Release 1.0.0"

g lg
g tag -l
g show v1.0.0 --stat | head -20
g push --tags
```

**The behaviour to notice:** `main` got the merge *and* `develop` was auto-updated from `main`, because `develop` has `autoupdate = true` and `main` is its parent. Verify:

```bash
g log --oneline develop..main      # empty — develop is not lagging
```

Tag control:

```bash
g rel start 1.1.0 && mk "chore: bump to 1.1.0"
g rel finish 1.1.0 --tagname v1.1.0-final -m "Release 1.1.0"
# --tag / --notag / --sign / --no-sign / --signingkey <id> / -m / --messagefile <f>

g config gitflow.release.finish.sign true
g config gitflow.release.finish.signingkey ABC123DEF
```

Maintenance on an open release:

```bash
g rel start 1.2.0
g ft start later && mk "feat: not for 1.2" && g ft finish later
g rel update 1.2.0                 # pull parent changes into the open release
g rel checkout 1.2.0 && mk "fix: release-only fix"
g rel finish 1.2.0 -m "Release 1.2.0"
```

### Checkpoint
1. Where does a release branch *start* vs where does it *merge to* — why different?
2. After finishing, which branch changed without you asking for it?
3. How do you finish a release with no tag?

---

## 6. Hotfixes and downstream propagation

This is the phase that justifies the tool over git-flow-avh: the classic failure is a hotfix reaching production but never reaching develop.

```bash
git flow hotfix start 1.2.1 v1.2.0     # start from the released tag
mk "fix: critical null pointer"
git flow hotfix finish 1.2.1 -m "Hotfix 1.2.1"

g lg
g tag -l
g log --oneline develop..main      # empty == the fix reached develop
g log --oneline develop -3
```

Harder case — hotfix while a release is open:

```bash
g rel start 1.3.0
git flow hotfix start 1.2.2 v1.2.1
mk "fix: another urgent one"
git flow hotfix finish 1.2.2 -m "Hotfix 1.2.2"

g rel update 1.3.0                 # pull the hotfix into the open release
g log --oneline release/1.3.0 -5
g rel finish 1.3.0 -m "Release 1.3.0"
```

Break it on purpose:

```bash
g config gitflow.branch.develop.autoupdate false
# repeat a hotfix
g log --oneline develop..main      # now non-empty: develop lags, avh-style
g config gitflow.branch.develop.autoupdate true
```

### Checkpoint
1. What does `autoupdate` on a *base* branch do that `update` on a *topic* branch does not?
2. Which strategy governs the parent→child update — `upstreamstrategy` or `downstreamstrategy`?

---

## 7. Conflicts: `--continue` and `--abort`

Manufacture a real one:

```bash
g checkout develop
echo "line: original" > conflict.txt
g add -A && g commit -m "chore: add conflict.txt"

g ft start left
echo "line: LEFT" > conflict.txt && g add -A && g commit -m "feat: left version"

g ft start right develop
echo "line: RIGHT" > conflict.txt && g add -A && g commit -m "feat: right version"

g ft finish left                   # clean
g ft finish right                  # conflict
```

Practice both exits:

```bash
g status
git flow overview                  # git-flow reports the operation state
cat conflict.txt

# path A: bail out
g ft finish right --abort
g status && g branch --show-current

# path B: resolve and continue
g ft finish right
echo "line: MERGED" > conflict.txt
g add conflict.txt
g ft finish right --continue
g lg
```

### Checkpoint
1. After `--abort`, where are you — mid-merge, on the topic branch, or on the parent?
2. Why is `finish` described as a state machine rather than a single merge?

---

## 8. Support branches

Long-lived maintenance lines for older versions.

```bash
git flow support start 1.2.x v1.2.2
g branch --show-current
mk "fix: backport for 1.2.x users"
git flow support list
git flow support publish
g config gitflow.support.finish.keep true
```

A topic type parented onto a support branch:

```bash
git flow config add topic backport support/1.2.x --prefix=backport/ --tag=false
git flow backport start fix-a
mk "fix: backported"
git flow backport finish fix-a
g lg
```

### Checkpoint
Why does a support branch have no meaningful "finish"?

---

## 9. Custom workflows

Two ways to run these. Re-initialising in place is fine on a playground and shows you what `--force` does to existing config; separate clones keep each experiment clean. Do the first one in place, then decide.

```bash
git flow init --preset=github --defaults --force
git flow config list
g ft start quick && mk "feat: straight to main" && g ft finish quick
g lg
```

```bash
git flow init --preset=gitlab --defaults --force
git flow config list               # production ← staging ← main
git flow overview
g ft start envtest && mk "feat: env test" && g ft finish envtest
g log --oneline main -2; g log --oneline staging -2; g log --oneline production -2
```

Build one from scratch:

```bash
git flow init --custom --force

git flow config add base production
git flow config add base staging production --auto-update=true
git flow config add base develop staging --auto-update=true

git flow config add topic feature develop --prefix=feat/
git flow config add topic bugfix  develop --prefix=bug/  --upstream-strategy=squash
git flow config add topic epic    develop --prefix=epic/ --downstream-strategy=rebase
git flow config add topic release staging --prefix=release/ --tag=true
git flow config add topic hotfix  production --prefix=hotfix/ --tag=true

git flow config list
git flow overview --format=json | head -40
```

Exercise the full CRUD — note that `rename base` touches the actual branch, not just config:

```bash
git flow config edit topic feature --upstream-strategy=rebase
git flow config edit base staging --auto-update=false
git flow config rename topic epic initiative
git flow config rename base develop integration     # renames config AND the git branch
git flow config delete topic initiative             # config only; branches survive
git flow config delete base staging                 # branch survives, management stops
git flow config list
```

Watch validation refuse bad input:

```bash
git flow config add base loopy loopy         # circular parent
git flow config add topic bad nonexistent    # missing parent
git flow config add topic weird develop --prefix='bad prefix/'
```

Then get back to classic:

```bash
git flow init --preset=classic --defaults --force
```

### Checkpoint
1. `config delete base X` vs actually deleting branch X?
2. Can a topic type parent onto another topic branch? (Section 8 answered this.)
3. Express: "features squash into develop, releases merge-commit into staging."

---

## 10. Hooks and filters

```bash
mkdir -p .githooks
g config gitflow.path.hooks .githooks     # preferred over core.hooksPath — git-flow only
```

Blocking pre-hook:

```bash
cat > .githooks/pre-flow-feature-start <<'HOOK'
#!/bin/sh
echo "pre-start: name=$1 origin=$2 branch=$3 base=$4"
echo "env: BRANCH=$BRANCH BRANCH_NAME=$BRANCH_NAME BRANCH_TYPE=$BRANCH_TYPE BASE_BRANCH=$BASE_BRANCH"
case "$BRANCH_NAME" in
  [A-Z][A-Z]*-*) exit 0 ;;
  *) echo "Feature names must look like ABC-123-description" >&2; exit 1 ;;
esac
HOOK
chmod +x .githooks/pre-flow-feature-start

g ft start badname            # blocked
g ft start JIRA-42-login      # allowed
```

Post-hook:

```bash
cat > .githooks/post-flow-feature-finish <<'HOOK'
#!/bin/sh
echo "post-finish: $BRANCH_TYPE $BRANCH_NAME exited with $EXIT_CODE"
HOOK
chmod +x .githooks/post-flow-feature-finish

mk "feat: hooked" && g ft finish JIRA-42-login
```

Version filter — semver enforcement:

```bash
cat > .githooks/filter-flow-release-start-version <<'HOOK'
#!/bin/sh
echo "$1" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+$' || { echo "semver required" >&2; exit 1; }
echo "$1"
HOOK
chmod +x .githooks/filter-flow-release-start-version

g rel start banana            # rejected
g rel start 2.0.0             # accepted
```

Tag-message filter — auto changelog:

```bash
cat > .githooks/filter-flow-release-finish-tag-message <<'HOOK'
#!/bin/sh
LAST=$(git describe --tags --abbrev=0 2>/dev/null)
LOG=$([ -n "$LAST" ] && git log --oneline "$LAST..HEAD" || git log --oneline -10)
printf '%s\n\nChanges:\n%s\n' "$2" "$LOG"
HOOK
chmod +x .githooks/filter-flow-release-finish-tag-message

mk "chore: bump 2.0.0" && g rel finish 2.0.0 -m "Release 2.0.0"
g tag -n99 -l v2.0.0
```

Naming: hooks are `{pre,post}-flow-{type}-{action}` with action ∈ `start finish publish track delete update`; filters are `filter-flow-{type}-{action}-{target}`.

Filter semantics to internalise: missing → original value used; non-executable → silently skipped; non-zero exit → operation fails; empty output → original value used.

### Checkpoint
1. Why prefer `gitflow.path.hooks` over `core.hooksPath` for team-shared git-flow hooks?
2. A filter that prints nothing — does it clear the value or leave it alone?

---

## 11. Tooling and AVH compatibility

```bash
git flow overview --format=json
git flow overview --format=yaml --no-color
git flow --verbose feature list       # see the underlying git commands
git flow completion zsh > /tmp/_git-flow
```

Simulate a legacy repo to see runtime translation — it reads avh keys without rewriting them:

```bash
g config gitflow.branch.master main
g config gitflow.prefix.feature feature/
g config gitflow.prefix.versiontag v
git flow config list                  # translated at runtime
g config --get-regexp '^gitflow\.'    # old keys untouched
```

---

## 12. Reset the playground

To wipe git-flow config and start over:

```bash
g config --name-only --get-regexp '^gitflow\.' \
  | sed 's/\.[^.]*$//' | sort -u \
  | while read -r sec; do g config --remove-section "$sec"; done

g config --get-regexp '^gitflow\.'    # empty
```

To clear lab tags and branches:

```bash
g tag -l 'v*' | xargs -r g tag -d
g push origin --delete $(g tag -l 'v*') 2>/dev/null
g branch | grep -E 'feature/|release/|hotfix/|support/' | xargs -r g branch -D
```

---

## Condensed cheat sheet

```bash
# setup
git flow init --preset=classic --defaults
git flow config list
git flow overview

# feature
g ft start NAME [base]
g ft publish [NAME] [-o push-option]
g ft track NAME
g ft update [NAME] [--rebase]
g ft list [glob]
g ft checkout PREFIX
g ft rename OLD NEW
g ft delete NAME [--force] [--remote]
g ft finish [NAME] [--squash|--rebase|--no-ff] [--keep] [--force]

# release
g rel start 1.2.0
g rel publish 1.2.0
g rel update 1.2.0
g rel finish 1.2.0 -m "Release 1.2.0" [--tag|--notag|--sign|--tagname X]

# hotfix / support
git flow hotfix start 1.2.1 v1.2.0
git flow hotfix finish 1.2.1 -m "Hotfix 1.2.1"
git flow support start 1.2.x v1.2.0

# shorthands (current branch)
git flow finish | update | rebase | publish | delete | rename NEW

# conflicts
git flow finish --continue | --abort

# config
git flow config add|edit|rename|delete base|topic ...
g config gitflow.branch.<type>.{prefix,parent,startpoint,upstreamstrategy,downstreamstrategy,tag,tagprefix,autoupdate,forcedelete}
g config gitflow.<type>.<command>.<option>
```

---

## Pacing

| Session | Sections | Focus |
|---|---|---|
| 1 (~45m) | 0–3 | Aliases, config model, feature loop |
| 2 (~45m) | 4 | Commit shaping, the three layers |
| 3 (~45m) | 5–6 | Releases, tags, hotfix propagation |
| 4 (~30m) | 7–8 | Conflict recovery, support branches |
| 5 (~60m) | 9–10 | Custom workflows, hooks, filters |
| 6 (~20m) | 11–12 | Tooling, AVH translation, reset |
