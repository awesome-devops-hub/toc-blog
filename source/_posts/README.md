# toc-blog

This is the origin repo serving all the raw blogs, which is the markdown file.

[Toc Blog](https://blog.toc-platform.com/)

## Tech stacks

- Hexo
- github pages
- github workflow
- AWS Route53
- markdown
- CI/CD

## How to create a blog post

1. run `./auto/create-template postname` to create a new markdown file in `source/_posts`.
1. Fill your content.
1. run `./auto/run-locally` to preview your post locally.
1. Create a test branch and create PR on this repo.
1. Wait the pr workflow success.
1. After the administrator of toc blog approved and merged, your blog will be displayed on our blog system.

## How to add image to a blog

### Where to put image

Create a folder the same name as your post in `source/images`.

### How to qoute image

Using ![](/images/${foldername}/${imagename}) in your post.

## How to change a theme

Currently, the theme is cloned from github. To use another theme, you need:


1. Change git submodule using.

```
  git submodule add giturl themes/themename
```

1. Change the `theme` `in _config.yml`.
