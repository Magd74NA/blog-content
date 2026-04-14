---
title: "JDExtractor: a local-first job search CRM"
date: "2026-04-09"
updated: "2026-04-09"
categories:
  - "go"
  - "svelte"
  - "llm"
  - "projects"
excerpt: "A single Go binary that wraps a Svelte UI, an LLM-driven resume tailoring pipeline, and a filesystem-as-database job tracker. Notes on the constraints I picked and what they bought me."
coverImage: "/images/posts/jdextractor-cover.svg"
coverWidth: 1200
coverHeight: 630
---

[JDExtractor](https://github.com/Magd74NA/jdextractor) is a tool I built to take the cognitive fatigue out of job applications. You paste a job posting URL, it fetches the description, runs your resume and cover letter through an LLM that aligns them to the role, and drops the output into a structured folder you can review and edit. The same binary also tracks every application you've submitted and helps you draft AI follow-ups for the people you've networked with.

It is *not* an auto-applier. It automates context-switching, not applying.

Below is a tour of the constraints I picked when I started, and what each one ended up buying me.

## Constraint 1: zero databases

The filesystem is the database. Every run creates a folder named `YYYY-MM-DD-{rand8}-{role-slug}` containing `meta.json`, `resume.txt`, and (optionally) `cover.txt`. Listing your applications is `os.ReadDir` plus a JSON unmarshal per folder.

What this bought me:

- **Trivial backup story:** your job hunt is a directory you can `tar` and copy.
- **Git friendliness:** you can version-control your applications if you want, and diffs are meaningful because the output is plain text.
- **No migrations.** I rewrote the metadata struct three times during development and never had to write a migration. The cost is a tiny `if field == ""` check on read.

## Constraint 2: zero Go module dependencies

The whole project is `go 1.25` with no `require` block. Every line of Go is either mine or stdlib.

This is harsh, but it forced a few good decisions:

- HTTP client is `net/http` with hand-rolled exponential backoff. No `resty`, no `retryablehttp`.
- The LLM "client" is a 50-line file that marshals a request body and parses the response. There is no SDK to keep up with.
- The web server is `net/http.ServeMux` with a CSRF middleware that checks the `Origin` header against `localhost:{port}`. No router framework.

The pleasant surprise: **the binary is small and the build is fast.** `go build` takes a second on a cold cache. There is nothing to vendor, nothing to audit, nothing to upgrade.

## Constraint 3: single binary, with a real frontend embedded

The UI is a Svelte 5 + Vite app that gets built into static assets, then embedded into the Go binary via `//go:embed web/dist`. The Go web server serves them from memory. Users download one file and run it.

The Makefile target is exactly what you'd hope:

```makefile
build: ui-build vet
	go build -trimpath -ldflags="-s -w" -o out/jdextractor ./cmd
```

`ui-build` runs `npm run build` and then deletes the non-gzip artifacts under `jdextract/web/dist`, so only the gzipped versions get embedded. The Go binary serves the gzipped files directly when the client supports it, which keeps the embedded payload small.

## Constraint 4: no headless browsers

To handle JS-heavy job sites without bundling Chromium, JDExtractor pipes every URL through `https://r.jina.ai/{fullURL}`, which returns clean markdown. Jina's reader endpoint requires no API key and handles SPAs gracefully. The result gets parsed by a small line-level classifier (`parse.go`) that tags each line as one of about 15 node types (title, list item, body, jina marker, nav link) and drops the noise before sending the rest to the LLM.

## Constraint 5: plain text mode, not JSON mode

DeepSeek supports a `response_format: {"type": "json_object"}` flag that constrains the model to emit valid JSON. I tried it and the resume quality dropped. The model has to simultaneously reason about content *and* maintain JSON syntax, and the two compete for the same generation capacity.

So instead, the system prompt asks the model to wrap each output field in XML-like tags:

```text
<company>Acme Corp</company>
<role>Senior Copywriter</role>
<score>7</score>
<resume>
full tailored resume text
</resume>
<cover>
optional tailored cover letter
</cover>
```

Five compiled regexps with `(?s)` (dot-matches-newline) extract each section. Plain-text resumes will never naturally contain a literal `<resume>` token, so the parser holds up fine in practice. If a required field is empty after extraction, `GenerateAll` errors immediately with a diagnostic, surfacing prompt-compliance failures instead of writing empty files.

## What's in the box

The Svelte UI has five sections:

- **Dashboard:** application stats, activity charts, match-score distribution, and an overdue follow-up queue.
- **Jobs:** a filterable table with inline editing and file viewers.
- **Process:** paste URLs (single or batch) or raw text; streaming output shows real-time progress.
- **Contacts:** manage networking contacts, log conversations, generate AI follow-up messages, and link contacts to jobs.
- **Settings:** pick the backend (DeepSeek or Kimi), set your API key, edit templates and prompts.

Before any data is sent to the LLM, a `sanitize.go` pass redacts emails and phone numbers. I am careful about what leaves the machine.

If you want to try it, the [releases page](https://github.com/Magd74NA/jdextractor/releases) has prebuilt binaries for Linux, macOS, and Windows. Run `./jdextractor setup` once to scaffold `config/templates/`, drop your real resume into `config/templates/resume.txt`, then `./jdextractor serve --open` and you're off.
