# gh-attach

Upload images and video to GitHub from the command line and get back a
`user-attachments` URL — the kind GitHub's web UI creates when you drag a file
into a PR description.

```console
$ gh-attach screenshot.png
![screenshot.png](https://github.com/user-attachments/assets/254e12e2-…)

$ gh-attach demo.mp4
https://github.com/user-attachments/assets/89ad44f0-…
```

## Why this exists

GitHub has no public API for attachment uploads, and `user-attachments` URLs are
the **only** ones its markdown renderer turns into an inline video player. A
video in a release asset or committed to the repo renders as a dead link.

So for embedding a demo video in a PR description, there is no supported
alternative. For images there are alternatives — release assets and
`raw.githubusercontent.com` both render fine — but this keeps images and video
on one path.

## What it does

One HTTP request, using the OAuth token your `gh` CLI already holds:

```
POST https://uploads.github.com/user-attachments/assets
     ?name=…&content_type=…&repository_id=…
Authorization: Bearer <gh auth token>
→ 201 {"url": "https://github.com/user-attachments/assets/<uuid>"}
```

It never reads browser cookies, never touches your Keychain, and contacts no
host other than `api.github.com` (via `gh`) and `uploads.github.com`.

That last point is the reason this exists as a 200-line shell script rather than
a dependency. The general-purpose tools in this space fall back to scraping your
GitHub **session cookie** out of your browser's encrypted cookie store, which is
what makes them work for arbitrary file types and read-only repos. A
`user_session` cookie is equivalent to your password — unscoped, no MFA
challenge. Restricting the scope to images and video on repos you can push to
means the OAuth token is sufficient, and none of that machinery is needed.

## Install

```bash
git clone https://github.com/joseustra/gh-attach
install -m 755 gh-attach/gh-attach ~/.local/bin/gh-attach
```

Any directory on your `PATH` works. Requires `gh` (authenticated), `curl`, and
bash — including the bash 3.2 that ships with macOS.

## Usage

```bash
# Infers the repository from the current git workspace
gh-attach screenshot.png

# Target a repository explicitly
gh-attach demo.mp4 --repo owner/repo

# Several files at once — one reference per line
gh-attach before.png after.png
```

Pipe it straight into a PR description or comment:

```bash
gh pr comment 123 --body "$(gh-attach screenshot.png)"
```

Images print as `![name](url)`. **Video prints as a bare URL** — paste it on its
own line and leave it unwrapped, because GitHub renders a bare
`user-attachments` URL as a player and a wrapped one as a link.

If one file in a batch fails, the rest still upload and the exit code is 1.

## Supported types

`png` `jpg` `jpeg` `gif` `webp` `svg` `mp4` `mov` `webm`

Anything else is rejected locally with a clear message rather than sent to be
refused. The endpoint accepts images and video only when authenticating with an
OAuth token.

## Credential handling

- The `gh` OAuth token is passed to curl through a **config file on stdin**
  (`-K -`), never as `-H` on the command line. Process arguments are readable
  via `/proc/<pid>/cmdline` on Linux and by same-user processes elsewhere.
- Every `gh` invocation is **pinned to `github.com`** with `--hostname`. Without
  it, `gh` honors `GH_HOST`, so a machine configured for GitHub Enterprise would
  read an Enterprise token and send it to the hardcoded `uploads.github.com`.
  Pinning makes that fail closed instead.
- Filenames are **escaped before interpolation** into markdown, so a file named
  `evil](#) ![pwn.png` cannot inject its own link. This matters because the
  agent skill uploads files the agent did not name.
- The token is never printed, logged, or written to disk.

## Requirements and limits

- **Push access to the target repository.** Without it the endpoint answers 404.
- Uploaded assets resolve **only for you** until referenced in rendered content.
  Once referenced, visibility follows whatever references them — an asset
  embedded in a public PR becomes public.
- The endpoint is **undocumented** and may change without notice.

## Use with AI agents

`skills/github-pr-attachment/` is an [Agent Skill](https://agentskills.io) that
teaches a coding agent to upload a file and embed it in a PR, issue, or comment.
It works with Claude Code, Codex, OpenCode, Cursor, and other clients that
support the standard.

```bash
cp -r skills/github-pr-attachment ~/.claude/skills/
```

## Credit

The upload protocol was documented by
[drogers0/gh-image](https://github.com/drogers0/gh-image) (MIT), a more general
tool that also handles arbitrary file types, read-only repositories, and
downloads. Use it if you need those; this script is the narrow subset that needs
nothing but the `gh` token.

## License

[MIT](LICENSE)
