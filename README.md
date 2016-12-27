# Jekyll Minibundle plugin

A straightforward asset bundling plugin for [Jekyll][Jekyll], utilizing
external minification tool of your choice. It provides asset
concatenation for bundling and asset fingerprinting with MD5 digest for
cache busting.

There are no runtime dependencies, except for the minification tool used
for bundling (fingerprinting has no dependencies).

The plugin requires Jekyll version 3.x. It is tested with Ruby MRI
2.x. Ruby 1.8 and 1.9 are *not* supported.

The plugin works with Jekyll's watch mode (auto-regeneration, Jekyll
option `--watch`), but not with incremental feature enabled (Jekyll
option `--incremental`).

[![Gem version](https://badge.fury.io/rb/jekyll-minibundle.svg)][MinibundleGem]
[![Build status](https://secure.travis-ci.org/tkareine/jekyll-minibundle.svg)][MinibundleBuild]

## Features

There are two features: asset fingerprinting with [MD5 digest][MD5] over
the contents of the asset, and asset bundling combined with the first
feature.

Asset bundling consists of concatenation and minification. The plugin
implements concatenation and leaves choosing the minification tool up to
you. [UglifyJS2][UglifyJS2] is a good and fast minifier, for
example. The plugin connects to the minifier with standard unix pipe,
feeding asset file contents to it in desired order via standard input,
and reads the result from standard output.

Why is this good? A fingerprint in asset's path is
the [recommended way][GoogleWebFundamentalsHttpCaching] to handle
caching of static resources, because you can allow browsers and
intermediate proxies to cache the asset for a very long
time. Calculating MD5 digest over the contents of the asset is fast and
the resulting digest is reasonably unique to be generated automatically.

Asset bundling is good for reducing the number of requests to the
backend upon page load. The minification of stylesheets and JavaScript
sources makes asset sizes smaller and thus faster to load over network.

## Usage

The plugin ships as a [RubyGem][MinibundleGem]. To install:

``` bash
$ gem install jekyll-minibundle
```

(You should use [Bundler][Bundler] to manage the gems in your project.)

Then, instruct Jekyll to load the gem by adding this line to the
[configuration file][JekyllConf] of your Jekyll site project
(`_config.yml`):

``` yaml
gems: ['jekyll/minibundle']
```

An alternative to using the `gems` configuration setting is to add
`_plugins/minibundle.rb` file to your site project with this line:

``` ruby
require 'jekyll/minibundle'
```

You must allow Jekyll to use custom plugins. That is, do not enable
Jekyll's `safe` setting.

### Asset fingerprinting

If you just want to have an MD5 fingerprint in your asset's path, use
`ministamp` [Liquid][Liquid] tag. For example:

``` html
<link href="{{ site.baseurl }}{% ministamp _assets/site.css assets/site.css %}" rel="stylesheet" media="screen, projection">
```

When it's time to render the `ministamp` tag, the plugin copies the
source file (`_assets/site.css`, the first tag argument) to the
specified destination path (`assets/site.css`, the second tag argument)
in Jekyll's output directory. The filename will contain a fingerprint.

Tag output, when `site.baseurl` is `/`:

``` html
<link href="/assets/site-390be921ee0eff063817bb5ef2954300.css" rel="stylesheet" media="screen, projection">
```

This feature is useful when combined with asset generation tools
external to Jekyll. For example, you can configure [Sass][Sass] to take
input files from `_assets/styles/*.scss` and to produce output to
`_tmp/site.css`. Then, you use `ministamp` tag to copy the file with a
fingerprint to Jekyll's output directory:

``` html
<link href="{{ site.baseurl }}{% ministamp _tmp/site.css assets/site.css %}" rel="stylesheet">
```

### Asset bundling

This is a straightforward way to bundle assets with any minification
tool that supports reading input from STDIN and writing the output to
STDOUT. You write the configuration for input sources directly into the
content file where you want the markup tag for the bundle file to
appear. The markup tag contains the path to the bundle file, and the
Jekyll's output directory will have the bundle file at that path. The
path will contain an MD5 fingerprint.

Place `minibundle` [Liquid][Liquid] block into your content file where
you want the generated markup to appear. Write bundling configuration
inside the block in [YAML][YAML] syntax. For example, to bundle a set of
JavaScript sources:

``` text
{% minibundle js %}
source_dir: _assets/scripts
destination_path: assets/site
baseurl: {{ site.baseurl }}
assets:
  - dependency
  - app
attributes:
  id: my-scripts
  async:
{% endminibundle %}
```

Then, specify the command for launching your favorite minifier in
`_config.yml`:

``` yaml
baseurl: /

minibundle:
  minifier_commands:
    js: node_modules/.bin/uglifyjs -
```

When it's time to render the `minibundle` block, the plugin launches the
minifier and connects to it with a Unix pipe. The plugin feeds the
contents of the asset files in `source_dir` directory as input to the
minifier (STDIN). The feeding order is the order of the files in the
`assets` key in the block configuration. The plugin expects the minifier
to produce output (STDOUT) and writes it to the file at
`destination_path` in Jekyll's output directory. The filename will
contain a fingerprint.

Block output in the content file:

``` html
<script src="/assets/site-8e764372a0dbd296033cb2a416f064b5.js" type="text/javascript" id="my-scripts" async></script>
```

You can pass custom attributes, like `id="my-scripts"` and `async`
above, to the generated markup with `attributes` map inside the
`minibundle` block.

For bundling CSS assets, use `css` as the argument to the `minibundle`
block:

``` text
{% minibundle css %}
source_dir: _assets/styles
destination_path: assets/site
baseurl: {{ site.baseurl }}
assets:
  - reset
  - common
attributes:
  media: screen
{% endminibundle %}
```

And then specify the minifier command in `_config.yml`:

``` yaml
minibundle:
  minifier_commands:
    css: _bin/remove_whitespace
    js: node_modules/.bin/uglifyjs -
```

### Minifier command specification

You can specify minifier commands in three places:

1. in `_config.yml` (as shown earlier):

   ``` yaml
   minibundle:
     minifier_commands:
       css: _bin/remove_whitespace
       js: node_modules/.bin/uglifyjs -
   ```

2. as environment variables:

   ``` bash
   export JEKYLL_MINIBUNDLE_CMD_CSS=_bin/remove_whitespace
   export JEKYLL_MINIBUNDLE_CMD_JS="node_modules/.bin/uglifyjs -"
   ```

3. inside the minibundle block with `minifier_cmd` setting, allowing
   blocks to have different commands from each other:

   ``` text
   {% minibundle js %}
   source_dir: _assets/scripts
   destination_path: assets/site
   minifier_cmd: node_modules/.bin/uglifyjs -
   assets:
     - dependency
     - app
   attributes:
     id: my-scripts
   {% endminibundle %}
   ```

These ways of specification are listed in increasing order of
specificity. Should multiple commands apply to a block, the most
specific one wins. For example, the `minifier_cmd` setting inside
`minibundle js` block overrides the setting in
`$JEKYLL_MINIBUNDLE_CMD_JS` environment variable.

### Recommended directory layout

It's recommended that you exclude the files you use as asset sources
from Jekyll itself. Otherwise, you end up with duplicate files in the
output directory.

For example, in the following snippet we're using `assets/src.css` as
asset source to `ministamp` tag:

``` html
<!-- BAD: unless assets dir is excluded, both src.css and dest.css will be copied to output directory -->
<link href="{{ site.baseurl }}{% ministamp assets/src.css assets/dest.css %}" rel="stylesheet" media="screen, projection">
```

By default, Jekyll includes this file to the output directory. As a
result, there will be both `src.css` and `dest-<md5>.css` files in
`_site/assets/` directory, which you probably do not want.

In order to avoid this, exclude the asset source file from
Jekyll. Because Jekyll excludes directories beginning with underscore
character (`_`), consider using the following directory layout:

* `_assets/` for JavaScript and CSS assets handled by the plugin that
  are in version control
* `_tmp/` for temporary JavaScript and CSS assets handled by the plugin
  that are not in version control (for example, Sass output files)
* `assets/` for images and other assets handled by Jekyll directly

See [Jekyll configuration][JekyllConf] for more about excluding files
and directories.

### Development mode

If you set `$JEKYLL_MINIBUNDLE_MODE` environment variable to
`development`, then the plugin will copy asset files as is to Jekyll's
output directory and omit fingerprinting. The `destination_path` setting
in `minibundle` block sets the destination directory for bundled
files. This is useful in development workflow, where you need the
filenames and line numbers of the original asset sources.

``` bash
$ JEKYLL_MINIBUNDLE_MODE=development jekyll serve --watch
```

Alternatively, you can enable development mode from `_config.yml`:

``` yaml
minibundle:
  mode: development
```

Should both be defined, the setting from the environment variable wins.

### Capturing Liquid output

Use Liquid's `capture` block to store output rendered inside the block
to a variable, as a string. Then you can process the string.

For example:

``` html
{% capture sitecss %}{% ministamp _assets/site.css assets/site.css %}{% endcapture %}
<link href="{{ site.cdnbaseurl }}{{ sitecss | remove_first: "assets/"}}" rel="stylesheet" media="screen, projection">
```

Output, provided `site.cdnbaseurl` is set to `https://cdn.example.com/`:

``` html
<link href="https://cdn.example.com/site-390be921ee0eff063817bb5ef2954300.css" rel="stylesheet" media="screen, projection">
```

## Example site

See the sources of [an example site][JekyllMinibundleExampleSite].

## Known caveats

The plugin does not work with Jekyll's incremental rebuild feature
(Jekyll option `--incremental`).

## License

MIT. See `LICENSE.txt`.

[Bundler]: http://bundler.io/
[GoogleWebFundamentalsHttpCaching]: https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching#invalidating-and-updating-cached-responses
[JekyllConf]: https://jekyllrb.com/docs/configuration/
[JekyllMinibundleExampleSite]: https://github.com/tkareine/jekyll-minibundle-example
[Jekyll]: https://jekyllrb.com/
[Liquid]: https://shopify.github.io/liquid/
[MD5]: https://en.wikipedia.org/wiki/MD5
[MinibundleBuild]: https://travis-ci.org/tkareine/jekyll-minibundle
[MinibundleGem]: https://rubygems.org/gems/jekyll-minibundle
[Sass]: http://sass-lang.com/
[UglifyJS2]: https://github.com/mishoo/UglifyJS2
[YAML]: http://www.yaml.org/
