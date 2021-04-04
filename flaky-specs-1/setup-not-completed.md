# Setup not completed

Saw this - could not reproduce locally? 

```text
I, [2021-04-03T15:20:46.672103 #8560]  INFO -- : [knapsack_pro] bundle exec rspec --format progress  --default-path spec "spec/system/articles/user_fixes_a_draft_spec.rb" "spec/requests/comments_create_spec.rb" "spec/system/user_views_a_reading_list_spec.rb" "spec/services/podcasts/feed_spec.rb" "spec/system/user/user_edits_extensions_spec.rb" "spec/requests/registrations_spec.rb" "spec/requests/chat_channels_spec.rb" "spec/requests/reactions_spec.rb" "spec/system/dashboards/user_sorts_dashboard_articles_spec.rb" "spec/system/articles/user_visits_articles_by_tag_spec.rb" "spec/system/user/user_self_destroy_spec.rb" "spec/requests/api/v0/analytics_spec.rb" "spec/requests/articles/articles_update_spec.rb" "spec/system/organization/user_views_an_organization_spec.rb" "spec/services/search/base_spec.rb"
[Zonebie] Setting timezone: ZONEBIE_TZ="Bratislava"
1st Try error in ./spec/system/articles/user_fixes_a_draft_spec.rb:12:
Unable to find link "Click to edit"
RSpec::Retry: 2nd try ./spec/system/articles/user_fixes_a_draft_spec.rb:12
F...............................................................................................................................................................................................................................................

Failures:
  1) let's the user fix a broken draft without publishing
     Failure/Error: expect(page).to have_content("Unpublished Post")
       expected to find text "Unpublished Post" in "Setup not completed yet, missing suggested tags and suggested users. Please visit the configuration page.\nDEV(local)\nWrite a new post\nEdit\nPreview\nClose the editor\nWhoops, something went wrong:\ntitle: can't be blank\nAdd a cover image\nUpload image\n{% codepen https://codepen.io/user/pen/abcdefg default-tab=result %}\nEditor Basics\nUse Markdown to write and format posts.\nCommonly used syntax\n\n\n\n\n\n\n\n\nYou can use Liquid tags to add rich content such as Tweets, YouTube videos, etc.\nIn addition to images for the post's content, you can also drag and drop a cover image\nPublish\nSave draft\nPost options\nRevert new changes"


```

