---
date: "2021-09-19"
title: Detecting duplicate slugs in Gatsby
---

### Background

At [OpenLoop Health](https://openloophealth.com), we leverage [Gatsby](https://www.gatsbyjs.com/) for our static content. A lot of our pages are created using `Markdown`, which makes it easy to offload some of this work to less technical members on our team. However, one issue immediately became apparent: with hundreds of pages created using `Markdown` and no formal warning or error when multiple pages share the same slug, it's impossible to know if there has been a collision.

### Detecting duplicate slugs

In Gatsby, pages that are written in `Markdown` are *created* in a file called `gatsby-node.js`. You can find it in the root of your site. Code in `gatsby-node.js` is run during the process of building your site (we want to know if there are duplicates *before* the build finishes).

Open up `gatsby-node.js` and look for where you query your `Markdown` content. For us, it looks something like this:

```js
const markdownContent = await graphql(`
  {
    allMarkdownRemark {
      nodes {
        fields {
          collection
        }
        frontmatter {
          path
        }
      }
    }
  }
`);
```

Normally, markdown content is queried with the intention of iterating through each node to create a new page (using the [createPages API](https://www.gatsbyjs.com/docs/reference/config-files/gatsby-node/#createPages)). We are going to leverage this variable (in our case, `markdownContent`) to find duplicate slugs. Don't forget to modify the code to use **your** variable name.

```js
const duplicateSlugs = markdownContent.data.allMarkdownRemark.nodes
  .map((page) => page.frontmatter.path)
  .filter((node, index, nodes) => nodes.indexOf(node) !== index);
```

Here, we filter `markdownContent` to return a list of every slug that is used in multiple pages. So, for example, if we had two pages that shared the same slug, `blog/first-post`, we'd see:

```js
markdownContent = ['blog/first-post']
```

### Handling duplicate slugs

Now that we've figured out a way to detect duplicate slugs in Gatsby, let's ensure that a build fails if duplicate(s) are detected. To do this, Gatsby provides a `reporter` helper. If you aren't already using it, look for where you call:

```js
exports.createPages = async ({ actions, graphql, reporter })
```

and ensure `reporter` is added above so you can access its methods.

Next, before you iterate through your `markdownContent` and create a page for each node, write:

```js
if (duplicateSlugs.length > 0)
  reporter.panic(`Duplicate slugs found at: ${duplicateSlugs.join(', ')}`);
```

This will cause the build process to **fail** and report any duplicate slugs to you in the console. It's recommended you do this *before* you create a page for each node to save yourself the time it takes to build new pages if the build process were to fail.

That's it! Feel free to verify it works to your liking by intentionally creating duplicate slugs in your `Markdown` pages and running a `yarn start`
