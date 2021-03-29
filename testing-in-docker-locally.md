# Testing in Docker locally

I'm trying to use the container images in docker-compose.yml to run rspec.

I modified the docker-compose file to include a container named "rspec" and can run specs via the following

```bash
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



### Part 2 - after lunch

So I decided to watch what carrier wave was doing \(on the file system at least\). I noticed the BaseUploader contains an upload directory in "uploads/{model.class.to\_s.underscore}/{mounted\_as}/{model.id}" - i.e. I would expect this to show as

`uploads/badge/badge_image/?` where the example has what looks like an object\_id but might not work as well when id is nil \(because the model has yet to be persisted?\). I did not see this \(I see it write to /tmp and unlink the written file, which might be the same problem under a different guise\). I'm not seeing stat or the like trying to find a directory like that.

### Next day

Could a read-only volume do this \(can write to /tmp successfully and read from /opt/ but not write? That's a simple test at least.

```text
$ cat scripts/write_file.sh 
#!/usr/bin/env bash

set -e

echo "foo" > file

$ docker-compose run rspec scripts/write_file.sh 
...
2021/03/29 13:53:19 Command finished successfully.

$ cat file
foo
```

Okay - so the source is mounted read/write and that's obviously not the problem, it seems more likely this is something related to carrierwave's configuration \(and I would bet that the config files are only expected/tested to work in local dev environments for test\). My next step is to compare what the setup looks like for development \(where I think this works running in docker?\) I guess I can try uploading an image to confirm before I chase down the configs.



Yes, that works - files are created in public/uploads/user/profile\_image/NN/XXXXXXXXXXXX.png, for example public/uploads/user/profile\_image/11/a5ee6ced-0dce-4ab3-9da6-09ea1b1f8265.png

This is in dev mode, and works correctly.

What's happening in test that's different \(I see a lot of organization uploads in this directory that _could_ have been tied to uploading files?, the timestamps are consistent with hammering the test suite\).

There sure are a lot of android-icon-36x36.png files in public/uploads/tmp which are all the same file \(distinct paths, look like epoch time prefixed paths, and all the DEV icon file \(they hash the same\). Something drops lots of garbage in here. All the organization/profile\_image/ files are similarly identical \(which makes sense if they're coming from a test case reading the same file repeatedly\).

#### Clean everything up

I removed everything except the public/uploads/ directory \(all children have been cleaned\) - need to sudo since this is a root created upload \(thanks again, docker\).

```text
$ find public/uploads/
public/uploads/
$ docker-compose run rspec

Randomized with seed 4677
[Zonebie] Setting timezone: ZONEBIE_TZ="Amsterdam"
FFFFFFFFFF...FFFF..FF.....F.....FFF...FFFFFFFF.....FFFFFFFFFFFFFFFFFF......F.FFFFFFFFFFFF.FFFFFFF.........FF...FF......................................................F...F.........FFFFFFF............FFFFFF.............FFFFFF......FF..FF........FFF................................FFFF......FF...........FFFFFFFFFFF...F.FFFFFF.....FFFFFFFFFFFFFFF....FF.....F......FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF*FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF...FFFF.FFFFFFFFFFFFF.........FFFFFFFFFFFFFFF............FF.FFFF....FFFFFFFFFFFFFFFFFFFF......F.........FFFF.......F.FFFFFFFF.....FFFFFFFFFFFFFFFF.F..........FFFF..FF.................FF.....FFFF...........FFFFFFFFFFFF..FFFFFF.........F.........FFFF.FFF***F.FFFFFFFFFFFFFFFFFFFFFF.FFFFFFFFFFFFFFFF..............FFFFFFFFFFFFFFFFFFFFFFFFFF..F.FFFFFFFF.FF.FFFF.FFF...FFF.F....F......FF....F......F.....*FFFF......FF.FFFFFFFF...FFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.......F......FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.F..F...FFFFFFFFFFFFFFFFFFFFFFFFF.....FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.FF.FF...FFFFFFFFFFFFFFFFFFF..................F..F..FFFFFFFFFFFFFFFF.F.F...FFFFFFFFFFFFFFFFFF.FFF......FF..FFFFFF...FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.......F.F.F.FFFFFFFFFFFFF..F..FFF.F.FFF.FFFFFFFFF.FFFF.........FFFFFFFFFFFFFFF........................FFFFFFF.FFF.FF.F.FFFFFFFFFFFFFFFFFFFFFFFFF.F...FFFFFFFFF..F.FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.FFFFFFFFFFF....F..F..FF..FF.F........FFF.....FF...F.FFFFF.F.F..F.F.FFFFFFFFFFFFFFFF..FFFFFFFFFFFFFF..FFFFFF.......FFFFFFFFFFFFFFFFFFFFFFFFF...................FFFFFFFFFFFF....FFFFFFFFF.FFFFF.......FFFFFFFFFFFFFFFF..............FFF..FFFFFFFFFFFFFFF....FFFFFFFFFF.F..FF...FF.FFFF.....FFFFFF....F.F.......FFFFFFFFFFFFFFFFFFFFFFFFFFFFFF..................FFFFFFFFF*F....F......F.FFFFFFFFFFFFFFFFFF..FFFFFF.FFFF.FFFFFFFFFFFFFFFFFFFFFF.FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF....FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF..FFFFFFFFFFFFFFFFFFF..FFFFFFFFFFFFFF...F.........FFFF.FFFFFFFFFFFFFFFFFFFFFF....FFFFFFFFF.FFFFFFFFFFFFFFFFFF*FFFFFFFFFFFFF..FFFFFFFFFFF.............FFFFF...................................F.......................FFFFFFFFFF......FFFFFFFFFFFF.FFFFF.........FFF.F.F.....FF.F....FFFFFFFFFFF.....FFFFFFFFFFFFFF...FF............FFFFFFFFFFFFFFFFFF....FFFFFFFFF.FFFFF............F..FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF...F..FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.FFFFF.FFFFFF.FFFFF.FFF......FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF.FFFFFFFF............FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF...FFFFFFF.FF...FF.FFFFF..FFFF.FFFFFFFF.FFFFFFFFF.FFFFFFFFFFFFF.FF.FFFFFFFFFFFFFFFFFFFFFFFFFFFFF.F.FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

