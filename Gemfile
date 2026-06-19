source 'https://rubygems.org'

# Standalone Jekyll (built via GitHub Actions, not the classic github-pages gem).
gem 'jekyll', '~> 4.4'

# Plugins used by the site.
group :jekyll_plugins do
  gem 'jekyll-sitemap', '~> 1.4'
end

# kramdown 2 split the GFM input parser into its own gem (see _config.yml).
gem 'kramdown-parser-gfm', '~> 1.1'

# Webrick is no longer bundled with Ruby 3.x; needed for `jekyll serve`.
gem 'webrick', '~> 1.8'
