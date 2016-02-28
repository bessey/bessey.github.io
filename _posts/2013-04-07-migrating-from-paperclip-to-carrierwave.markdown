---
layout: post
title: "Migrating from Paperclip to Carrierwave"
date: 2013-04-07 21:25
comments: true
categories: ruby rails carrierwave paperclip
---

After experiencing some problems configuring [Paperclip](https://github.com/thoughtbot/paperclip) to behave the way we wanted with S3, I decided to move [SNGTRKR](http://www.sngtrkr.com) to the [Carrierwave](https://github.com/jnicklas/carrierwave) gem. I had feared this would be a horrible breaking change, but in reality it wasn't too bad at all. Nevertheless there were some undocumented or poorly documented differences that I thought I'd cover here.

The vast majority of what you need to know is covered in the Carrierwave documentation, specifically the [migrating from paperclip](https://github.com/jnicklas/carrierwave#migrating-from-paperclip) section. In summary, you include a mixin that changes the URL syntax in your Carrierwave uploader to be equivalent to Paperclip's, allowing you to copy paste your Paperclip path into Carrierwave without problem.

The one bit of advice that was not clear to be till delving into the Carrierwave docs was this: the first argument to the `mount_uploader` argument dictates the column name in which the filename of your upload is expected to be held. <!-- more -->For example:

{% highlight ruby %} # app/models/artist.rb
  mount_uploader :image, ArtistUploader

{% endhighlight %}

Will check `@artist.image` for text. Now if you migrated from paperclip, and accessed your images previously through `@artist.image`, you will *not* have an `image` field in your `artists` table. What you will have, containing that exact information, is an `image_file_name` field.

So all that needs to be done, is a column rename! Or if that's not an option and you just want to test out Carrierwave, you could also change `mount_uploader :image, ...` to `mount_uploader :image_file_name, ...`. My migration was as follows:

{% highlight ruby %} # migration_to_carrierwave.rb
class PaperclipToCarrierwave < ActiveRecord::Migration
  def change
  	rename_column :artists, :image_file_name, :image
  	rename_column :releases, :image_file_name, :image
  end
end
{% endhighlight %}

Note the use of `change` to allow for two way migrations should we switch back for whatever reason.