---
{{ $name := replaceRE `^[0-9]{4}-[0-9]{2}-[0-9]{2}-` "" .File.ContentBaseName -}}

title: {{ replace $name "-" " " | title }}
slug: {{ $name }}
date: {{ .Date }}
draft: true
---
