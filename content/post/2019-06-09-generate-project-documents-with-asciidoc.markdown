---
featured: false
title: "DocuOps with Asciidoc & Gradle plugin"
tags: [asciidoc, asciidocj, gradle, groovy, jruby, docuops]
date: 2019-06-09T14:24:10+10:00
lastmod: 2019-06-09T14:24:10+10:00
draft: false
---

![Header](/img/posts/2019-06-09-head.png#headerimage 'Spec')

If you love Asciidoc and want to generate cool technical specs in HTML5 and RevealJS slide for technical showcase. [This gradle plugin](https://github.com/thanhnamit/docgen-gradle-plugin) can help:

* Creating a sane default structure for project docos and templates
* Generating HTML5 docos (PlantUML supported)
* Generating RevealJS slide from Asciidoc format

To work with Asciidoc, refer to this [writer's guide](https://asciidoctor.org/docs/asciidoc-writers-guide/#listing-and-source-code-blocks), and [compare Asciidoc with markdown](https://asciidoctor.org/docs/asciidoc-vs-markdown/), also [heap of projects are using Asciidoc](https://github.com/asciidoctor/asciidoctor.org/issues/270)

## Examples

*Tech spec*
![Spec](/img/posts/2019-06-09-spec.png#bodyimage 'Spec')

*RevealJS slideshow*
![Spec](/img/posts/2019-06-09-slide.png#bodyimage 'Spec')

## Build
This plugin is not available in gradle plugin portal. You can build and push to local or private repositories, for example:

```sh
./gradlew clean build
./gradlew publishToMavenLocal
```
<p/>
 
## Installation
Simply decorate your build.gradle with following:

```groovy
plugins {
    id 'com.tna.gradle.docgen-plugin' version '1.0.0-SNAPSHOT'
}

docGen {
  //location to store documents for project (presentation, specs...)
  documentRoot 'src' 
  projectInfo {
    projectName = 'CoolApiV1'
    projectVersion = '1.0.0'
    projectAuthors = 'Author A, Author B'
    contactEmail = 'author@mail.com'   
  }
}
```
<p/>

## Usage 
This plugin has three tasks under `documentation` group:

`docGenDirTemplate` - generate sensible opinionated document structure with template files (README.adoc, LICENSE, CONTRIBUTING.adoc, CHANGELOG.adoc). 

`docGenHtml` - generate formated html5 from .adoc file under `docs/asciidoc/specification`. Note this path is created by the previous task.

`docGenSlide` - generate RevealJS presentation from .adoc file under `docs/asciidoc/presentation`.

Enjoy Asciidoc! :feet:

