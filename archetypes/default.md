---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: "{{ .Date }}"
lastmod: '{{ .Lastmod | time.Format ":date_medium" }}'
draft: true
---
