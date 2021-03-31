# Webdriver and chrome headless

I encountered a surprising behavior when trying to use chrome headless in a docker container

While I had installed chromium-headless in fedora, webdriver's chrome \#location method was looking for a chrome or chromium binary and could not find one.

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

### 

