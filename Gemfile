source "https://rubygems.org"

# Jekyll 版本
gem "jekyll", "~> 4.2.2"

# Hydejack 主题（开源免费版，GPL-3.0）
gem "jekyll-theme-hydejack", "~> 9.1"

# Hydejack 依赖插件
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-paginate"
  gem "jekyll-include-cache"
  gem "jekyll-default-layout"
  gem "jekyll-optional-front-matter"
  gem "jekyll-readme-index"
  gem "jekyll-redirect-from"
  gem "jekyll-relative-links"
  gem "jekyll-titles-from-headings"
  gem "jekyll-last-modified-at"
  gem "jekyll-admin"
end

# Windows / JRuby 不含 zoneinfo，需要时引入
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Windows 下监听文件变更加速
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# JRuby 专用锁定（保留）
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Ruby 3 下 jekyll serve 必需
gem "webrick", "~> 1.9"
