source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 4.0"

# http_parser 使用最新代码，解决路径中有空格导致 bundle install 失败的问题
# 参考：
# 1. https://github.com/tmm1/http_parser.rb/issues/47#issuecomment-544196403
# 2. https://github.com/Huang-Libo/Huang-Libo.github.io#faq
gem 'http_parser.rb', git: "git@github.com:tmm1/http_parser.rb.git"

group :jekyll_plugins do
  # If you have any plugins, put them here!
  # gem "jekyll-xxx", "~> x.y"
end

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
