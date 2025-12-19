---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*)
description: Review pull request
disable-model-invocation: false
---

Provide a code review for the given pull request.

To do this, follow these steps precisely:

1. Use another Haiku agent to give you a list of file paths to (but not the contents of) any relevant CLAUDE.md files from the codebase: the root CLAUDE.md file (if one exists), as well as any CLAUDE.md files in the directories whose files the pull request modified
2. Use a Sonnet agent to view the pull request, and ask the agent to return a summary of the change
3. Then, launch 6 parallel Sonnet agents to independently code review the change. The agents should do the following, then return a list of issues and the reason each issue was flagged (eg. CLAUDE.md adherence, bug, historical git context, etc.):
   a. Agent #1: Audit the changes to make sure they compily with the CLAUDE.md. Note that CLAUDE.md is guidance for Claude as it writes code, so not all instructions will be applicable during code review. Ignore any check command or things can be checked by command, as they are guarded by CI.
   b. Agent #2: Read the file changes in the pull request, then do a throughful scan for logical bugs. Read a bit extra context beyond the changes for larger scope. Focus on real bugs, and avoid nitpicks. Ignore likely false positives or ask another Opus agent for double check.
   c. Agent #3: Read the file changes in the pull request, then do a throughful scan for performance issues. Take both suboptimal coding style and combinations. Ignore likely false positives.
   d. Agent #4: Read the file changes in the pull request, then scan for red flags that violate design philosophy like shallow module/function, information leakage, overexposure, non-obvious code etc. Only flag those violations you have a clear improve path and ignore nit-picking things.
   e. Agent #5: Read code comments in the modified files, and make sure the changes in the pull request comply with guidances, requirements or invariances in the comments.
   f. Agent #6: Read the file changes in the pull request, and find unnecessary or verbose comments, overlapping test coverage cases, code duplications. Flag only real duplication, not tiny style improvement.
4. Update todo list to include every issue found in step 3. Then for each issue, launch a parallel Sonnet agent that takes the PR, issue description, and list of CLAUDE.md files (from step 1), and returns a score to indicate the agent's level of confidence for whether the issue is real or false positive. To do that, the agent should score each issue on a scale from 0-100, indicating its level of confidence. For issues that were flagged due to CLAUDE.md instructions, the agent should double check that the CLAUDE.md actually calls out that issue specifically. The scale is (give this rubric to the agent verbatim):
   a. 0: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
   b. 25: Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
   c. 50: Moderately confident. The agent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
   d. 75: Highly confident. The agent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach in the PR is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant CLAUDE.md.
   e. 100: Absolutely certain. The agent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.
5. Filter out any issues with a score less than 70. If there are no issues that meet this criteria, do not proceed.
6. Finally, write a report with the result. When writing your report, keep in mind to:
   a. Keep your output brief
   b. Avoid emojis
   c. Link and cite relevant code, files, and URLs

Use this list when evaluating issues in Steps 3 and 4 (these are false positives, do NOT flag):

- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (eg. missing or incorrect imports, type errors, broken tests, formatting issues, pedantic style issues like newlines). No need to run these build steps yourself -- it is safe to assume that they will be run separately as part of CI.
- General code quality issues (eg. lack of test coverage, general security issues, poor documentation), unless explicitly required in CLAUDE.md
- Issues that are called out in CLAUDE.md, but explicitly silenced in the code (eg. due to a lint ignore comment)
- Changes in functionality that are likely intentional or are directly related to the broader change
- Real issues, but on lines that the user did not modify in their pull request

Notes:

- Use gh CLI to interact with GitHub (e.g., fetch pull requests). Do not use web fetch.
- Create a todo list before starting.
- You must cite and link each issue (e.g., if referring to a CLAUDE.md, include a link to it).
- For your final summary, follow the following format precisely (assuming for this example that you found 3 issues):

---

## Code review

Found 3 issues:

1. <brief description of bug> (CLAUDE.md says: "<exact quote from CLAUDE.md>")
<link to file and line with relative path + line range for context, eg. src/lib.rs:13-17>
<a brief explaination of why this is flagged>

2. <brief description of bug> (some/other/CLAUDE.md says: "<exact quote from CLAUDE.md>")
<link to file and line with relative path + line range for context>
<a brief explaination of why this is flagged>

3. <brief description of bug> (bug due to <file and code snippet>)
<link to file and line with relative path + line range for context>
<a brief explaination of why this is flagged>

---

- Or, if you found no issues:

---

## Auto code review

No issues found.

---

- When linking to code, follow the following format precisely, otherwise the Markdown preview won't render correctly: src/lib.rs:13-17
  - Requires relative path to the project root
  - Repo name must match the repo you're code reviewing
  - : sign after the file name
  - Line range format is [start]-[end]
