# Contributing

You will find here the conventions and rules to publish new blog posts and to develop new blog features and enhancements.

Please,
- [open an issue](https://github.com/src-d/blog/issues) to request any new feature or to report a bug.
- [create a new PR](https://github.com/src-d/blog/pulls) to request the review of a new post or feature.


## Table of contents

<!-- TOC -->

- [Contributing](#contributing)
    - [Table of contents](#table-of-contents)
    - [Preview the blog posts](#preview-the-blog-posts)
    - [Creating a new post](#creating-a-new-post)
        - [Content schema](#content-schema)
        - [Front matter](#front-matter)
        - [Formatting the content](#formatting-the-content)
        - [Links to other source{d} blog posts](#links-to-other-sourced-blog-posts)
        - [Assets](#assets)
        - [Authors](#authors)
    - [Peer review](#peer-review)
    - [How to publish blog posts and deploy the blog](#how-to-publish-blog-posts-and-deploy-the-blog)

<!-- /TOC -->


## Preview the blog posts

While you are developing or creating new contents for the blog, and always before publishing a PR, you should validate your changes locally, specially if you're modifying common features or you're using shortcodes.

To locally serve the blog, you need to satisfy the [project requirements](README.md#requirements), and then run from the project root:

```shell
make serve;
```
Finally, go to [http://localhost:8484](http://localhost:8484)


## Creating a new post

Posts are stored under [`post` content directory](content/post) as plain text `.md` files.

To create a new blog post, create a new `.md` file with under:<br />
`content/post/__POST_FILE_NAME__.md`<br />
it will be accessible in the the URL:<br />
`//__BLOG_HOST_NAME__/post/__POST_FILE_NAME__`

The post `.md` files must have the following [schema](#content-schema):


### Content schema

Every blog post content file is stored under the path `content/post/__POST_FILE_NAME__.md`, and has the following schema:

```
---
author: authorKeyName
date: 2006-01-02
title: "Post title"
image: /post/__POST_FILE_NAME__/header-image.png
description: "Short description of the post"
categories: ["science", "technical", "culture"]
draft: true
---

Whatever content in `markdown` format.

```


### Front matter

The **front matter** is the first section inside the content file &ndash;enclosed by "`---`" chars&ndash; where the content metadata is defined.<br />(Read more about it in the Hugo docs for the [Front Matter section](https://gohugo.io/content-management/front-matter))

- `author`: Defines the authorship as explained in the [authors section](#authors)
- `date`: Date when you plan to publish the post, `2006-01-02` format. **Do not forget to update it before publishing it.**
- `title`: Post title, as it will appear in the very top of the content
- `image`: Defines the image that will be shown in the [assets section](#assets)
- `description`: Short description of the post. Plain text only; neither HTML nor markdown allowed.
- `categories`: An array of at least one of the following: `science`, `technical`, `culture`. If it is not defined, it will not be served by the json blog api, so it will not appear in the source{d} landing.
- `draft`: This key should not be present in public posts. Read more in the [peer review section](#peer-review)


### Formatting the content

**It is strictly forbidden to use HTML tags and/or custom CSS to format the content of the blog.** These add unnecessary complexity to the blog maintenance and will eventually break in layout changes.

If you want/need special formatting for your content, please read the [shortcodes tutorial](https://blog.sourced.tech/documentation/shortcodes)

If you believe your content **really** requires a feature that is not currently supported by the current shortcodes, please [check the issues and create a feature request](https://github.com/src-d/blog/issues/). The product owner will evaluate feasibility and the benefit to all users before approving implementation.


### Links to other source{d} blog posts

Links from source{d} blog posts to other source{d} blog posts must be always relative to the blog url. That means that these urls will be like:
```
[link text](/post/__POST_FILE_NAME__)
```


### Assets

Given a post content file stored under <br />
`content/post/__POST_FILE_NAME__.md`,<br />
its assets should be stored under :<br />
`static/post/__POST_FILE_NAME__/__ASSET_FILE_NAME_PLUS_EXTENSION__`

That way, the linkable urls would be:<br />
`/post/__POST_FILE_NAME__/__ASSET_FILE_NAME_PLUS_EXTENSION__`

When demos or runnable stuff need resources, these must not be hosted in the blog repository, but externally. To do that, the [`codepen` shortcode](https://blog.sourced.tech/documentation/shortcodes#codepen) should be used.

### Authors

Every post entry must define its authorship; to do so, in the content [Front Matter section](#front-matter) it is needed to define the `author` key. 

The `author` key will be one of the authors defined by [the data/authors.yml](data/authors.yml).

The authors entries follow this schema:
```yaml
authorKeyName:
  name: Author Name and Surname
  thumbnail: https://avatars1.githubusercontent.com/u/__USER_ID__
  bio: "Short bio/description of the author"
  social:
    github: authorUserName
    twitter: authorTwitterHandler
```

If you are writing a blog post and your author entry is not defined under `data/authors.yml`, you should add yourself to that authors data file.


## Peer review

New content must be validated via [a PR](https://github.com/src-d/blog/pulls), by *at least both*:
- [vmarkovtsev](//github.com/vmarkovtsev) on the content;
- [platform](https://github.com/orgs/src-d/teams/platform/members) on the layout.

To let your peers review any new blog post, it needs to be published as a **"draft"**. Drafts are only accessible at staging environment http://blog-staging.srcd.run

To publish a post as a **draft** in staging, it is needed to set its `draft` [front matter](#front-matter) key to `true`, and merge it into [`src-d:staging` branch](https://github.com/src-d/blog/tree/staging) following the source{d} [Continous Delivery rules for web applications](https://github.com/src-d/guide/blob/master/engineering/continuous-delivery.md)


## How to publish blog posts and deploy the blog

The blog is published automatically following the source{d} [Continous Delivery rules for web applications](https://github.com/src-d/guide/blob/master/engineering/continuous-delivery.md)

For any blog post to be published, it must follow the conventions given by this guide, and the following technical ones regarding the post [front matter](#front-matter):
- `draft` key must be unset (or at least it must be `false`),
- `date` key must be set to before &ndash;or equals to&ndash; the deploy date
