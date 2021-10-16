# Forest

Forest is a tool intended to replace `git log --graph` with something prettier.
I made this repo because it was maddening to me that there are all these forks and
copies of (git-)forest(a) and no clarity regarding their differences.

In order to [integrate git-forest into vim-flog](https://github.com/TamaMcGlinn/vim-flog-forest), I wanted to first be sure that I have
all features and bugfixes merged into one repo.

# Forks / Copies

Forest was originally created in 2008 Jan Engelhardt. His repository is still present [here](http://inai.de/projects/hxtools/),
but I have pushed [this github mirror](https://github.com/TamaMcGlinn/hxtools) so that I can share http links to commits.

There are four forks that I am aware of, and vim-flog-forest is an attempt to reconcile all their improvements, while staying
compatible with vim-flog.

- The original 2008 HXTools, with a [github mirror here](https://github.com/TamaMcGlinn/hxtools).
- John Wiegley's 2013 copy in [git-scripts](https://github.com/jwiegley/git-scripts/blob/master/git-forest)
- Takaaki Kasai's 2017 copy [git-foresta](https://github.com/takaaki-kasai/git-foresta)
- Roger Bongers' 2021 fork of [git-scripts](https://github.com/rbong/git-scripts/blob/master/git-forest)

## Key differences

### Fix hxtools git-forest

The original hxtools git-forest needs this modification to even run on modern perl versions:

```
@@ -12,7 +12,7 @@
 use Getopt::Long;
 use Git;
 use strict;
-use encoding "utf8";
+use utf8;
 my $Repo          = Git->repository($ENV{"GIT_DIR"} || ".");
 my $Pretty_fmt    = "format:%s";
 my $Reverse_order = 0;
```

### Coloured branches with git-foresta

Git-foresta chooses a colour for each branch, and sticks with it, making it easier to see
which commits belong to each.

### But, git-foresta doesn't support vim-flog

As mentioned [here](https://github.com/rbong/vim-flog/issues/49#issuecomment-940162907),
git-foresta does not accept a custom format, and hence vim-flog won't work.

## Benchmark

Using [hyperfine](https://github.com/sharkdp/hyperfine), I found the original hxtools
git-forest is 10% faster than the git-scripts' version, and 15% faster than git-foresta.

```
Benchmark #1: ./git-foresta
  Time (mean ± σ):      73.2 ms ±   1.0 ms    [User: 60.0 ms, System: 16.1 ms]
  Range (min … max):    71.6 ms …  75.2 ms    40 runs

Benchmark #2: ../hxtools/sdevel/git-forest
  Time (mean ± σ):      63.4 ms ±   1.0 ms    [User: 52.5 ms, System: 13.1 ms]
  Range (min … max):    61.6 ms …  65.6 ms    46 runs

Benchmark #3: ../git-scripts/git-forest
  Time (mean ± σ):      70.0 ms ±   0.8 ms    [User: 61.2 ms, System: 10.9 ms]
  Range (min … max):    68.2 ms …  71.6 ms    42 runs

Summary
  '../hxtools/sdevel/git-forest' ran
    1.10 ± 0.02 times faster than '../git-scripts/git-forest'
    1.15 ± 0.02 times faster than './git-foresta'
```

## Grafting repo's

In order to be able to reconcile these copies, I set out to find versions of each that correspond to each other, so that
the version in one repo can be rebased onto the other. I found:

- John Wiegley's first commit (re-)adding git-forest, [1d2a0e36cc](https://github.com/jwiegley/git-scripts/commit/1d2a0e36ccc9ce1744a5831ef84c5ed5a7dc9b42), is identical to Jan Engelhardt's [2ef51708fb0](https://github.com/TamaMcGlinn/hxtools/commit/2ef51708fb0b9b95c7ea62b124b98222f6662105) (then named bin/git-forest, later renamed sdevel/git-forest). There seem to be fixes in both repositories that did not make it into the other.
- Unfortunately Takaaki Kasai has made so many changes in the very first commit, that it is difficult to pinpoint where he got the script from. But it seems from line 224 of his first commit that he has included the latest of Jan Engelhardt's fixes, b9b2bf1543. Also, I can't find any place to even apply John Wiegley's changes, as all those lines seem to be missing in git-foresta from the beginning.

First, each repository is converted to only contain git-forest. Then, they can be merged.

### extract_hxtools

For hxtools, we use `git filter-branch --tree-filter '~/code/forest_forks/extract_hxtools' --prune-empty -f HEAD`.

The following `extract_hxtools` script deletes everything but git-forest, and ensures it is called git-forest and in the root of the repo:

```
#!/usr/bin/env bash

find . -type d | grep -v '^\./\.git\|^\.$\|^\./sdevel$\|^\./bin$' | xargs rm -rf
find . -type f | grep -v '^\./\.git\|git-forest$' | xargs rm -f
rm -f .gitignore
rm -f NEWS.rst

if [[ -f "bin/git-forest" ]]; then
  mv bin/git-forest .
fi
if [[ -f "sdevel/git-forest" ]]; then
  mv sdevel/git-forest .
fi

rm -rf sdevel
rm -rf bin
```

The resulting filtered hxtools history of git-forest is available as [hxtools_git_forest](https://github.com/TamaMcGlinn/git-forest/tree/hxtools_git_forest).

### extract_git-scripts

Similarly for git-scripts, there is `git filter-branch --tree-filter '~/code/forest_forks/extract_git-scripts' --prune-empty -f HEAD`.

```
#!/usr/bin/env bash

find . -type d | grep -v '^\./\.git\|^\.$' | xargs rm -rf
find . -type f | grep -v '^\./\.git\|git-forest$' | xargs rm -f
find . -type l | xargs rm
rm -f .gitignore
```

Because the script was removed and then re-added, this leaves some noise commits near the root 
of the repo, which are removed by magic. Finally, the unnecessary merges are removed with a rebase
without `--preserve-merges` onto the aforementioned corresponding commit in hxtools.

The resulting filtered git-scripts history of git-forest is available as [git_scripts_forest](https://github.com/TamaMcGlinn/git-forest/tree/git_scripts_forest).

### extract_foresta

And for git-foresta, `git filter-branch --tree-filter '~/code/forest_forks/extract_foresta' --prune-empty -f HEAD`

```
#!/usr/bin/env bash

find . -type d | grep -v '^\./\.git\|^\.$\|^\./script$' | xargs rm -rf
find . -type f | grep -v '^\./\.git\|git-foresta$' | xargs rm -f
rm -f .gitignore

if [[ -f "script/git-foresta" ]]; then
  mv script/git-foresta git-forest
fi
if [[ -f "git-foresta" ]]; then
  mv git-foresta git-forest
fi

rm -rf script/
```

The resulting filtered git-foresta history of git-forest is available as [git_foresta](https://github.com/TamaMcGlinn/git-forest/tree/git_foresta).