And while the tests are running I see the organization profile images being created - so this is either part of the seed or part of the test execution - in any case - it does _not_ look like this is any problem with specs writing to the filesystem when stubbing an upload \(I'll revisit my local testing in pry and confirm a new file is being created by the test case and _not_ by the seeds only\).

One interesting thing I noted is that while there were over 100 organization/profile\_image files, there were only 2 for users? This might be a seed vs test execution difference.

ProfileImageUploader spec uses `fixture_file_upload` which comes from ActionDispatch::Testing as a wrapper for Rack::Test::UploadedFile - part 1 - change the users factory to create one of these instead of a Rack::Test::UploadedFile 



```text
 /opt/apps/forem/spec/services/suggester/users/recent_spec.rb:29 :
 
    25: 
    26:   context "without cached_followed_tags" do
    27:     it "returns recent_commenters and recent top producers" do
    28:       binding.pry
 => 29:       productive_user = create(:user, comments_count: 1, articles_count: 1)
    30:       unproductive_user = create(:user, comments_count: 0, articles_count: 0)
    31: 
    32:       suggested_users = suggester.suggest
    33:       expect(suggested_users.size).to eq(1)
    34:       expect(suggested_users.map(&:id)).to include(productive_user.id)
    
    
[1] pry(#<RSpec::ExampleGroups::SuggesterUsersRecent::WithoutCachedFollowedTags>)> productive_user = create(:user, comments_count: 1, articles_count: 1)      
ActiveRecord::RecordInvalid: Validation failed: Profile image Image has an unsupported type
from /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activerecord-6.0.3.5/lib/active_record/validations.rb:80:in `raise_validation_error'
```

Definitely just putting "profile\_image" on a user model with a Rack::Test::Upladed.file is not causing files to be created.

The CarrierWave object has `base_path = nil` \(should that be present? can check in dev\) and `tmp_path = "/opt/apps/forem/tmp"` which seems totally reasonable

```ruby
[3] pry(#<RSpec::ExampleGroups::SuggesterUsersRecent::WithoutCachedFollowedTags>)> cd CarrierWave                                                                                     
[4] pry(CarrierWave):1> ls
constants:                
  ActiveRecord   DownloadError     Mount            SanitizedFile        UploadError
  BombShelter    FormNotMultipart  Mounter          Storage              Utilities  
  CacheCounter   IntegrityError    ProcessingError  Test                 Validations
  Compatibility  InvalidParameter  Railtie          UnknownStorageError  VERSION    
  Downloader     MiniMagick        RMagick          Uploader             Vips       
CarrierWave.methods: 
  base_path   clean_cached_files!  generate_cache_id  root=     tmp_path=
  base_path=  configure            root               tmp_path
instance variables: @base_path  @root
locals: _  __  _dir_  _ex_  _file_  _in_  _out_  pry_instance
[5] pry(CarrierWave):1> base_path
=> nil                           
[6] pry(CarrierWave):1> tmp_path
=> "/opt/apps/forem/tmp"        
[7] pry(CarrierWave):1> root
=> "/opt/apps/forem/public"  
```

BaseUploader defines `store_dir` as `uploads/model/mounted_as/model.id` which appears to be happening fine here... 

One thing that's not being set in the initializer is `enable_processing` \(false in test\) and `asset_host` \(nil in test because neither production? nor imgproxy\_enabled? are true\)

All of this works fine in travis which should be the same environment more or less - except rails server container is not being started... and this was _not_ the hangup in the prior tests to get buildkite integrated by molly. I'm going to back off this direction \(using the docker-compose file and forem images as a base case\) and approach this from a more typical direction - which is a ruby:2.7.2 image, a postgres and redis and es images, and run rspec like we do locally. Testing forem in forem's containers is a goal - it might not be the right one for right now.



