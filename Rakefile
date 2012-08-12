# encoding: utf-8

require 'rubygems'
require 'bundler'
begin
  Bundler.setup(:default, :development)
rescue Bundler::BundlerError => e
  $stderr.puts e.message
  $stderr.puts "Run `bundle install` to install missing gems"
  exit e.status_code
end
require 'rake'

require 'jeweler'
DEVELOPMENT_GROUPS=[:development, :test]
RUNTIME_GROUPS=Bundler.definition.groups - DEVELOPMENT_GROUPS
Jeweler::Tasks.new do |gem|
  # gem is a Gem::Specification... see http://docs.rubygems.org/read/chapter/20 for more options
  gem.name = "lascivious"
  gem.homepage = "https://github.com/cloudability/lascivious"
  gem.license = "MIT"
  gem.summary = %Q{Easy KISSmetrics integration for Rails}
  gem.description = %Q{Easy interface between Rails & Javascript for KISSmetrics}
  gem.email = "support@cloudability.com"
  gem.authors = ["Mat Ellis", "Jon Frisby"]
  gem.extra_rdoc_files = FileList['*.md', 'LICENSE']
  # gem.required_ruby_version = "> 1.9.2"

  # Jeweler wants to manage dependencies for us when there's a Gemfile.
  # We override it so we can skip development dependencies, and so we can
  # do lockdowns on runtime dependencies while letting them float in the
  # Gemfile.
  #
  # This allows us to ensure that using Friston as a gem will behave how
  # we want, while letting us handle updating dependencies gracefully.
  #
  # The lockfile is already used for production deployments, but NOT having
  # it be obeyed in the gemspec meant that we needed to add explicit
  # lockdowns in the Gemfile to avoid having weirdness ensue in GUI.
  #
  # This is probably a not particularly great way of handling this, but it
  # should suffice for now.
  gem.dependencies.clear

  Bundler.load.dependencies_for(*RUNTIME_GROUPS).each do |dep|
    # gem.add_dependency dep.name, *dependency.requirement.as_list
    # dev_resolved = Bundler.definition.specs_for(DEVELOPMENT_GROUPS).select { |spec| spec.name == dep.name }.first
    runtime_resolved = Bundler.definition.specs_for(Bundler.definition.groups - DEVELOPMENT_GROUPS).select { |spec| spec.name == dep.name }.first
    if(!runtime_resolved.nil?)
      gem.add_dependency(dep.name, ">= #{runtime_resolved.version}")
    else
      # In gem mode, we don't want/need dev tools that would be useful for
      # mucking with the Optimizer itself...
      #gem.add_development_dependency(dep.name, "~> #{dev_resolved.version}")
    end
  end

  gem.files.reject! do |fn|
    fn =~ /^.rvmrc/ ||
    fn =~ /^Gemfile.*/ ||
    fn =~ /^Rakefile/ ||
    fn =~ /^VERSION/ ||
    fn =~ /^\.document/ ||
    fn =~ /^\.yardopts/ ||
    fn =~ /^vendor\/cache/
  end
end
Jeweler::RubygemsDotOrgTasks.new

require 'rake/testtask'
Rake::TestTask.new(:test) do |test|
  test.libs << 'lib' << 'test'
  test.pattern = 'test/**/test_*.rb'
  test.verbose = true
end

require 'rcov/rcovtask'
Rcov::RcovTask.new do |test|
  test.libs << 'test'
  test.pattern = 'test/**/test_*.rb'
  test.verbose = true
  test.rcov_opts << '--exclude "gems/*"'
end

task :default => :test

require 'rake/rdoctask'
Rake::RDocTask.new do |rdoc|
  rdoc.rdoc_dir = 'rdoc'
  rdoc.title = "lascivious #{Lascivious::Version::STRING}"
  rdoc.rdoc_files.include('README*')
  rdoc.rdoc_files.include('lib/**/*.rb')
end
