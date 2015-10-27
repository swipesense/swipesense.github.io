---
layout: post
title: "Serving webfonts to IE9 from a CDN"
description: "Or how to solve \"@font-face failed cross-origin request. Resource access is restricted.\""
permalink: "serving-webfonts-to-ie9-from-a-cdn"
categories: []
tags: []
date: 2015-10-27
published: true
---

# The Problem

You're likely here because you're having trouble loading font-awesome or some other webfont in IE9. Rest-assured, you're not alone. Google will reveal many people who have experienced various facets of this problem, but this post intends to offer a definitive guide for getting IE9 to play nicely with webfonts. Depending on your setup, you might experience one or both of the issues below.

* You're serving your assets from a CDN such as CloudFront, and your IE9 developer console shows: ```@font-face failed cross-origin request. Resource access is restricted.```
* You're serving fonts to an HTTPS site, and after refreshing/reloading the page, the font fails to load. Clicking internal site links then causes the font to show up.
  * When viewing the developer console, the network tab shows that multiple font formats are fetched with a 200, but the received field shows a very small size (B rather than KB).
  * Pages loaded from internal links correctly fetch just the .eot file with either a 302 (from cache) or a received of the expected size.

# The Solution TL;DR

IE9 requires the following before it will correctly load webfonts from a remote cdn over https:

* The css must be defined like so:

    ```
    @font-face {
      font-family: 'MyWebFont';
      src: url('webfont.eot'); /* IE9 Compat Modes */
      src: url('webfont.eot?#iefix') format('embedded-opentype'), /* IE6-IE8 */
           url('webfont.woff') format('woff'), /* Modern Browsers */
           url('webfont.ttf')  format('truetype'), /* Safari, Android, iOS */
           url('webfont.svg#svgFontName') format('svg'); /* Legacy iOS */
      }
    ```

* The HTTP response header ```Access-Control-Allow-Origin``` must be present and must be set to ```*```
* The HTTP reponse header ```Vary``` must **not** be present.
* The HTTP response header ```Pragma: no-cache``` must **not** be present.
* The HTTP response header ```Cache-Control``` must be present and must **not** contain ```no-cache``` or ```no-store```. It must have a ```max-age``` set.
* The .eot file must return a mime type of ```application/vnd.ms-fontobject```

Our working solution served a .eot webfont with the following headers. Besides the requirements above, the other headers are optional:

```
HTTP/1.1 200 OK
Content-Type: application/vnd.ms-fontobject
Content-Length: 68875
Connection: keep-alive
Access-Control-Allow-Methods: HEAD, GET, POST, PUT, OPTIONS, DELETE
Access-Control-Allow-Origin: *
Access-Control-Max-Age: 31557600
Cache-Control: public, max-age=31536000
Date: Tue, 27 Oct 2015 21:24:31 GMT
Last-Modified: Mon, 26 Oct 2015 17:42:25 GMT
Server: nginx/1.6.2
X-Cache: Miss from cloudfront
Via: 1.1 7de459deeb400b606355e335ec12751a.cloudfront.net (CloudFront)
X-Amz-Cf-Id: Rv_2HKDGyTVXGwd1iJkVXhaZ4R78ooKBXg9pLPmc49N8_VMFI9EwFw==
```

# Our Setup

Before digging into the solution, I should mention that we are running a rails 4.2 application with a HTTPS endpoint and an asset host pointing to Amazon CloudFront. Our initial symptom was font-awesome icons showing up as squares. Below, I will give general a general explanation that should apply to anyone experiencing this issue, as well as the solution that worked for our particular stack.

# Solving the cross-origin request error

When fetching fonts from a remote location, IE9 uses the server's cross-origin request policy to verify that it can load the requested resource. [Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) is a security mechanism that uses a special collection of HTTP headers to define which sites (origins) may access a requested resource. When IE requests the fonts from your CDN, it expects a specific set of headers in the response, otherwise it ignores the requested resource.

If you're using CloudFront as your CDN, there are two steps for getting it to respect your CORS policy.

