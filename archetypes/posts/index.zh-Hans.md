---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
description: # in html metadata
summary: # for user interface
categories:
  - Tech
tags:
  - hugo
keywords:
  - hugo # in html metadata
feature: ./feature.png # The feature will be used in place of both the thumb and cover images, and in the article metadata, which is included when content is shared to third-party networks like Facebook and Twitter.
featureAlt: 
cover: ./cover.png # The cover image will be displayed at the top of the article content on individual article pages.
coverAlt: 
coverCaption:
thumbnail: ./thumb.png # The thumb image is used as the article thumbnail and will be displayed in article lists
thumbnailAlt: 
images:
  - ./feature.png
  - ./thumb.png
  - ./cover.png
slug: {{ .File.ContentBaseName }}
audio:
  - ./tts.ogg
videos:
  - ./intro.mp4
lastmod: {{ .Date }}
publishDate: {{ .Date }}
expiryDate: 
# externalUrl: # Go to external website directly
type: {{ .Type }}
author: holger
showComments: true
author: holger
isCJKLanguage: true
draft: false
showTableOfContents: true
---
