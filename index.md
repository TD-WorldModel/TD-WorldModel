---
layout: project
title: Contrastive is all you need
subtitle:
description: A project page for Contrastive World Models.

# ---- Authors (the sup numbers map to the affiliations list below) ----
authors:
  - name: William Peng 
    url: "#"
    affiliations: "1"
  - name: Holger Molin 
    url: "#"
    affiliations: "1"
  - name: Third Author
    url: "#"
    affiliations: "2"

affiliations:
  - id: "1"
    name: Stanford University
  - id: "2"
    name: Collaborating Lab

# ---- Buttons. Common icons: fas fa-file-pdf | ai ai-arxiv | fab fa-github | far fa-images ----
links:
  - text: Paper
    url: "#"
    icon: fas fa-file-pdf
  - text: arXiv
    url: "#"
    icon: ai ai-arxiv
  - text: Code
    url: "#"
    icon: fab fa-github

# ---- Teaser (optional). Drop a file in static/images/ and point here. ----
# teaser: /static/images/teaser.png
# teaser_caption: A one-sentence caption describing your headline result.

# ---- Citation (optional) ----
bibtex: |
  @article{yourkey2026contrastive,
    author  = {First Author and Second Author and Third Author},
    title   = {Contrastive World Models},
    journal = {arXiv preprint arXiv:XXXX.XXXXX},
    year    = {2026},
  }
---

## Abstract

Write your abstract here. Everything below the `---` block is plain Markdown,
so just write prose. Summarize the problem, your key idea (contrastive
representation learning for world models), the method, and the headline results.

A second paragraph can describe experimental highlights and broader implications.

## Method

Describe your approach: the architecture, the contrastive objective, and how the
world model is trained and used for planning. Drop a figure in `static/images/`
and reference it:

![Method overview]({{ '/static/images/method.png' | relative_url }})

## Results

### Qualitative results

Use normal Markdown — images, lists, and tables all work:

![Result]({{ '/static/images/result1.png' | relative_url }})

### Quantitative comparison

| Method        | Metric A | Metric B |
|---------------|:--------:|:--------:|
| Baseline      |   0.00   |   0.00   |
| **Ours**      | **0.00** | **0.00** |

Add as many `##` sections as you need — this is one long scrolling page.
