---
title: "Sample Specification"
version: "1.0.0"
latest: true
---

## How to run the specification site locally
- install [Hugo](https://gohugo.io/)
- Clone the repository at https://github.com/predicated-dev/predicate.dev
- Navigate to the folder and run `hugo serve` or `hugo server`

## Where to place new specifications
- Create a subfolder under the `content` folder with a short name for your specification. This folder will be used in the url, so make it lowercase, short and representative.
- Create an `_index.md` file with the full name of the specification

{{< codebox filename="content/sample/_index.md" lang="md" >}}
---
title: "Sample Specification"
---
{{< /codebox >}}

## Front-matter tags in specifications
- `title`: Shown before any other content in an H1 tag
- `version`: A [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html) compliant version number. Shown directly below the title in an H2 tag
- `latest`: true or false (default). Used to determine the specification shown when navigating to the specification main URL. At least one specification should be marked as `latest`, usually the latest release version of the specification 


{{< codebox filename="content/sample/1.0.0/index.md" lang="md" >}}
---
Title: "Sample Specification"
version: "1.0.0"
latest: true
---

Content of the specification goes here
{{< /codebox >}}

## Translations
- Check `hugo.toml` file to see if the language is present in the `[languages]` section, else add it.
- Note the language abbreviation.  Lets say its `xyz`
- Check if `i18n/xyz.yaml` is available (subsitute the language abbreviation for `xyz`)
- Translate the content of `i18n/xyz.yaml` based on `i18n/en.yaml`
- Locate the specification file to translate, say `content/sample/1.0.0/index.md`and copy it to `content/sample/1.0.0/index.xyz.md` (again substitute language abbreviation for `xyz`)

## Publishing your specification to predicate.dev
Create a pull request on Github with a request to add your specification. Once approved your specification will go live shortly after the merge is completed.
