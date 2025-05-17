# frozen_string_literal: true

source "https://rubygems.org"
gem "jekyll", "~> 4.3"  # 明确指定版本
gem "csv"

platforms :ruby do
  gem "sass-embedded", "~> 1.58.0", "< 1.59", platform: "x86_64-linux"  # 明确版本和平台[<sup data-citation='{&quot;url&quot;:&quot;https://stackoverflow.com/questions/75452016/installation-messed-up-with-ruby-unable-to-install-jekyll&quot;,&quot;title&quot;:&quot;Installation messed up with Ruby: Unable to install jekyll&quot;,&quot;content&quot;:&quot;The last version of sass-embedded (~&gt; 1.54) to support your Ruby &amp; RubyGems was 1.58.0. Try installing it with `gem install sass-embedded -v 1.58.0` and then running the current command again&quot;}'>2</sup>](https://stackoverflow.com/questions/75452016/installation-messed-up-with-ruby-unable-to-install-jekyll)[<sup data-citation='{&quot;url&quot;:&quot;https://github.com/jekyll/jekyll/issues/9563&quot;,&quot;title&quot;:&quot;[Bug]: Can&apos;t serve website due to sass-embedded version #9563 - GitHub&quot;,&quot;content&quot;:&quot;So, because sass-embedded ~ &gt; 1.54 could not be found in locally installed gems for any resolution platforms (x86_64-linux) and Gemfile depends on jekyll ~ &gt; 4.3.3, version solving has failed. The sou&quot;}'>8</sup>](https://github.com/jekyll/jekyll/issues/9563)[<sup data-citation='{&quot;url&quot;:&quot;https://stackoverflow.com/questions/72331753/ruby-and-rails-github-action-exit-code-16&quot;,&quot;title&quot;:&quot;Ruby and Rails Github Action exit code 16 - Stack Overflow&quot;,&quot;content&quot;:&quot;I am trying to set up a continuous integration workflow with Github actions for a new Rails project. This is the error: 2022-05-21T17:07:01.1242737Z Your bundle only supports platforms [\&quot;x86_64-darwin&quot;}'>14</sup>](https://stackoverflow.com/questions/72331753/ruby-and-rails-github-action-exit-code-16)
end

gem "jekyll-theme-chirpy", "~> 5.3", ">= 5.3.2"

group :test do
  gem "html-proofer", "~> 3.18"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :install_if => Gem.win_platform?

# Jekyll <= 4.2.0 compatibility with Ruby 3.0
gem "webrick", "~> 1.7"
