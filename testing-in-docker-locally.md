# Testing in Docker locally

I'm trying to use the container images in docker-compose.yml to run rspec.

I modified the docker-compose file to include a container named "rspec" and can run specs via the following

```text
docker-compose run rspec bundle exec rspec spec/.../file_spec.rb

```

This seems like it works, _except_ carrierwave uploads in the factories \(on any object with an associated image, like a user or a badge or an organization\). I'm focusing on the behavior of the badge factory as a specific concrete example but this is presenting more generally for all carrierwave uploaders, in  the docker hosted spec executions.

In all cases the setup includes this breakpoint added to spec/factories/badges.rb

```ruby
FactoryBot.define do
  image_path = Rails.root.join("spec/support/fixtures/images/image1.jpeg")

  factory :badge do
    # stopping here to inspect behavior
    binding.pry
    sequence(:title) { |n| "#{Faker::Book.title}-#{n}" }
    description { Faker::Lorem.sentence }
    badge_image { Rack::Test::UploadedFile.new(image_path, "image/jpeg") }
  end
end

```

I can replicate the same steps in rspec running locally \(on the host, not in docker\) and see the following \(correct\) behavior, yielding a passing test:

```ruby

djuber@forem:~/src/forem$ bundle exec rspec spec/uploaders/badge_uploader_spec.rb:29
[1] pry(#<FactoryBot::Declaration::Implicit>)> image_path = Rails.root.join("spec/support/fixtures/images/image1.jpeg")
=> #<Pathname:/home/djuber/src/forem/spec/support/fixtures/images/image1.jpeg>

[2] pry(#<FactoryBot::Declaration::Implicit>)> file = Rack::Test::UploadedFile.new(image_path, "image/jpeg")
=> #<Rack::Test::UploadedFile:0x00005589a9b2ae70
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210325-77708-7oti7y.jpeg>>

[4] pry(#<FactoryBot::Declaration::Implicit>)> b = Badge.new(title: :f, description: :f, badge_image: file)
=> #<Badge:0x00005589aa78b908
 id: nil,
 badge_image: nil,
 created_at: nil,
 credits_awarded: 0,
 description: "f",
 slug: nil,
 title: "f",
 updated_at: nil>

[5] pry(#<FactoryBot::Declaration::Implicit>)> b.valid?
=> true                                                

[6] pry(#<FactoryBot::Declaration::Implicit>)> b
=> #<Badge:0x00005589aa78b908                   
 id: nil,
 badge_image: nil,
 created_at: nil,
 credits_awarded: 0,
 description: "f",
 slug: "f",
 title: "f",
 updated_at: nil>

[7] pry(#<FactoryBot::Declaration::Implicit>)> b.valid?
=> true                                                

[8] pry(#<FactoryBot::Declaration::Implicit>)> b.errors
=> #<ActiveModel::Errors:0x00005589aa95b828            
 @base=
  #<Badge:0x00005589aa78b908
   id: nil,
   badge_image: nil,
   created_at: nil,
   credits_awarded: 0,
   description: "f",
   slug: "f",
   title: "f",
   updated_at: nil>,
 @details={},
 @messages={}>

[9] pry(#<FactoryBot::Declaration::Implicit>)> b.badge_image
=> #<BadgeUploader:0x00005589aa78afa8                       
 @cache_id="1616689706-170175881901995-0001-3524",
 @cache_storage=
  #<CarrierWave::Storage::File:0x00005589aa788050
   @cache_called=nil,
   @uploader=#<BadgeUploader:0x00005589aa78afa8 ...>>,
 @file=
  #<CarrierWave::SanitizedFile:0x00005589aa789e50
   @content=nil,
   @content_type="image/jpeg",
   @file="/home/djuber/src/forem/public/uploads/tmp/1616689706-170175881901995-0001-3524/image1.jpeg",
   @original_filename="image1.jpeg">,
 @filename="image1.jpeg",
 @identifier=nil,
 @model=
  #<Badge:0x00005589aa78b908
   id: nil,
   badge_image: nil,
   created_at: nil,
   credits_awarded: 0,
   description: "f",
   slug: "f",
   title: "f",
   updated_at: nil>,
 @mounted_as=:badge_image,
 @original_filename="image1.jpeg",
 @staged=true,
 @versions={},
 @versions_to_cache=nil,
@versions_to_store=nil>


[10] pry(#<FactoryBot::Declaration::Implicit>)> b
=> #<Badge:0x00005589aa78b908                    
 id: nil,
 badge_image: nil,
 created_at: nil,
 credits_awarded: 0,
 description: "f",
 slug: "f",
 title: "f",
 updated_at: nil>

[11] pry(#<FactoryBot::Declaration::Implicit>)> b.save
=> true                                               

[12] pry(#<FactoryBot::Declaration::Implicit>)> b
=> #<Badge:0x00005589aa78b908                    
 id: 2,
 badge_image: "image1.jpeg",
 created_at: Thu, 25 Mar 2021 16:30:10 UTC +00:00,
 credits_awarded: 0,
 description: "f",
 slug: "f",
 title: "f",
 updated_at: Thu, 25 Mar 2021 16:30:10 UTC +00:00>

[13] pry(#<FactoryBot::Declaration::Implicit>)> b.badge_image
=> #<BadgeUploader:0x00005589aa78afa8                        
 @cache_id=nil,
 @cache_storage=
  #<CarrierWave::Storage::File:0x00005589aa788050
   @cache_called=nil,
   @uploader=#<BadgeUploader:0x00005589aa78afa8 ...>>,
 @file=
  #<CarrierWave::SanitizedFile:0x00005589ac9d2a98
   @content=nil,
   @content_type="image/jpeg",
   @file="/home/djuber/src/forem/public/uploads/badge/badge_image/2/image1.jpeg",
   @original_filename=nil>,
 @filename="image1.jpeg",
 @identifier=nil,
 @model=
  #<Badge:0x00005589aa78b908
   id: 2,
   badge_image: "image1.jpeg",
   created_at: Thu, 25 Mar 2021 16:30:10 UTC +00:00,
   credits_awarded: 0,
   description: "f",
   slug: "f",
   title: "f",
   updated_at: Thu, 25 Mar 2021 16:30:10 UTC +00:00>,
 @mounted_as=:badge_image,
 @original_filename="image1.jpeg",
 @staged=false,
 @storage=
  #<CarrierWave::Storage::File:0x00005589aa530320
   @cache_called=nil,
   @uploader=#<BadgeUploader:0x00005589aa78afa8 ...>>,
 @versions={},
 @versions_to_cache=nil,
 @versions_to_store=nil>

```

