# predicate.dev
predicate.dev website for software specifications

## Create a new specification
- Create a subfolder under the `content` folder with a short name for your specification. This folder will be used in the url, so make it lowercase, short and representative.
- Create an `_index.md` file with the full name of the specification

**content/sample/_index.md**
```
---
title: "Sample Specification"
---
{{< /codebox >}}
 ```

## Front-matter tags in specifications
- `title`: Shown before any other content in an H1 tag
- `version`: A [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html) compliant version number. Shown directly below the title in an H2 tag
- `latest`: true or false (default). Used to determine the specification shown when navigating to the specification main URL. At least one specification should be marked as `latest`, usually the latest release version of the specification 

**content/sample/1.0.0/index.md**
``` markdown
---
Title: "Sample Specification"
version: "1.0.0"
latest: true
---

Content of the specification goes here
```


## Translations
- Check `hugo.toml` file to see if the language is present in the `[languages]` section, else add it.
- Note the language abbreviation.  Lets say its `xyz`
- Check if `i18n/xyz.yaml` is available (subsitute the language abbreviation for `xyz`)
- Translate the content of `i18n/xyz.yaml` based on `i18n/en.yaml`
- Locate the specification file to translate, say `content/sample/1.0.0/index.md`and copy it to `content/sample/1.0.0/index.xyz.md` (again substitute language abbreviation for `xyz`)

## Create a pull request with your specfication
- Create a pull request via GitHub and we will consider your submission for approval