1. Follow this [guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/header-caching.html#header-caching-web-cors) which will walk you through configuring your CloudFront distribution to forward the necessary headers from each request. The short of it is that the cdn must forward at least the ```Origin``` header to the host endpoint since CORS policies typically rely on knowing the requesting origin.

2. CloudFront (and other CDNs) don't just cache resources - they cache entire requests, headers included. Now that CloudFront is set to forward the correct headers to your server, your server must be configured to respond to font requests with the proper headers. I've collected a few implementations from various sources here and will give our rails solution at the bottom.
  * [S3](http://www.holovaty.com/writing/cors-ie-cloudfront/)
  * [.net](https://stackoverflow.com/questions/13415073/on-ie-css-font-face-works-only-when-navigating-through-inner-links)
  * [Apache](http://enable-cors.org/server_apache.html)

  Keep in mind, that these solutions set cors headers **globally**. Ensure you understand the security concerns here before just copy/pasting.

# Solving the reload error

So, you have correct CORS headers and have banished the dreaded IE console error. But wait! Fonts still aren't rendering on page reloads. There are a few HTTP headers that cause IE9 to ignore your resource entierely. At least one of these has been [submitted to Microsoft](https://connect.microsoft.com/IE/feedbackdetail/view/992569/font-face-not-working-with-internet-explorer-and-http-header-pragma-no-cache), but in true M$ fashion, it has been acknowledged and marked as "Won't Fix". Offending headers are:

* ```Pragma```
* ```Cache-Control``` when set to ```no-cache``` or ```no-store``` (or both)
* ```Vary```

The solution is to remove ```Pragma``` and ```Vary``` and to set ```Cache-Control``` to have a ```max-age=(any number 0 - whatever)```.

# Testing Your Solution

You can use curl from the command line to ensure that your headers are being properly set. Replace the Origin designation below with your own:

```
curl -H 'Origin: https://www.swipesense.com' -I <URL_OF_YOUR_FONT>
```

# Our Rails Solution

If, like us, you are on rails with cloudfront, you can find our fully working solution right here! It uses a custom middleware to populate the correct headers.

First, we added the following middleware to vendor/lib (although you can put it wherever you like). This middleware allows us to apply custom headers to asset requests.

```ruby
# vendor/lib/rack_headers.rb
# Modified from https://github.com/thomasklemm/Rails-Starting-Point/blob/master/lib/rack_headers.rb

require 'rack/utils'
#
# Rack::Headers
#
# Set custom HTTP Headers for based on rules in Rails 3:
#
#      # Put this in your config/environments/production.rb
#
#      # Rack Headers
#      # Set HTTP Headers on static assets
#      config.assets.header_rules = {
#        # Cache all static files in public caches (e.g. Rack::Cache)
#        #  as well as in the browser
#        all: {'Cache-Control' => 'public, max-age=31536000'},
#
#        # Provide web fonts with cross-origin access-control-headers
#        #  Firefox requires this when serving assets using a Content Delivery Network
#        fonts: {'Access-Control-Allow-Origin' => '*'}
#      }
#      require 'rack_headers'
#      config.middleware.insert_before '::ActionDispatch::Static', '::Rack::Headers'
#
#  Rules for selecting files:
#
#  1) All files
#     Provide the :all symbol
#     :all => Matches every file
#
#  2) Folders
#     Provide the folder path as a string
#     '/folder' or '/folder/subfolder' => Matches files in a certain folder
#
#  3) File Extensions
#     Provide the file extensions as an array
#     ['css', 'js'] or %w(css js) => Matches files ending in .css or .js
#
#  4) Regular Expressions / Regexp
#     Provide a regular expression
#     %r{\.(?:css|js)\z} => Matches files ending in .css or .js
#     /\.(?:eot|ttf|otf|woff|svg)\z/ => Matches files ending in
#       the most common web font formats (.eot, .ttf, .otf, .woff, .svg)
#       Note: This Regexp is available as a shortcut, using the :fonts rule
#
#  5) Font Shortcut
#     Provide the :fonts symbol
#     :fonts => Uses the Regexp rule stated right above to match all common web font endings
#
#  Rule Ordering:
#    Rules are applied in the order that they are provided.
#    List rather general rules above special ones.
#
module Rack
  class Headers

    def initialize(app, opts={})
      @app = app

      asset_path = Rails.application.config.assets.prefix || '/assets'
      @asset_path = opts.fetch(:path, asset_path)

      header_rules = Rails.application.config.assets.header_rules || []
      @header_rules = opts.fetch(:header_rules, header_rules)
    end

    def call(env)
      dup._call(env)
    end

    def _call(env)
      status, @headers, response = @app.call(env)
      @path = ::Rack::Utils.unescape(env['PATH_INFO'])

      # Set HTTP Headers
      set_headers if @path.start_with?(@asset_path)

      [status, @headers, response]
    end

    # Convert HTTP header rules to HTTP headers
    def set_headers
      @header_rules.each do |rule, headers|
        apply_rule(rule, headers)
      end
    end

    def apply_rule(rule, headers)
      case rule
      when :all    # All files
        set_header(headers)
      when :fonts  # Fonts Shortcut
        set_header(headers) if @path.match(/\.(?:ttf|otf|eot|woff|svg)\z/)
      when String  # Folder
        path = ::Rack::Utils.unescape(@path)
        set_header(headers) if (path.start_with?(rule) || path.start_with?('/' + rule))
      when Array   # Extension/Extensions
        extensions = rule.join('|')
        set_header(headers) if @path.match(/\.(#{extensions})\z/)
      when Regexp  # Flexible Regexp
        set_header(headers) if @path.match(rule)
      else
      end
    end

    def set_header(headers)
      headers.each do |field, content|
        if content.nil?
          @headers.delete(field)
        else
          @headers[field] = content
        end
      end
    end
  end
end

```

Next, we modified our assets production.rb to configure and insert the middleware.

```ruby
# config/environments/production.rb snippet

config.assets.header_rules = {
  # Cache all static files in public caches (e.g. Rack::Cache)
  # as well as in the browser/
  all: {'Cache-Control' => 'public, max-age=31536000'},

  # Provide web fonts with cross-origin access-control-headers.
  # Firefox/IE9 require this when serving assets using a CDN.
  fonts: {
    'Access-Control-Allow-Origin' => '*',
    'Vary' => nil # Removes the Vary header if it is set.
  }
}

require Rails.root.join('vendor', 'lib', 'rack_headers')
config.middleware.insert_after Rack::Sendfile, Rack::Headers
```

A quick gotcha to note is that this can interfere with rack-cors if you are using that for other CORS requests. Rack::Headers must be inserted before Rack::Cors in the middleware stack, otherwise the headers set here will be overwritten.

And that's it! Bump your config.assets.version to force expire all assets so CloudFront can repopulate.

This issue sidelined us for a good 12 hours, so hopefully this prevents you from the same fate. Did we miss another gotcha? Please [email us](mailto:jori@swipesense.com) so we can keep this guide up-to-date.
