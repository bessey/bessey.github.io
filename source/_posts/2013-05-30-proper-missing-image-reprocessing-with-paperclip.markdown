---
layout: post
title: "Proper missing image reprocessing with Paperclip + Rails"
date: 2013-05-30 16:23
comments: true
categories: ruby rails paperclip
---

As [SNGTRKR](http://sngtrkr.com) is still in its infancy, we find ourselves tweaking the image sizes our models have on a semi-regular basis. Fortunately [Paperclip](https://github.com/thoughtbot/paperclip) makes that a doddle; just a one line change to the model definition and away we go.

Now all thats left to address is the missing copies of this new image size in production. We have well over 10,000 records with large images associated with them in the database, so the naïve approach of reprocessing every model isn't going to suffice here. In theory though we still have a weapon in our arsenal; Paperclip provides the `paperclip:refresh:missing_styles` rake task, which will look for a YAML file in your project that stores the previously generated styles.

Unfortunately, that's as far as that file goes. It does not store per record granularity, so if that rake should bail at some point through your 10,000+ records, well, start again! Haha, *screw that*. This actual happened to us, for reasons I am still not 100% sure on, but I can only assume it was an S3 connection issue or similar. To solve it, I wrote my own method to do roughly the same as what Paperclip does, but in a slightly more intelligent way; we check whether a potentially missing image exists for each record of a given ActiveRecord model (this is a Rails-centric solution) by actually trying to open the URL the image would be at.
<!-- more -->

Not only can this be stopped and effectively resumed (that is it will never reprocess an image that has already been processed), but if an error is raised during the reprocessing step, it will entirely delete the image from the record, as we assume there has been some kind of corruption between the record and its S3 files.

``` ruby admin_tools.rb

class AdminTools

	# For reprocessing Paperclip images, when certain styles are missing.
	# Had to do it myself because their rake seems to crash for unknown reasons.
	# This also has the advantage of checking if the file exists rather than just assuming all don't.
	# We assume if the large image is missing, we need to reprocess this item.
	# AdminTools.reprocess_missing_images Class, :image_size
	def self.reprocess_missing_images klass, missing_style
		klass.find_each do |inst|
			rep = false
			puts "START #{inst.id}"
			unless inst.image.present?
				next
			end
			image = inst.image(missing_style)
			begin
				actual = open(image)
			rescue
				rep = true
			end
			if actual.is_a?(StringIO) or !rep
				puts "SKIP #{inst.id}"
				next
			end

			puts "REPROCESS #{inst.id}"

			begin
				inst.image.reprocess!(missing_style)
			rescue
				# We assume that for images that fail the record is in some way corrupt, 
				# you may not want to do this.
				inst.image.destroy 
			end
		end
	end

end

```

Feel free to use it, and if you make any improvements, please leave a comment!