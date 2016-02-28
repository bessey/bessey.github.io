---
layout: post
title: "Deploying Sunpot / Solr with Capistrano"
date: 2013-03-13 23:24
comments: true
categories: ruby rails solr
---

At [SNGTRKR](http://sngtrkr.com) we needed something to make fuzzy searching in two models and across multiple fields easy. In other words we needed an indexer. This was my first time setting up any indexing engine, and my research led me to [Apache Solr](http://lucene.apache.org/solr/). Part of the reason for choosing Solr over any other solution was its tight Rails integration thanks to the fantastic [Sunspot](http://sunspot.github.com/) gem. Not only does Sunspot have a super easy API to connect your Rails app to your Solr service, but it also has a ready packaged Solr engine that is easily spun up and down with a rake task.

We use Capistrano for deployment, so making sure we could get Sunspot to automate itself during deployment was critical. To that end, here's what I needed to add to my `deploy.rb`. This is an amalgamation of two or three scripts I've seen online, none of them had quite them same setup as me, so I thought it would be worth adding one to the pool.

<!-- more -->

``` ruby deploy.rb

namespace :solr do
  desc "start solr"
  task :start, :roles => :app, :except => { :no_release => true } do 
    run "cd #{current_path} && RAILS_ENV=#{rails_env} bundle exec rake sunspot:solr:start"
  end

  desc "stop solr"
  task :stop, :roles => :app, :except => { :no_release => true } do 
    run "cd #{current_path} && RAILS_ENV=#{rails_env} bundle exec rake sunspot:solr:stop"
  end

  desc "reindex the whole database"
  task :reindex, :roles => :app do
    stop
    run "rm -rf #{shared_path}/solr/data/*"
    start
    puts "You need to run this yourself now:"
    puts "cd #{current_path} && RAILS_ENV=#{rails_env} bundle exec rake sunspot:solr:reindex"
  end

  desc "Symlink in-progress deployment to a shared Solr index"
  task :symlink, :except => { :no_release => true } do
    run "ln -s #{shared_path}/solr/data/ #{release_path}/solr/data"
    run "ln -s #{shared_path}/solr/pids/ #{release_path}/solr/pids"
  end
end
 
after "deploy:update_code", "solr:symlink"

```

You'll notice that the reindex task is a little weird in that it doesn't do what it should! I ran into the problem that Capistrano does not send keystrokes through to the remote machine, and Solr asks for a confirmation when reindexing, leaving you unable to "confirm". There is a [pull request to fix this](https://github.com/sunspot/sunspot/pull/370) in the [Sunspot repo](https://github.com/sunspot/sunspot) so do have a look there. At time of writing it hasn't been merged, and I haven't tested it, so I can't confirm it's solved. In the meantime as you can see, my 'solution' has been to print out the command that Capistrano *would* run, and SSH into my production server and run it myself. Not ideal, but you shouldn't be doing a full reindex to often anyways.

The main point of these Capistrano tasks is to ensure that `/path_to_my_app/solr/data` folder is symlinked to the shared folder, and thus not overwritten on each deploy.

Lastly, here's my `sunspot.yml` config. I don't think it's anything special, but just in case it's of any use to anyone, here she is:

``` yaml sunspot.yml

development:
  solr:
    hostname: localhost
    port: 8980
    log_level: INFO
  auto_commit_after_delete_request: true

test:
  solr:
    hostname: localhost
    port: 8981
    log_level: OFF

production:
  solr:
    hostname: localhost
    port: 8983
    log_level: WARNING
  auto_commit_after_request: true  

``` 

I hope this post is of some help to someone out there! And equally, if you see anything that makes you go "Sweet jesus, this guy has no idea what he's doing", do tell me, because you're almost certainly right!

Peace,

Matt.