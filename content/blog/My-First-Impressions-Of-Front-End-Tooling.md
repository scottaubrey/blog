---
title: "My first impressions of front end tooling"
date: 2018-09-01T16:45:00Z
draft: false
---

# My first impressions of front end tooling 

I'm a back end developer by trade, usually PHP, and I'm much more at home in the worlds of APIs, microservices, databases, and ORMs than preprocessors, bundlers and async modules. 

However, for a side project I'm working on, I'm making a web presence "shop front" for an Etsy store. I don't profess to be much of a design person, but I know my way around front end technologies enough to respond and create someone else's design. 

At least, when I say I "know my way around", what I really mean is I know HTML, I know how CSS works, and I know javascript, and I know "good enough" practises from about 10 years ago. What I don't know, is how these technologies are used in a modern workflow with massive CSS and javascript frameworks that we have today. So this is my little experiment into front end technologies, and in particular my evaluation of the tooling involved.

## The prototype

I've used bootstrap as a Grid/CSS framework before - and that seems as good a place to start as any. For the initial prototype I used some PHP components to get a back end system serving up some templated HTML (Remember: web-presence - No SPAs right now. I could probably get away with static HTML generator if I didn't have grander plans for the future). Then I linked the bootstrap using `<link>` and `<script>` tags from the CDN, and started building out the front end HTML, using custom css and js files served from the local webserver.

Once I was resonably happy with the design prototype, I started to research and experiment with what tools I could use to make this better.

## Step 1: Package manager

Here's the extent of my knowledge prior to this:

* I use composer extensively in my PHP work - package managers are really useful for managing dependacies.
* I have used npm for node.js experiments, and had heard of a thing called "bower" which I remember someone telling me was the npm of the frontend world. I also recently had a project at work where someone was using something about "yarn".

That's about it. It might not be much, but it was enough to get me googling. After about 20 minutes of googling around, I think I discovered this:

* Bower is dead. It recommends "Yarn and Webpack, or Parcel". 
* Npm is now the go to repo for frontend *and* node.js libraries, including CSS.
* Yarn is a replacement for npm - it's better because it has a lock file (npm doesn't?!?)
* There are a few others too, but don't seem to get as much attention.

As I mentioned, I've used composer for years, and cannot live without it in PHP land, so got to work installing yarn. Yarn is a package in Homebrew for macOS - so installed that. Quickly running `yarn` in my project directory runs what I would call an install process, but since I have nothing installed, just created an empty `node_modules` directory, and `yarn.lock` file. Another round of googling, and from bootstrap's homepage I gather the npm package is called 'bootstrap', and from yarn's documentation that to add a new dependancy I need to use `yarn add`. So, `yarn add bootstrap` I go.

Being used to a dependancy resolver built into composer, I came away a bit surprised that yarn didn't automatically resolve and install all dependancies. Researching the issue, I can't say I'm anymore the wiser - they are not marked as optional dependancies, but I have no other dependancies defined locally to conflict with.

I don't know if I need these dependancies, or will cause problems by not having them. In any case, I resolved them by `yarn add query popper.js`. After installing those dependancies, I no longer had warning messages beyond lacking a license field in my package.json, or a file.

## Serving up my `node_modules`

Coming from old school web front end dev, I'm used to dropping my dependancies in a `public` or similar folder, and having them served by the webserver. This weird hybrid of front end dependancies being put into an equivalent of a vendor folder confused me somewhat. My first attempt to correct this involved moving the node_modules folder. After yet more research, I found the `.yarnrc` config file, and the `--install.modules-folder` option. After adding it with  `--install.modules-folder public/assets/modules`  to the root of the folder, I ran `yarn` again and found all my dependancies in my public folder. Altering my HTML `<link>` and `<script>` tags to find the new "dist" versions in my `node_modules` folder, and I was finally in business with a locally served, but version managed dependancies!

## Step 2: Bundles to the rescue!

I was still bothered by this setup. Firstly, I have no idea what other files are brought in packages by yarn, and I'm just serviing them up as is. Secondly, the paths that I had to use in HTML seem archiac: `/assets/modules/bootstrap/dist/css/boostrap.min.css`, etc. Finally, I was pretty sure this was not the right thing to do.

So, a bit more researching, and I discovered what I need is a bundler! When researching ealier about bower, I'd looked at webpack and parcel.js. parcel.js had promised zero config, and since I don't know what I'm config'ing - I figure that's a good place to start!

I removed `.yarnrc` and ran `yarn` again, moving the folder back to `node_modules`. Then following the parcel docs, I ran `yarn global add parcel-bundler` which installed it to `/usr/local/bin/parcel`, then ran `parcel`, and was greeted with this:

```bash
$ parcel
Server running at http://localhost:1234
✨  Built in 41ms.	
```

Web server? Reading up a bit more, it seems to be geared towards SPA, and provides a simple webserver to serve up your "entry point" html, then parcel interprets that file's assets, recursively, to build the bundle.

That's great, but not what I need. More reading I ended up with `parcel watch public/assets/app.js --out-dir public/dist --public-url=/dist` for development which watches the file `public/assets/app.js` (and it's dependancies), and if anything changes, rebuilds the bundled files in the `public/dist` directory. Inside the `app.js` file, I had this:

