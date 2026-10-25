---
layout: post
title: "Your PR Template Is in Your Commit History"
date: 2026-10-27 09:00:00 +0800
categories: [engineering, mining-software-repositories]
tags: [git, pull-requests, commit-messages, keyword-heuristics, false-positives, data-cleaning, methodology]
excerpt: "One repo's fix commits lit up my keyword counts, but the authors never wrote those words: squash merges had pasted in the PR template."
lang: en
---

I was doing the kind of analysis that looks easy on a whiteboard. Take fix commits from a handful of open-source agent frameworks, scan the commit bodies for two families of keywords, and see whether one family shows up more often alongside the other. One family was about errors and failures. The other was about status: words like "ready", "reported", "surfaced". The question was whether fixes that mention errors also tend to talk about how the error was made visible.

The early numbers looked mildly interesting. Then I broke them down per repository and one repo stood out. Its fix commits were full of both keyword families. I had expected to find a good habit there. What I actually found was a pull request template.

## How a template gets into git log

GitHub lets a repository choose what the default squash-merge commit message should be. One option is to use the pull request body. That sounds reasonable until you remember what a lot of PR bodies look like: a template with a polite greeting, a checklist, an example of a well-formed title, and instructions like "mark your pull request as ready for review". If the author fills in a description inside that template and doesn't delete the boilerplate, all of it gets squashed into the commit message, permanently.

In my case the template contained the word "ready" in its instructions and an example commit title with the word "error" in it. So every commit that kept the template matched *both* keyword families, and none of those matches had anything to do with the change.

A few properties of this made it nastier than random noise:

- **It clusters.** The false positives weren't spread across the dataset. They sat in one repository, and inside my time window, mostly on a single day when a batch of old PRs was merged. Clustered noise doesn't just add variance. It distorts any comparison between repositories.
- **It makes bodies longer.** I had a separate observation that longer commit bodies matched status keywords more often. A template inflates length *and* keyword hits at the same time, so I had to check whether my "length effect" was really a "template effect". It turned out to be mostly a real length effect, not a template one, but I would never have known without checking.
- **It's not a one-off.** I first wrote it off as a single bad merge day. Then I found the template showing up again in other commits months apart, including a partial version with the greeting deleted but the checklist intact. The template is always there, and it leaks whenever someone doesn't delete it.

## Two kinds of wrong

While I was cleaning this up, I realized I had two different problems that I had been treating as one.

The first is **wrong source**: the text isn't the author's. It's boilerplate that the tooling copied in. You can detect this against an external reference, the template file itself, and you can write the rule down before looking at any results.

The second is **wrong sense**: the text is the author's, but the keyword means something else. "Improve error logging" matches "error" and "log" but has nothing to do with surfacing status. A code comment like `# After: no error` matches "error" while saying the opposite. Deciding these cases takes human judgment about meaning, and if you make that judgment after seeing which group a commit falls into, you're tuning the result.

Those two need different fixes. Wrong source needs a provenance filter. Wrong sense needs something syntax-aware, like an exclusion list for collocations and negations, and that list is only trustworthy if you write it before you look at the data.

## My first fix deleted real data

The obvious cleanup for template contamination is to find the template's opening sentence and throw away everything after it. I did exactly that, and for a couple of commits the result was clean: nothing left, so drop the commit from the analysis.

Then I went back and read the commits line by line, and it turned out the author's description lives *inside* the template, under a "Description" heading. Truncating at the anchor threw away the boilerplate and the only sentences the author actually wrote. In one commit those deleted lines were a perfectly legitimate description of an error-message fix. My "cleaner" was removing real signal.

What worked better was cleaning at the level of lines:

1. **Detect** with a repository-agnostic rule: any line of at least a modest length that appears verbatim in several commits of the same repository is probably boilerplate. It doesn't need to know what the template says, so it works across repos.
2. **Delete** by comparing each line against the actual template text, and keep everything that doesn't match.

The detector's thresholds were touchy. Loosen the repeat count and it starts eating a real author's line that happened to appear twice. Raise the minimum length and it misses short template lines like an example commit title. I ended up writing the exact thresholds down and marking them as chosen after I had seen the data, which is honest but also means they need to be checked again on a fresh dataset.

## Don't stack evidence of different strength

The part I keep coming back to has nothing to do with templates.

Before I understood the mechanism, I had written a summary that went roughly: "The signal falls apart under cleaning: drop the template commits and it shrinks, drop the logging collocations and it shrinks further." That was one sentence describing two steps, and it presented them as equally strong.

They weren't. Removing template text is correcting a provenance error against an external reference. It's a high-evidence step that you could have specified in advance. Removing "error logging" matches is a judgment about meaning made after seeing the numbers. That's a low-evidence step, and it's exactly where an analyst's preferences sneak in. Putting them in one chain lets the weak step borrow credibility from the strong one, and that made the result look more fragile than it was. Once I split them, the honest summary was different: the result holds up against the well-grounded cleanup and is only sensitive to the judgment-based one.

So now, when I report the robustness of a result, I show a column per cleanup step, labeled with how that step was justified, rather than a single "cleaned" number.

## A cheap way to read template history

One open item was what the template looked like at the time those commits were made. You don't need to download a whole repository's blobs for that. A blobless, no-checkout clone is enough to walk the history of a single path:

```bash
git clone --filter=blob:none --no-checkout https://github.com/<org>/<repo>.git
cd <repo>
git log --all --name-status -- \
  ':(icase,glob)**/pull_request_template*' \
  ':(icase,glob)**/pull_request_template/**'
```

The `icase` magic matters because people spell that file every way imaginable, and `glob` with `**` catches templates in `.github/`, `docs/`, or the root. The second pathspec is for repositories that keep several templates in a `PULL_REQUEST_TEMPLATE/` directory; the first one alone silently skips the files inside it. With the commit list in hand, `git show <sha>:<path>` fetches only the blobs you actually need.

## Takeaways

- If you mine commit messages, check whether the repo squash-merges with PR bodies. If it does, assume template text is in your data until you've shown otherwise.
- Clean by line against the real template. Don't truncate at an anchor; authors write inside templates.
- Keep "this text isn't the author's" separate from "this word doesn't mean what the regex thinks". They fail differently and deserve different levels of trust.
- If you tune a detector's thresholds after seeing the data, write them down, say so, and re-check them on a fresh sample.
- When you describe how a result survives cleanup, break the chain into its steps and grade each one. A weak step shouldn't hide behind a strong one.
