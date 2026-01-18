---
title: Your Blog Post Title Here
author: Grace Li
pubDatetime: 2026-01-18T10:00:00Z  # YYYY-MM-DD or full timestamp
postSlug: your-custom-url-slug      # Optional: overrides the filename as the URL
featured: false                     # Set to true to pin this to the "Featured" section
draft: true                         # Set to false when you are ready to publish
tags:
  - general
  - machine-learning
  - coding
description: Write a short, catchy summary of your post here. This appears on the card.
---

## Introduction

Write your introduction here. You can use **bold**, *italics*, or [links](https://google.com).

## My First Section

Here is a list:
- Item 1
- Item 2
- Item 3

## Adding Code

You can easily add code blocks like this:

```python
def hello_world():
    print("Hello, Grace!")
```

## Adding Images

1. **Place your image** in the `public/assets/` folder.
2. **Link to it** using the path starting with `/`:

![My Image Caption](/assets/images/example-image.png)

## Adding Videos

### Local Video
If you have a video file (like `.mp4`), put it in `public/assets/` and use an HTML tag:

<video controls width="100%">
  <source src="/assets/videos/my-video.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### YouTube Video
You can embed a YouTube video using an iframe (Copy "Embed" code from YouTube):

<iframe width="100%" height="315" src="https://www.youtube.com/embed/dQw4w9WgXcQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

---
*Happy Writing!*
