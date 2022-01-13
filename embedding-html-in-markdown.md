---
description: >-
  discussing the requirements, and challenges, of using Redcarpet's markdown
  engine to mix html snippets with markdown content
---

# Embedding HTML in Markdown

Arit has been adding a "unified embed" tag to replace the numerous named liquid tags in Forem. Rather than `{% youtube 123123123 %}`, users will use `{% embed youtube.com/watch/123123123 %}` or similar. In either case, the desired outcome is a frame or embedding container is added to the rendered html from the tag.

When testing some of these, I noticed we would see html  comment tags being rendered as text (instead of `<!-- comment -->` you would see `&lt;!-- comment --%gt;` on the page.

Arit also sees this, but says she cannot replicate it in production, that it only happens locally. I'm as confused as she is and up for a sleuthing session.



#### Isolating the source of the issue

I looked at our articles controller's preview action, since I know that when you type markdown, you can see the rendered html either by posting/saving (create) or previewing (the preview tab has an async request to send the body markdown, and receive rendered html back).

Let's check what's [there](https://github.com/forem/forem/blob/main/app/controllers/articles\_controller.rb#L76) - it's a fairly straightforward setup of a few processes - "fix for preview" (presumably this sanitizes or cleans up input in a way that's similar to normal fixup, in my simple case it didn't change the input so it's safe for me to ignore that line)

```ruby
      fixed_body_markdown = MarkdownProcessor::Fixer::FixForPreview.call(params[:article_body])
      parsed = FrontMatterParser::Parser.new(:md).call(fixed_body_markdown)
      parsed_markdown = MarkdownProcessor::Parser.new(parsed.content, source: Article.new, user: current_user)
      processed_html = parsed_markdown.finalize
```

The `FrontMatterParser#call` is to handle the yaml front matter (if you use the Forem v1 Editor rather than the v2 Editor, the front matter is between the `----` lines at the top of the body, and lets you set the title, cover image, tags, while that's handled by UI elements in the v2 editor. It's _also_ safe to ignore that, and assume that `parsed == fixed_body_markdown == params[:article_body]` - i.e. the first two lines made no changes to the input.

Since testing code in controllers is a headache, I leveraged a helpful fact about Forem's Article class, it persists both the `body_markdown` (almost raw markdown input from the user) and the `processed_html` (the result of passing the markdown input to the rendering stage). So the fastest way to test this on real input from the console was to save an article with an embed tag, and then use it's `body_markdown` attribute to get the `parsed_markdown` and finalize it.

Here's a comment embed that I was working with (this comment belongs to a faker generated user's faker generated comment on my local dev environment, you probably don't have _this_ url).

```ruby
[15] pry(main)> Article.last.body_markdown
=> "{% embed http://localhost:3000/torpdarlene/comment/c %}"

# this embeds the following comment preview
[21] pry(main)> User.find_by(username: :torpdarlene).comments.find_by(id_code: :c).processed_html
=> "<p>Keffiyeh portland locavore lomo readymade waistcoat. Tofu pork belly farm-to-table. Schlitz paleo letterpress 8-bit helvetica austin cardigan gluten-free.</p>\n\n"
```

The problem ensues when we embed the comment and generate the processed\_html for the article, we get something we didn't want, a code block with the html escaped closing div tag, and a p tag with the html escaped comment about the end of the comment liquid tag view template.

```ruby
[22] pry(main)> Article.last.processed_html
  Article Load (2.7ms)  SELECT "articles".* FROM "articles" ORDER BY "articles"."id" DESC LIMIT $1  [["LIMIT", 1]]
=> "<p><!-- BEGIN app/views/comments/_liquid.html.erb --></p>\n<div class=\"liquid-comment\">\n    <div class=\"details\">\n      <a href=\"/torpdarlene\">\n        <img class=\"profile-pic\" src=\"/uploads/user/profile_image/4/96272310-dc87-4f6a-a5d5-b937810e99f3.png\" alt=\"torpdarlene profile image\" loading=\"lazy\">\n      </a>\n      <a href=\"/torpdarlene\">\n        <span class=\"comment-username\">Darlene Torp</span>\n      </a>\n      <span class=\"color-base-30 px-2 m:pl-0\" role=\"presentation\">•</span>\n\n<a href=\"http://localhost:3000/torpdarlene/comment/c\" class=\"comment-date crayons-link crayons-link--secondary fs-s\">\n  <time datetime=\"2022-01-11T20:32:35Z\">\n    Jan 11\n  </time>\n\n</a>\n\n    </div>\n    <div class=\"body\">\n      <p>Keffiyeh portland locavore lomo readymade waistcoat. Tofu pork belly farm-to-table. Schlitz paleo letterpress 8-bit helvetica austin cardigan gluten-free.</p>\n<div class=\"highlight js-code-highlight\">\n<pre class=\"highlight plaintext\"><code>&lt;/div&gt;\n</code></pre>\n<div class=\"highlight__panel js-actions-panel\">\n<div class=\"highlight__panel-action js-fullscreen-code-action\">\n    <svg xmlns=\"http://www.w3.org/2000/svg\" width=\"20px\" height=\"20px\" viewbox=\"0 0 24 24\" class=\"highlight-action crayons-icon highlight-action--fullscreen-on\"><title>Enter fullscreen mode</title>\n    <path d=\"M16 3h6v6h-2V5h-4V3zM2 3h6v2H4v4H2V3zm18 16v-4h2v6h-6v-2h4zM4 19h4v2H2v-6h2v4z\"></path>\n</svg>\n\n    <svg xmlns=\"http://www.w3.org/2000/svg\" width=\"20px\" height=\"20px\" viewbox=\"0 0 24 24\" class=\"highlight-action crayons-icon highlight-action--fullscreen-off\"><title>Exit fullscreen mode</title>\n    <path d=\"M18 7h4v2h-6V3h2v4zM8 9H2V7h4V3h2v6zm10 8v4h-2v-6h6v2h-4zM8 15v6H6v-4H2v-2h6z\"></path>\n</svg>\n\n</div>\n</div>\n</div>\n\n</div>\n<br>\n&lt;!-- END app/views/comments/_liquid.html.erb --&gt;\n</div>\n\n"
```

Everything here was fine until the closing `</p>` after gluten free, when a div is created to house a `<pre><code>` block (it's larger then it might be because controls for the code block are added during processing, but the core is `<code>&lt;/div&gt;\n</code></pre>`).

After that code block another line is added with what was intended to be an html comment, now showing in the article body as text:

```
</div>\n<br>\n&lt;!-- END app/views/comments/_liquid.html.erb --&gt;\n</div>\n\n"
```

![](<.gitbook/assets/Screenshot from 2022-01-13 11-30-45.png>)

So something is going wrong. But what?

### Tracing the Code along its path

#### ArticlesController#preview

Let's go back to the controller snippet and try to simulate this.\


```ruby
input = Article.last.body_markdown
=> "{% embed http://localhost:3000/torpdarlene/comment/c %}"

fixed_body_markdown = MarkdownProcessor::Fixer::FixForPreview.call input

# parsed returns a front matter parser (there's no front matter, so this is fine):
parsed = FrontMatterParser::Parser.new(:md).call(fixed_body_markdown)  
=> #<FrontMatterParser::Parsed:0x0000557081382150
 @content="{% embed http://localhost:3000/torpdarlene/comment/c %}",
 @front_matter={}>

#  I don't have a current user, let's re-use darlene (it won't matter):
  parsed_markdown = MarkdownProcessor::Parser.new(parsed.content, source: Article.new, user: User.find_by(username: :torpdarlene))
=> #<MarkdownProcessor::Parser:0x0000557081241cc8
 @content="{% embed http://localhost:3000/torpdarlene/comment/c %}",
 @source=
  #<Article:0x0000557081276130
   id: nil,
   ...

parsed_markdown.finalize
# long html blob, with code block (unwanted) and END app/views/comments/_liquid.html.erb comment as text
```

So, one thing I would check when the html for a rendered template comes out malformed is the template. It's here [https://github.com/forem/forem/blob/main/app/views/comments/\_liquid.html.erb](https://github.com/forem/forem/blob/main/app/views/comments/\_liquid.html.erb#L1) (that comment sure makes finding it easy!) - and it looks right (there aren't any mismatched tags, the parts of the template inside the `<% if comment %>` block seem to have been added to the rendered view, and the parts that are in the else block aren't rendered. So far so good (we could write a unit test for the view and pass a comment in the context, but there's an existing test in [https://github.com/forem/forem/blob/main/spec/liquid\_tags/comment\_tag\_spec.rb](https://github.com/forem/forem/blob/main/spec/liquid\_tags/comment\_tag\_spec.rb#L3) that covers the core idea of what should and shouldn't be showing here).

#### MarkdownProcessor::Parse#finalize

So let's take a look at this `MarkdownProcessor::Parser#finalize` method and see what _it_ does. We'll be passing it a string (it's the body markdown from the comment), a source and user keyword.&#x20;

If you're not used to pry, there are a few convenience utilities we can use. In my case, I'll use "show-source" and "cd". `cd` changes the current context so methods/names are looked up inside an object (instead of in Kernel or whatever object is the top-level when you start pry). I don't know that `irb` has the same facilities, but Forem's rails console launches a pry session for you, so this is accessible:

```ruby
[28] pry(main)> cd parsed_markdown
[29] pry(#<MarkdownProcessor::Parser>):1> show-source finalize

From: /home/djuber/src/forem/app/services/markdown_processor/parser.rb:18:
Owner: MarkdownProcessor::Parser
Visibility: public
Signature: finalize(link_attributes:?)
Number of lines: 24

def finalize(link_attributes: {})
  options = { hard_wrap: true, filter_html: false, link_attributes: link_attributes }
  renderer = Redcarpet::Render::HTMLRouge.new(options)
  markdown = Redcarpet::Markdown.new(renderer, Constants::Redcarpet::CONFIG)
  catch_xss_attempts(@content)
  code_tag_content = convert_code_tags_to_triple_backticks(@content)
  escaped_content = escape_liquid_tags_in_codeblock(code_tag_content)
  html = markdown.render(escaped_content)
  sanitized_content = sanitize_rendered_markdown(html)
  begin
    liquid_tag_options = { source: @source, user: @user }

    # NOTE: [@rhymes] liquid 5.0.0 does not support ActiveSupport::SafeBuffer,
    # a String substitute, hence we force the conversion before passing it to Liquid::Template.
    # See <https://github.com/Shopify/liquid/issues/1390>
    parsed_liquid = Liquid::Template.parse(sanitized_content.to_str, liquid_tag_options)

    html = markdown.render(parsed_liquid.render)
  rescue Liquid::SyntaxError => e
    html = e.message
  end

  parse_html(html)
end
[30] pry(#<MarkdownProcessor::Parser>):1> 

```

Note, we render twice, once before interpreting liquid tags, `render(escaped_content)`, and once after `render(parsed_liquid.render).`

{% hint style="info" %}
Show method does a few things under the hood - it uses `method(:finalize).source_location` from the ruby instance to find where the code for this selector lives, in my case it returns a pair of file and line number `["forem/app/services/markdown_processor/parser.rb", 18]`. Pry then grabs the content from the file on disk and shows the defining block. If you want to confirm this, just modify the source file after loading the class, you'll be seeing the on disk representation of the code, not necessarily the method body that's running right now. That's something to be aware of if you have a long running console session and have been editing these files. This also has limitations - it's unlikely to work for methods that are implemented in compiled C extensions (we'll get to that in a moment) and it may give less than stellar results if the method was defined with metaprogramming (by `define_method`, for example).
{% endhint %}

We pass a source and user to new ([https://github.com/forem/forem/blob/main/app/services/markdown\_processor/parser.rb#L12](https://github.com/forem/forem/blob/main/app/services/markdown\_processor/parser.rb#L12)) and since preview didn't pass the link\_attributes keyword arg, we'll use a default empty hash. `@content`, `@source,` and `@user` are set (I've cd'd into the existing parser), so we can call finalize at will.

Here's the method we're looking at (the result is rendered html, and has the unwanted content we're investigating):

```ruby
    def finalize(link_attributes: {})
      options = { hard_wrap: true, filter_html: false, link_attributes: link_attributes }
      renderer = Redcarpet::Render::HTMLRouge.new(options)
      markdown = Redcarpet::Markdown.new(renderer, Constants::Redcarpet::CONFIG)
      catch_xss_attempts(@content)
      code_tag_content = convert_code_tags_to_triple_backticks(@content)
      escaped_content = escape_liquid_tags_in_codeblock(code_tag_content)
      html = markdown.render(escaped_content)
      sanitized_content = sanitize_rendered_markdown(html)
      begin
        liquid_tag_options = { source: @source, user: @user }

        # NOTE: [@rhymes] liquid 5.0.0 does not support ActiveSupport::SafeBuffer,
        # a String substitute, hence we force the conversion before passing it to Liquid::Template.
        # See <https://github.com/Shopify/liquid/issues/1390>
        parsed_liquid = Liquid::Template.parse(sanitized_content.to_str, liquid_tag_options)

        html = markdown.render(parsed_liquid.render)
      rescue Liquid::SyntaxError => e
        html = e.message
      end

      parse_html(html)
    end
```

So we encounter a few new co-conspirators in our crime story, Redcarpet::Markdown and [Redcarpet::Render::HTMLRouge](https://github.com/forem/forem/blob/main/app/lib/redcarpet/render/html\_rouge.rb), Redcarpet comes from a gem, the Rouge code syntax highlighting gem has a plugin module for Redcarpet that's included in the custom class (Forem's code, linked) as well as noticing we have a [config constant](https://github.com/forem/forem/blob/main/app/lib/constants/redcarpet.rb) defined for Redcarpet options. Like the preview code, we see a few "pre-wash" methods called to sanitize or transform input to the state we need before rendering, and the main-even here is `markdown.render(escaped_content)`. We'll be passing the renderer to the Markdown instance, presumable it will use the renderer to render, and will concern itself with parsing rather than presentation. Let's just assume those two always travel together, and we can see we set hard wrap true, filter html false, and no link attributes in the renderer. The renderer appears to mainly be concerned with rules for showing links, formatting headings so they have linkable anchors,  and formatting code blocks to permit highlighting using a capitalized hint language like "C" or "Ruby" rather than "c" or "ruby".  It includes Rouge and inherits from [Redcarpet::Render::HTML](https://rubydoc.info/gems/redcarpet/Redcarpet/Render/HTML) - which is where initialization is defined (we call new and pass it args, but don't define initialize, so we're only extending behavior). It's almost correct to call this a plain HTML object (we could swap out the renderer to experiment, but that's a bit further off course than I'm going here).

Since the input `@content` string is so short, I'll skip over the `catch_xss_attempts`, `convert_code_tags_to_triple_backticks` and `escape_liquid_tags_in_codeblock` methods, which leave the content unchanged. It's safe, for this limited situation, to assume this holds, and jump into the render method.

```ruby
escaped_content == @content
```

This means `html = markdown.render(escaped_content)` is the thing to look at.

```ruby
      options = { hard_wrap: true, filter_html: false, link_attributes: link_attributes }
      renderer = Redcarpet::Render::HTMLRouge.new(options)
      markdown = Redcarpet::Markdown.new(renderer, Constants::Redcarpet::CONFIG)
      html = markdown.render(escaped_content)

```

Now is about the time I do a little searching, and I come up with this great article from Vaidehi on how to use Redcarpet [http://vaidehijoshi.github.io/blog/2015/08/11/rolling-out-the-redcarpet-for-rendering-markdown/](http://vaidehijoshi.github.io/blog/2015/08/11/rolling-out-the-redcarpet-for-rendering-markdown/) - which almost looks like it was a dry run for writing this code. There's also a "cheat sheet" [https://www.lukeko.com/11/using-rouge-with-redcarpet](https://www.lukeko.com/11/using-rouge-with-redcarpet) here - most of this is good background for what we're looking at and whether we're doing it right, but ultimately doesn't help get closer to _our_ mystery - which is "why do we malform comment embeds"?

#### Markdown#render

It's about to get a little hairy.&#x20;

First pass (markdown render escaped content) just does link->link expansion - given an unadorned link, it generates an A tag with the link target as the content)

```ruby
 markdown.render(@content)
=> "<p>{% embed <a href=\"http://localhost:3000/torpdarlene/comment/c\">http://localhost:3000/torpdarlene/comment/c</a> %}</p>\n"
```

`sanitize_rendered_markdown` does nothing important here, since there's nothing to sanitize, so build up the liquid template parser, and render it (basically, change the embed to the rendered comment template in the views/ directory)

```ruby
[19] pry(#<MarkdownProcessor::Parser>):1> Liquid::Template.parse(@content).render
  Comment Load (1.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."id_code" = $1 LIMIT $2  [["id_code", "c"], ["LIMIT", 1]]
  User Load (0.9ms)  SELECT "users".* FROM "users" WHERE "users"."id" = $1 LIMIT $2  [["id", 4], ["LIMIT", 1]]
  Rendered comments/_comment_date.erb (Duration: 0.7ms | Allocations: 184)
  Rendered comments/_liquid.html.erb (Duration: 5.0ms | Allocations: 1097)
=> "<!-- BEGIN app/views/comments/_liquid.html.erb --><div class=\"liquid-comment\">\n    <div class=\"details\">\n      <a href=\"/torpdarlene\">\n        <img class=\"profile-pic\" src=\"/uploads/user/profile_image/4/96272310-dc87-4f6a-a5d5-b937810e99f3.png\" alt=\"torpdarlene profile image\" />\n      </a>\n      <a href=\"/torpdarlene\">\n        <span class=\"comment-username\">Darlene Torp</span>\n      </a>\n      <span class=\"color-base-30 px-2 m:pl-0\" role=\"presentation\">&bull;</span>\n\n<a href=\"http://localhost:3000/torpdarlene/comment/c\" class=\"comment-date crayons-link crayons-link--secondary fs-s\">\n  <time datetime=\"2022-01-11T20:32:35Z\">\n    Jan 11\n  </time>\n\n</a>\n\n    </div>\n    <div class=\"body\">\n      <p>Keffiyeh portland locavore lomo readymade waistcoat. Tofu pork belly farm-to-table. Schlitz paleo letterpress 8-bit helvetica austin cardigan gluten-free.</p>\n\n\n    </div>\n</div>\n<!-- END app/views/comments/_liquid.html.erb -->"
```

We see the console output that the comment, it's author/user were loaded from the db, and two templates were rendered (the date partial is embedded in the comment partial).

This stage's output looks right - after substituting the embed we get a liquid comment div, more or less what we expect from the liquid parser. At this point, nothing has been changed that we didn't want.

This is the input to the second pass of render:

```ruby
markdown.render parsed_liquid.render
=> "<p><!-- BEGIN app/views/comments/_liquid.html.erb --><div class=\"liquid-comment\">\n    <div class=\"details\">\n      <a href=\"/torpdarlene\">\n        <img class=\"profile-pic\" src=\"/uploads/user/profile_image/4/96272310-dc87-4f6a-a5d5-b937810e99f3.png\" alt=\"torpdarlene profile image\" />\n      </a>\n      <a href=\"/torpdarlene\">\n        <span class=\"comment-username\">Darlene Torp</span>\n      </a>\n      <span class=\"color-base-30 px-2 m:pl-0\" role=\"presentation\">&bull;</span>\n\n<a href=\"http://localhost:3000/torpdarlene/comment/c\" class=\"comment-date crayons-link crayons-link--secondary fs-s\">\n  <time datetime=\"2022-01-11T20:32:35Z\">\n    Jan 11\n  </time>\n\n</a>\n\n    </div>\n    <div class=\"body\">\n      <p>Keffiyeh portland locavore lomo readymade waistcoat. Tofu pork belly farm-to-table. Schlitz paleo letterpress 8-bit helvetica austin cardigan gluten-free.</p>\n<div class=\"highlight\"><pre class=\"highlight plaintext\"><code>&lt;/div&gt;\n</code></pre></div>\n<p></div><br>\n&lt;!-- END app/views/comments/_liquid.html.erb --&gt;</p></p>\n"
```

So we're at the last line of _our_ code before we change what we think we want into what we don't want...

#### Redcarpet::Markdown#render

The Markdown class is defined in [lib/redcarpet.rb](https://github.com/vmg/redcarpet/blob/master/lib/redcarpet.rb#L7) - as stub to add an attr\_reader for the renderer. Most of the functionality is defined in the extension

```ruby
[3] pry(#<MarkdownProcessor::Parser>)> show-method markdown

From: /home/djuber/src/forem/vendor/cache/ruby/3.0.0/gems/redcarpet-3.5.1/lib/redcarpet.rb:7
Class name: Redcarpet::Markdown
Number of lines: 3

class Markdown
  attr_reader :renderer
end
[4] pry(#<MarkdownProcessor::Parser>)> cd markdown
[5] pry(#<Redcarpet::Markdown>):1> show-source
You're inside an object, whose class is defined by means of the C Ruby API.
Pry cannot display the information for this class.
However, you can view monkey-patches applied to this class.
.Just execute the same command with the '--all' switch.
```

So we're at a limit of what ruby introspection can give us. But, where does _render_ get defined? While we ponder that we can try to build a smaller test case. Since the problem behavior is "we get a code block when we didn't want one" or "we left the html processing too early", we can try to make a string that should give all html, but instead gives a mix of code and html.

I'm of course cheating _a little_ here, I know what the end result will be so I can isolate this a little better. In markdown, lines that start with four spaces indentation (or more) are assumed to be code snippets (this is in distinction to the other convention  of fenced code blocks using \`\`\` or \~\~\~ characters to start code blocks).&#x20;

```ruby
[27] pry(#<Redcarpet::Markdown>):1> render "<p>Text</p><p>Text 2</p>"
=> "<p>Text</p><p>Text 2</p>\n"
[28] pry(#<Redcarpet::Markdown>):1> render "<p>Text</p>\n\n<p>Text 2</p>"
=> "<p>Text</p>\n\n<p>Text 2</p>\n"

# a space before the html puts us in "paragraph mode" - wrapping the line in a <p> tag
[29] pry(#<Redcarpet::Markdown>):1> render "<p>Text</p>\n\n <p>Text 2</p>"
=> "<p>Text</p>\n\n<p><p>Text 2</p></p>\n"
# four spaces at the beginning of a line put us in code tag mode:
[30] pry(#<Redcarpet::Markdown>):1> render "<p>Text</p>\n\n    <p>Text 2</p>"
=> "<p>Text</p>\n<div class=\"highlight\"><pre class=\"highlight plaintext\"><code>&lt;p&gt;Text 2&lt;/p&gt;\n</code></pre></div>"

# four spaces _between_ tags, but not in line-initial position, is fine:
[31] pry(#<Redcarpet::Markdown>):1> render "<p>Text</p>    <p>Text 2</p>"
=> "<p>Text</p>    <p>Text 2</p>\n"
```

So we're looking for a situation where what _might_ be html gets interpreted instead as a code block (and sanitized for display).

#### Redcarpet internals

So, where is the class defined? Let's go to the source in the [rc\_markdown.c](https://github.com/vmg/redcarpet/blob/master/ext/redcarpet/rc\_markdown.c#L24-L27) file, we see VALUE (ruby object's C represenation, everything C passes back or receives from ruby will be of type VALUE).

```c

VALUE rb_mRedcarpet;
VALUE rb_cMarkdown;
VALUE rb_cRenderHTML_TOC;

extern VALUE rb_cRenderBase;
```

The convention here is `_` is a :: scope resolution, a c prefix marks a class, an m prefix marks a module... The markdown extensions (in our Constants::Redcarpet::CONFIG hash) are a map of symbol keys to booleans, the code loads those in `rb_redcarpet_md_flags`

```c
	unsigned int extensions = 0;

	Check_Type(hash, T_HASH);

	/**
	 * Markdown extensions -- all disabled by default
	 */
	if (rb_hash_lookup(hash, CSTR2SYM("no_intra_emphasis")) == Qtrue)
		extensions |= MKDEXT_NO_INTRA_EMPHASIS;

	if (rb_hash_lookup(hash, CSTR2SYM("tables")) == Qtrue)
		extensions |= MKDEXT_TABLES;

```

the `rb_redcarpet_md__new` method returns a VALUE (an instance of Redcarpet::Markdown, or a subclass?), and wraps the underlying [Sundown](https://github.com/vmg/sundown)  library that redcarpet builds on, you see names like `sd_markdown` which grew out of the original library (same author). `rc_markdown.c` has [`rb_redcarpet_md_render`](https://github.com/vmg/redcarpet/blob/master/ext/redcarpet/rc\_markdown.c#L132-L171) which is the ruby facing `Redcarpet::Markdown#render` method I was looking for.&#x20;

This looks like normal glue code (hint, this is not the interesting part, it's the bridge between ruby and the C guts where the parsing actually happens), there's some type coercion, precondition checks (input text was a `T_STRING` __ value), pulls the `@renderer` instance variable into the `rb_rndr` variable, sets up the output buffer (where are we sending the data, in our case a string like object to be returned later), then, following the comment "render the magic" we call `sd_markdown_render` (the actual work starts here), and then we'll build a return value from the output buffer by encoding as a ruby string type, release the allocated output buffer, check if the renderer responds to "postprocess", pass the pre-return string `text` to `rb_render.postprocess(text)` (the `rb_funcall` line)  and finally answer `text` back to the caller (in ruby). If you've written C extensions in Ruby, or Python, very little should be surprising in this file, and `sd_markdown_render` is our _real_ entry point.

#### sd\_markdown\_render

This was lifted from the prior "sundown" markdown library.

{% embed url="https://github.com/vmg/redcarpet/blob/master/ext/redcarpet/markdown.c#L2827-L2929" %}

The important call (most of this is scanning the document for special cases, references, headers and footnotes) is `parse_block(ob, md, text->data, text->size)` which has the dispatch table to determine what to do with the input. The "return" value from this void function is parsed markdown in the ob buffer.

#### parse\_block

This is the dispatch table, so it importantly includes the tests to determine which case we're in

{% embed url="https://github.com/vmg/redcarpet/blob/master/ext/redcarpet/markdown.c#L2448-L2507" %}

The function acts as a large loop, handling the next processable unit of text, until the input has been fully consumed. Each of the dispatched functions will advance the offset pointer `beg` - which marks the beginning of the unprocessed text.&#x20;

Per pass, in order, one of these will be performed:

* `is_atxheader` - true when the line starts with some number (1-6) of `#` characters - it recognizes from h1 to h6, followed by a space (so seven hash marks in a row should not be marked as `<h6>#</h6>`, but would fall through to the next case.
* Line starts with a '<' character, an html rendering callback exists, conditionally `parse_html`, treating as success if an html block was parsed.
* `is_empty` - if the line is empty, skip it.
* `is_hrule` - may consist of up to three initial spaces, then either \* or - or \_ characters, all the way to the end of the line, at least three (but four or more are recognized).
* attempt to parse a fenced code block in `parse_fencecode` like parse\_htmlblock, treating as success if one was found and handled
* if `prefix_quote`, call `parse_blockquote` - prefix quote checks up to three space characters, followed by a '>' character, followed by a space.&#x20;
* if `prefix_code`, `parse_blockcode` prefix code checks if there are 4 spaces (or more) at the beginning of the line. This expansion is controlled by setting the `MKDEXT_DISABLE_INDENTED_CODE` flag in the options.
* if `prefix_uli`, then `parse_list` (is this an unorderd list item (ul)?)
* if `prefix_oli`, then parse\_list, setting `MKD_LIST_ORDERED` option (was this an ordered list item?)
* otherwise, this is just plain text, call `parse_paragraph`



