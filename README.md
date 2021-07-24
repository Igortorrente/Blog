# Caedrium blog

Repository of Caedrium Blog

## Installation

Install the jekyll packages

```bash
$ sudo apt-get install ruby-full build-essential && sudo gem install jekyll bundler
```

And install the specific version of the rake

```bash
$ sudo gem install rake -v 12.3.3
```

## Build

Enter the site folder

```bash
$ cd blog
```

Now build and run the site

```bash
$ bundle exec jekyll serve --livereload
```

## License

The theme used is the [Yet Another theme](https://github.com/jeffreytse/jekyll-theme-yat)
where the [MIT license © JeffreyTse](blog/LICENSE.txt) applies to all content under
[blog](blog). Except under the [blog/_posts](blog/_posts) and [blog/assets](blog/assets),
where the [GNU Affero General Public License v3.0 © Igor M. A. Torrente](LICENSE) applies.

The current version of the [Yet Another theme](https://github.com/jeffreytse/jekyll-theme-yat)
is based on the
[fix: typo in README.md · jeffreytse/jekyll-theme-yat@149c641 · GitHub](https://github.com/jeffreytse/jekyll-theme-yat/commit/149c641d21850f1adeba27513895893089f80c89)
commit.