```javascript
import "bootswatch/dist/lux/bootstrap.min.css";
import "./app.css";
import "bootstrap";
```

and in my HTML template I replaced my previous `<link>` and `<script>` tags with :

```html
<script src="/dist/app.js"></script>
<link rel="stylesheet" href="/dist/app.css">
```
This contains all the css and js my app needed. When ready for production build, I'll use `parcel build public/assets/app.js --out-dir public/dist --public-url=/dist` for production.

## Bonus step: SCSS in your bundles.

Bootstrap is big. I really didn't need all of it, but wanted to keep my options for using more of it open. I know bootstrap uses Sass preprocessor to build bootstrap with, so I embarked to do the same with the aim of reducing the overall size of my bundle.

First, install the parcel recommended sass compiler:

```bash
yarn add node-sass
```

Then, renaming `app.css` to `app.scss`, and some slight tweaks to `app.js` to remove the production build of bootswatch/bootstrap:

```javascript
import "./app.scss";
import "bootstrap";
```

Finally, adding in the required scss modules into app.scss. These were copied from `bootstrap/scss/bootstrap.scss` and prefixed with `~bootstrap/scss/` (I found the tilde (`~`) prefix in an example somewhere, tried it and it works. It seems to be a prefix in other projects to mean "node_modules path" for a number of projects, but I can't work out of this is parcel, node-sass or something else making it work). I had fun working out the ordering with bootswatch too. Final version of `app.scss` looked like this:

```scss
// import lux variables
@import "~bootswatch/dist/lux/_variables.scss";
// import bootstrap modules (or not)
@import "~bootstrap/scss/_functions.scss";
@import "~bootstrap/scss/_variables.scss";
@import "~bootstrap/scss/_mixins.scss";
@import "~bootstrap/scss/_root.scss";
@import "~bootstrap/scss/_reboot.scss";
@import "~bootstrap/scss/_type.scss";
// @import "~bootstrap/scss/_images.scss";
// @import "~bootstrap/scss/_code.scss";
@import "~bootstrap/scss/_grid.scss";
// @import "~bootstrap/scss/_tables.scss";
// @import "~bootstrap/scss/_forms.scss";
@import "~bootstrap/scss/_buttons.scss";
@import "~bootstrap/scss/_transitions.scss";
@import "~bootstrap/scss/_dropdown.scss";
// @import "~bootstrap/scss/_button-group.scss";
// @import "~bootstrap/scss/_input-group.scss";
// @import "~bootstrap/scss/_custom-forms.scss";
@import "~bootstrap/scss/_nav.scss";
@import "~bootstrap/scss/_navbar.scss";
// @import "~bootstrap/scss/_card.scss";
// @import "~bootstrap/scss/_breadcrumb.scss";
// @import "~bootstrap/scss/_pagination.scss";
// @import "~bootstrap/scss/_badge.scss";
// @import "~bootstrap/scss/_jumbotron.scss";
// @import "~bootstrap/scss/_alert.scss";
// @import "~bootstrap/scss/_progress.scss";
@import "~bootstrap/scss/_media.scss";
// @import "~bootstrap/scss/_list-group.scss";
// @import "~bootstrap/scss/_close.scss";
// @import "~bootstrap/scss/_modal.scss";
// @import "~bootstrap/scss/_tooltip.scss";
// @import "~bootstrap/scss/_popover.scss";
// @import "~bootstrap/scss/_carousel.scss";
@import "~bootstrap/scss/_utilities.scss";
@import "~bootstrap/scss/_print.scss";
// finally import lux-specific styles
@import "~bootswatch/dist/lux/_bootswatch.scss";

// my custom styles here
```
This is just a guess about which modules I would need, but it worked, and total delivery went from 125kb to 72kb, which still feels big, but at least a bit smaller.

## Final thoughts

Although presented fairly straightforwardly here, there was quite a bit of barely useful information to work through online, from stack overflow answers that barely answer the question, to documentation focusing not on documenting the tool, but documenting the authors desrired workflow. As an outsider, there seems to be a lot of assumptions in the frontend and node.js communities about what and how things should be setup that make it unnecessarily more difficult to integrate into another project IMO. 

I'm also not a big fan of trusting "magic" - by which I mean "prefix this path with ~ and it will find it in the node_modules folder", which a lot of advice online seems to fall into this trap. Why does this work? Why that character? Can I change the path? I hit a few issues trying to dig under this as I stepped outside of defaults, and it makes it difficult to understand where the problem is. 

Finally, I had a number of stability issues with the tools themselves. Yarn failed to install packages a few times, seemingly requiring me to remove the whole `node_modules` and/or `yarn.lock`. `parcel watch` seemed to easily hit file handle limits. Yarn complains of deprecated API usage. None of these things stopped me, but the js communities reputation for fast change and low quality seems, at least for these supposedly well-usage packages, to be at least a little deserved.

Overall - it worked. And I think I understand most of the reasons why. but I'd not be surprised to come back to this in a few months and find it doesn't work. I don't like feeling like that.