Notably - the badge\_image is only present on the model after save \(it's added in memory to the object when I save\) - but the uploader is created with a file object that's a carrier wave sanitized file.



In docker prompts 1, 2, 4 \(I removed 3 which was a typo and error response\) are the same, but valid? returns false, errors points out the image is blank, calling that "unsupported type".

```ruby

[1] pry(#<FactoryBot::Declaration::Implicit>)> image_path = Rails.root.join("spec/support/fixtures/images/image1.jpeg")                                                              
=> #<Pathname:/opt/apps/forem/spec/support/fixtures/images/image1.jpeg>

[2] pry(#<FactoryBot::Declaration::Implicit>)> file = Rack::Test::UploadedFile.new(image_path, "image/jpeg")                                                                                            
=> #<Rack::Test::UploadedFile:0x00007f0c419db5d0
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210325-21-p4c7is.jpeg>>
 
 
[9] pry(#<FactoryBot::Declaration::Implicit>)> b = Badge.new(title: :f, description: :f, badge_image: file)                                                                                                    
=> #<Badge:0x00007f0c41c354c0
 id: nil,
 title: "f",
 slug: nil,
 description: "f",
 badge_image: nil,
 created_at: nil,
 updated_at: nil>
 
[10] pry(#<FactoryBot::Declaration::Implicit>)> b.valid?
=> false                      
                         
[11] pry(#<FactoryBot::Declaration::Implicit>)> b.errors.messages
=> {:badge_image=>["Image has an unsupported type", "can't be blank"]}

[21] pry(#<FactoryBot::Declaration::Implicit>)> b.badge_image
=> #<BadgeUploader:0x00007f0c401bbea0                        
 @cache_id=nil,
 @file=nil,
 @filename=nil,
 @identifier=nil,
 @model=
  #<Badge:0x00007f0c41c354c0
   id: nil,
   title: "f",
   slug: "f",
   description: "f",
   badge_image: nil,
   created_at: nil,
   updated_at: nil>,
 @mounted_as=:badge_image,
 @staged=false,
 @versions=nil,
 @versions_to_cache=nil,
 @versions_to_store=nil>
 
[22] pry(#<FactoryBot::Declaration::Implicit>)> file
=> #<Rack::Test::UploadedFile:0x00007f0c419db5d0    
 @content_type="image/jpeg",
 @original_filename="image1.jpeg",
 @tempfile=#<File:/tmp/image120210325-21-p4c7is.jpeg>>

[25] pry(#<FactoryBot::Declaration::Implicit>)> File.exist?(file.tempfile)
=> true                                               


```

the tempfile exists. The badge\_image is a badge uploader. In test, carrierwave disables processing \(is that the problem? why didn't that also happen in local testing\). 

Confused a litte here. Carrierwave mounted uploaders don't provide a lot of good surface area on the model classes to track this down.

