---
title: "Path Traversal"
kind: module
category: "Server-side"
module: path-traversal
order: 1
date: 2026-10-07
summary: "Reading arbitrary files from the server by manipulating file path parameters."
---

<!-- Replace this paragraph with your collated module notes from Obsidian. -->
Path traversal lets an attacker read files outside the directory an application intends to serve from, by injecting `../` sequences or absolute paths into a parameter that is used to build a filesystem path.

## Labs

{% include websec-labs.html module=page.module %}
