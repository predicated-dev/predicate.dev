# predicate.dev
[predicate.dev](https://predicate.dev) - A website for software specifications

## How to run the specification site locally
- install [Hugo](https://gohugo.io/)
- Fork or clone this repository (https://github.com/predicated-dev/predicate.dev)
- Navigate to the folder and run `hugo serve` or `hugo server`

## Create a new specification
- Create a subfolder under the `content` folder with a short name for your specification. This folder will be used in the url, so make it lowercase, short and representative.
- Create an `_index.md` file with the full name of the specification 

**content/sample/_index.md** (subsitute `sample` with the spec foldername)
```
---
title: "Sample Specification"
description : "Sample specification that doubles as instructions on how to add your own spec"
---
```
- `title`: Shown in each versioned spec page in an H1 tag
- `description`: A meta tag added for search engines
- 
## Front-matter tags in specifications
- `version`: A [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html) compliant version number. Shown directly below the title in an H2 tag
- `latest`: true or false (default). Used to determine the specification shown when navigating to the specification main URL. At least one specification should be marked as `latest`, usually the latest release version of the specification 

**content/sample/1.0.0/index.md** (subsitute `sample` with the spec foldername)
``` markdown
---
version: "1.0.0"
latest: true
---

Content of the specification goes here
```

## Translations
- Check `hugo.toml` file to see if the language is present in the `[languages]` section, else add it.
- Note the language abbreviation. We will use `fr` (French) as an example. In all cases below subsitute `fr` for your language abbreviation
- Check if `i18n/fr.yaml` is available, if not copy `i18n/en.yaml` to `i18n/fr.yaml` 
- Translate the content of `i18n/fr.yaml` based on `i18n/en.yaml` as needed. 
- Check if `content/sample/_index.fr.md` is available, if not copy and translate `content/sample/_index.fr.md`
- Copy and translate a version of a specification file, for example to translate Semantic Version Query Language Specification 1.0.0 copy `content/svql/1.0.0/index.md` to `content/svql/1.0.0/index.fr.md`

## Create a pull request with your specfication
- Create a pull request via GitHub and we will consider your submission for approval
