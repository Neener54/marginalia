#!/usr/bin/env rake
#-*-ruby-*-
# Add your own tasks in files placed in lib/tasks ending in .rake,
# for example lib/tasks/capistrano.rake, and they will automatically be available to Rake.

require File.expand_path('../config/application', __FILE__)
require 'aws/s3'
require 'net/https'
require 'uri'
require 'netrc'

Ideas::Application.load_tasks

task :prod do
  ENV['GIT_REMOTE_NAME'] = 'marginalia'
  ENV['HEROKU_APP_NAME'] = 'marginalia-io'
end

task :stage do
  ENV['GIT_REMOTE_NAME'] = 'heroku'
  ENV['HEROKU_APP_NAME'] = 'bugsplat-notes'
end

task :ensure_clean_tree do
  abort("Tree dirty!") unless `git diff-files`.strip.length == 0
end

task :ensure_commits do
  if `git log --pretty=oneline #{ENV['GIT_REMOTE_NAME']}/master..`.strip.length == 0
    abort("No commits to push!")
  end
end

task :deploy => [:ensure_commits, :ensure_clean_tree, :test] do
  sh "git push origin master"
  sh "git push github master"
  sh "git push #{ENV['GIT_REMOTE_NAME']} master"
  sh "heroku run bundle exec rake db:migrate --app #{ENV['HEROKU_APP_NAME']}"
  sh "heroku ps:restart web --app #{ENV['HEROKU_APP_NAME']}"
end

task :console do
  sh "heroku run rails console --app #{ENV['HEROKU_APP_NAME']}"
end

task :clean_guest_users => :environment do
  users = User.where("email like 'guest%@example.com' and created_at < now() - '1 days'::interval")
  users.each do |user|
    user.notes.each do |note|
      note.destroy
    end
    user.user_emails.each do |email|
      email.destroy
    end
    user.destroy
  end
end

task :export, [:user_id] => :environment do |t, args|
  job = ExportJob.new(args[:user_id])
  job.perform
end

task :clean_exports => :environment do
  AWS::S3::Base.establish_connection!(
    :access_key_id     => ENV['AWS_ACCESS_KEY_ID'],
    :secret_access_key => ENV['AWS_SECRET_ACCESS_KEY']
  )

  Export.where("updated_at < now() - '4 hours'::interval").each do |export|
    next unless export.filename
    AWS::S3::S3Object.delete export.filename, ENV['AWS_EXPORT_BUCKET']
    export.destroy
  end
end

task :refresh_stripe do
  Dir.chdir "#{ENV['HOME']}/financials" do
    netrc = Netrc.read
    uri = URI('https://www.marginalia.io/__stripe')

    creds = netrc[uri.hostname]

    req = Net::HTTP::Get.new(uri.request_uri)

    req.basic_auth creds[0], creds[1]

    res = Net::HTTP.start(uri.hostname, 443, :use_ssl => true) do |http|
      http.request(req)
    end

    puts res.body
  end
end
