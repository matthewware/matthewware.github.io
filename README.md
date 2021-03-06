@matthewware

[![Jekyll](https://img.shields.io/badge/jekyll-%3E%3D%203.6-blue.svg)](https://jekyllrb.com/)

Personal website and Blog [matthewware.github.io](https://matthewware.github.io)

## Building

### Local Development

1. Check that you have Ruby 2.1.0 or higher installed.

```
$ ruby --version
> ruby 2.X.X
```

If you don't have Ruby installed, install [Ruby 2.1.0 or higher](https://www.ruby-lang.org/en/downloads/).

2. Install Bundler

```
$ gem install bundler
```

3. Install all dependencies

```
$ bundle install
```

4. Build/serve the site locally

```
$ ./start_local.sh
```

This loads the site at `localhost:4444`.

#### Extras

For setting up URLs, especially with google, I found this [link](https://medium.com/@hossainkhan/using-custom-domain-for-github-pages-86b303d3918a) very helpful.

For adding tags to post, see [this](https://longqian.me/2017/02/09/github-jekyll-tag/).

### To-do
  - [ ] Dark mode switch
  - [ ] New font
