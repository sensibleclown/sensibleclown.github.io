# Gemfile
source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.3"

group :test do
  gem "html-proofer", "~> 5.0"
end

group :jekyll_plugins do
  gem "jekyll-sass-converter", "~> 3.0"
end

# Windows and MacOS
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
  gem "wdm", "~> 0.1.1"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
