---
layout: post
title:  "Using file pattern matching in 'az artifacts universal'"
date:   2021-01-01 01:01:00 +0000
categories: jekyll update
---

**\--file-filter** option is available to filter files while downloading files from a Universal Package in Azure Devops.

I am looking to download a single file which is four directories deep,

**Ex: dir1/dir2/dir3/dir4/file**.


I tried to download the file with the following file-filter option and it didn't work:
```
az artifacts universal download --organization "<ORG>" --feed "<FEED>" \
 --name "<PKG_NAME>" --version "*" --file-filter "*/file" --path . 

```
This ended up downloading no files.

Digging through documentation I found File macthing patterns reference. On this page, there is a note **Note, extended globs cannot span directory separators**. This means, I cannot use a single **"*"** to match multiple directory levels.


Instead, I tried the following:
```
az artifacts universal download --organization "<ORG>" --feed  \
"<FEED>" --name "<PKG_NAME>" --version "*" --file-filter \
"*/*/*/*/file" --path .
```

and it worked. After running the above command I see the following file downloaded to my system:
```
dir1/dir2/dir3/dir4/file
```
