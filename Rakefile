require 'bundler/gem_tasks'
require 'rake/testtask'
require 'fileutils'
require 'pathname'
require 'git'

desc 'Default: run unit tests.'
task :default => :test

desc 'Update misc tables'
task :update do
  Dir.glob('tools/update_*.rb').each do |path|
    ruby path
  end
end

namespace :rbs do
  desc 'Validate RBS type definitions'
  task :validate do
    sh 'bundle exec rbs validate'
  end

  desc 'Run Steep type checker'
  task :check do
    sh 'bundle exec steep check'
  end

  desc 'Run all type checks (RBS validation + Steep)'
  task :all => [:validate, :check]
end

namespace :test do
  desc 'Preparation of external modules'
  task :prepare do
    external_repos = [
      'jpmobile-ipaddresses',
      'jpmobile-terminfo',
    ]
    github_prefix = 'https://github.com/jpmobile'
    vendor_path = Pathname.new(Dir.pwd).join('vendor')
    FileUtils.mkdir_p(vendor_path)

    FileUtils.cd(vendor_path) do
      external_repos.each do |repos|
        unless File.directory?("#{repos}/.git")
          Git.clone("#{github_prefix}/#{repos}.git", repos, { :path => vendor_path })
        end
      end
    end
  end
end

task :test => ['test:prepare', 'spec:unit', 'spec:rack', 'test:rails']
load 'lib/tasks/jpmobile_tasks.rake'

desc 'Run the full test suite with coverage and emit a merged report'
task :coverage do
  ENV['COVERAGE'] = '1'
  Rake::Task['test'].invoke
  Rake::Task['coverage:report'].invoke
end

namespace :coverage do
  # The Rails run (test:rails) exercises a *copy* of jpmobile under
  # test/rails/rails_root/vendor/jpmobile, so its result paths differ from the
  # in-process unit/rack runs. Rewrite those paths back onto the real lib/ tree
  # so SimpleCov merges them as the same files, then collate everything.
  desc 'Merge SimpleCov results from all test runs into a single report'
  task :report do
    require 'simplecov'
    require 'simplecov-lcov'
    require 'simplecov_json_formatter'
    require 'json'

    gem_root = Dir.pwd
    specs_result = File.join(gem_root, 'coverage', '.resultset.json')
    rails_result = File.join(gem_root, 'coverage', 'rails', '.resultset.json')

    result_files = []
    result_files << specs_result if File.exist?(specs_result)

    if File.exist?(rails_result)
      data = JSON.parse(File.read(rails_result))
      data.each_value do |command|
        next unless command.is_a?(Hash) && command['coverage'].is_a?(Hash)

        command['coverage'] = command['coverage'].transform_keys do |path|
          path.sub(%r{.*/rails_root/vendor/jpmobile/}, "#{gem_root}/")
        end
      end
      fixed = File.join(gem_root, 'coverage', 'rails', '.resultset-remapped.json')
      File.write(fixed, JSON.generate(data))
      result_files << fixed
    end

    if result_files.empty?
      abort 'No SimpleCov results found. Run `rake coverage` first.'
    end

    SimpleCov::Formatter::LcovFormatter.config do |c|
      c.report_with_single_file = true
      c.single_report_path = 'coverage/lcov.info'
    end

    SimpleCov.collate(result_files) do
      track_files 'lib/**/*.rb'
      add_filter '/spec/'
      add_filter '/test/'
      add_filter '/vendor/'
      enable_coverage :branch
      formatter SimpleCov::Formatter::MultiFormatter.new(
        [
          SimpleCov::Formatter::JSONFormatter,
          SimpleCov::Formatter::LcovFormatter,
        ],
      )
    end
  end
end
