---
layout: post
title:  "Supporting hreflang alternate with Middleman in three steps"
date:   2016-09-13 12:39:00 +0100
summary:    Improving SEO by flagging duplicate content with hreflang alternate tags. 
minutes: 5
---

After you've setup a new static site with [middleman and localis(z)ed][middleman] any content you want, you'll want to setup those pesky hreflang tags to ensure you don't get penalised by the search engines for duplicate content.  It will also help with ensuring users get search results with the correct localisation - [read Google's webmasters article here][seo]


## Step 1
First, look for this block of code in `config.rb` located in the root of your middleman site:

~~~ruby
###
# Helpers
###
# Methods defined in the helpers block are available in templates
# helpers do
#   def some_helper
#     "Helping"
#   end
# end
~~~

This is the helpers section, that allows you to define functions to be called within pages.

## Step 2
Modify the previous block of code to include the following function definition:

~~~ruby
###
# Helpers
###
# Methods defined in the helpers block are available in templates
helpers do
	def localized_paths_for(page)
		localized_paths = {}
		(langs).each do |locale|
			# Loop over all pages to find the ones using the same templates (target) for each language
			sitemap.resources.select do |resource|
				if resource.is_a?(Middleman::Sitemap::ProxyResource)
					if(resource.path != page.path)
						if resource.target == page.target && resource.metadata[:options][:locale] == locale
							# is a page that has specific translation
							localized_paths[locale] = resource.url
							break
						end
						if resource.target.gsub(locale.to_s, '').gsub('..','.') == page.target.gsub(page.metadata[:options][:locale].to_s, '').gsub('..','.') && resource.metadata[:options][:locale] == locale
							# is a page with a custom translation, need to match it back to the root localization
							localized_paths[locale] = resource.url
							break
						end
					end
				end
			end
		end
		localized_paths
	end
end 
~~~

This will expose the My Account `localized_paths_for` that takes a  [Middleman::Sitemap::Resource][resourceclass] (Correct as of middleman 4.1.8)

## Step 3
Lastly, add the following section to the `<head>` section of your `layout.erb` file:

~~~html
<head>
 <!-- header tags go here -->
 <!--Hreflang alternate tags get auto-generated here-->
 <% localized_paths_for(current_page).each do |text, path| %>
  <link rel="alternate" hreflang="<%= text %>" href="<%= path %>"/>
 <% end %>
</head>
~~~

Your site will now auto-generate hreflang alternate tags effortlessly.


[middleman]: https://middlemanapp.com/advanced/localization/
[seo]: https://support.google.com/webmasters/answer/189077?hl=en
[resourceclass]: http://www.rubydoc.info/gems/middleman-core/4.1.8/Middleman/Sitemap/Resource