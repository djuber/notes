# Webdriver and chrome headless

I encountered a surprising behavior when trying to use chrome headless in a docker container

While I had installed chromium-headless in fedora, webdriver's chrome \#location method was looking for a chrome or chromium binary and could not find one.

```ruby
  134) Views an article shows all comments
       Failure/Error: driven_by :selenium, using: :chrome, screen_size: [1400, 2000], options: { url: ENV["SELENIUM_URL"] }
       
       Webdrivers::BrowserNotFound:
         Failed to find Chrome binary.
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chrome_finder.rb:21:in `location'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chrome_finder.rb:10:in `version'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:51:in `browser_version'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:145:in `browser_build_version'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:32:in `latest_version'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/common.rb:135:in `correct_binary?'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/common.rb:91:in `update'
       # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:160:in `block in <main>'
```



Just circling back so I capture this - the chromium-headless package includes `/usr/lib64/chromium-browser/headless_shell` but  webdrivers looks in "the usual places" for "the usual names"

```ruby

      def linux_location
        return wsl_location if System.wsl_v1?

        directories = %w[/usr/local/sbin /usr/local/bin /usr/sbin /usr/bin /sbin /bin /snap/bin /opt/google/chrome]
        files = %w[google-chrome chrome chromium chromium-browser]

        directories.each do |dir|
          files.each do |file|
            option = "#{dir}/#{file}"
            return option if File.exist?(option)
          end
        end

        nil
      end

```

Does adding `WD_CHROME_PATH="/usr/lib64/chromium-browser/headless_shell`  magically fix this \(and allow me to not install chromium full to run headless tests\)? 

### [https://fedora.pkgs.org/34/fedora-updates-testing-x86\_64/chromium-headless-89.0.4389.90-3.fc34.x86\_64.rpm.html](https://fedora.pkgs.org/34/fedora-updates-testing-x86_64/chromium-headless-89.0.4389.90-3.fc34.x86_64.rpm.html)

### Files

| Path |
| :--- |
| /usr/lib/.build-id/ |
| /usr/lib/.build-id/87/2be63e1ef0e028cd8072ab28b5d65f06a37db9 |
| /usr/lib64/chromium-browser/headless\_shell |

So the answer appears to be "no" - or the chrome headless shell doesn't have the same behavior as a full chrome binary does - the first place this falls down is with asking for version:

```text

[0401/050419.736754:ERROR:zygote_host_impl_linux.cc(90)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
F[0401/050419.765748:ERROR:zygote_host_impl_linux.cc(90)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
F[0401/050419.796463:ERROR:zygote_host_impl_linux.cc(90)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
F[0401/050419.826276:ERROR:zygote_host_impl_linux.cc(90)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
F

Failures:

  1) Views an article shows all comments
     Failure/Error: driven_by :selenium, using: :chrome, screen_size: [1400, 2000], options: { url: ENV["SELENIUM_URL"] }
     
     RuntimeError:
       Failed to make system call: ["/usr/lib64/chromium-browser/headless_shell", "--product-version"]
     # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/system.rb:190:in `call'
     # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chrome_finder.rb:119:in `linux_version'
     # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chrome_finder.rb:10:in `version'
     # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:51:in `browser_version'
     # /opt/apps/bundle/gems/webdrivers-4.6.0/lib/webdrivers/chromedriver.rb:145:in `browser_build_version'
```



