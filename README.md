# Cheatsheets

A personal reference library for daily SWE / DevOps work and system-design / interview prep.
Markdown source of truth + a searchable [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
site on top.

Three ways to use it:

- **Grep** - `grep -ri "consumer group" docs/` while coding.
- **Browse** - searchable site with a sidebar (`mkdocs serve`).
- **Publish** - a public URL that doubles as content.

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve            # http://localhost:8000, live-reload
```

## Build / check

```bash
mkdocs build --strict   # fails on broken links or nav warnings
```

## Publish (GitHub Pages)

Every push to `main` runs `.github/workflows/deploy.yml`, which builds the site and
deploys it to GitHub Pages. One-time setup: in the repo, set **Settings > Pages >
Source** to **GitHub Actions**.

Live at: https://dancan254.github.io/cheatsheets/

## Structure

```
docs/
  java/         Java 25, Spring Boot 4, Testcontainers
  devops/       Docker, Kubernetes, AWS, Jenkins, GitHub Actions, Terraform
  linux/        Linux, Bash, text-tools, tmux/vim
  messaging/    Kafka, RabbitMQ, broker comparison
  system-design/ building blocks, patterns, CAP, napkin math
  interview/    LeetCode patterns, complexity, Java idioms
  data/         SQL, Flyway
  languages/    Go/Python/Node → Java Rosetta
```

Each sheet follows the same skeleton: a "when you reach for this" intro, sections
of real commands/snippets, a **Gotchas** section, and a **Quick reference** table.
