#!/usr/bin/env ruby
require 'fileutils'

task :default => [:build]

desc "deletes _site"
task :delete do
  puts "deleting site"
  rm_rf '_site'
end

desc "build site"
task :build => :delete do
  system "jekyll"
end

desc "run server"
task :server => :delete do
  puts "running server"
  system "jekyll --server --auto"
end
