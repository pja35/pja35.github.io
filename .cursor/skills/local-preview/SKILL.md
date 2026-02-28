---
name: local-preview
description: Serve the Jekyll site locally and open it in a browser. Use when the user wants to preview, view, visualize, or test the site locally, or asks to see their changes.
---

# Local Preview

Ensure the Jekyll site is served locally and open a browser to view it.

## Prerequisites

This project requires Ruby, ruby-dev, and Bundler installed system-wide:

```bash
ruby --version        # needs Ruby
bundle --version      # needs Bundler (gem install bundler)
```

If `bundle` is missing, ask the user to run `sudo gem install bundler`.
If native gem builds fail with missing headers, ask the user to run `sudo apt-get install -y ruby-dev`.

## Workflow

### 1. Check if already serving

```bash
lsof -ti:4000
```

- If a process is listening on port 4000, the server is likely already running. Skip to step 3.

### 2. Start the server

Run `bundle install` if `Gemfile.lock` is missing or stale, then start Jekyll:

```bash
bundle install
bundle exec jekyll serve
```

Background the `jekyll serve` command (`block_until_ms: 15000`). Confirm the output contains `Server running... press ctrl-c to stop.` before proceeding.

If port 4000 is busy from a stale process, kill it first:

```bash
lsof -ti:4000 | xargs kill -9
```

### 3. Open the browser

```bash
xdg-open http://127.0.0.1:4000/
```

### 4. Report to the user

Tell the user the site is live at `http://127.0.0.1:4000/` with auto-regeneration enabled. Remind them to press Ctrl+C in the terminal to stop the server when done.
