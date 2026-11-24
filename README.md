# Research notes

Notes on technical questions I look into, one folder per topic. They are written in Markdown with Mermaid diagrams, and every source is public, mostly vendor documentation from Microsoft, Red Hat and AWS.

## Topics

| Topic | What it covers |
|---|---|
| [linux-windows-authentication](linux-windows-authentication/README.md) | How a Python app in a Linux container on ROSA HCP logs in to SQL Server with Windows Authentication: Kerberos, with a sidecar that keeps the ticket fresh. [Diagrams](https://su-ekachai.github.io/research-notes/linux-windows-authentication/architecture.html) |

## Reading on another machine

```bash
git clone https://github.com/su-ekachai/research-notes.git
cd research-notes
git pull   # later, to get new or updated topics
```

GitHub renders the Markdown and Mermaid diagrams in the browser. It shows `.html` files as source, so open those from the GitHub Pages site at `https://su-ekachai.github.io/research-notes/<topic>/<file>.html`, or locally after cloning.

## Adding a topic

1. Create a folder at the root. Name it in lowercase with hyphens between words, for example `kafka-on-openshift`.
2. Give it a `README.md` that opens with the short answer, then lists the other files in reading order.
3. Add a row for it to the table above.
