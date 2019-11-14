# faical.dev

This repository contains the source files of the articles I publish on my blog [https://faical.dev](https://faical.dev). I write my articles in [Markdown](https://daringfireball.net/projects/markdown/) and use a custom-made command line tool to manage and convert them to static HTML files.

## Workflow

I wrote a command line tool in Swift called `article` to handle everything related to my blog; from scaffolding files and directories to generating static HTML files. My typical flow with `article` would be:

#### Create

```shell
~ $ article new <article-id>
```

This command creates a new article with the id `article-id` in my local blog source directory.

#### Write

```shell
~ $ article open <article-id>
```

This command opens the Markdown file corresponding to the article with id `article-id` in my favorite Markdown editor [iAWriter](https://ia.net/writer).

#### Preview

```shell
~ $ article make <article-id> && article open-html <article-id>
```

To preview the current article I'm working on, I *build* (generate the static HTML files from the Markdown files) the article and open the resulting artifact in my web browser with the command above.

#### Publish

When I'm ready to publish, I open a pull request on this repository. A Bitrise workflow is triggered whenever a PR is merged to `master` or a commit is directly made to `master`. The Bitrise workflow uses `article` to rebuild the whole blog and uploads it to an Amazon S3 bucket.
