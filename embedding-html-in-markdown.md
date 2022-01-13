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

So we encounter a few new co-conspirators in our crime story, Redcarpet::Markdown and [Redcarpet::Render::HTMLRouge](https://github.com/forem/forem/blob/main/app/lib/redcarpet/render/html\_rouge.rb), Redcarpet comes from a gem, the Rouge renderer has a plugin module for Redcarpet that's included in the custom class (Forem's code, linked) as well as noticing we have a [config constant](https://github.com/forem/forem/blob/main/app/lib/constants/redcarpet.rb) defined for Redcarpet options. Like the preview code, we see a few "pre-wash" methods called to sanitize or transform input to the state we need before rendering, and the main-even here is `markdown.render(escaped_content)`. We'll be passing the renderer to the Markdown instance, presumable it will use the renderer to render, and will concern itself with parsing rather than presentation. Let's just assume those two always travel together, and we can see we set hard wrap true, filter html false, and no link attributes in the renderer. The renderer appears to mainly be concerned with rules for showing links, formatting headings so they have linkable anchors,  and formatting code blocks to permit highlighting using a capitalized hint language like "C" or "Ruby" rather than "c" or "ruby".  It includes Rouge and inherits from [Redcarpet::Render::HTML](https://rubydoc.info/gems/redcarpet/Redcarpet/Render/HTML) - which is where initialization is defined (we call new and pass it args, but don't define initialize, so we're only extending behavior). It's almost correct to call this a plain HTML object (we could swap out the renderer to experiment, but that's a bit further off course than I'm going here).

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

