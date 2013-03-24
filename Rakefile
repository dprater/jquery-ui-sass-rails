require 'json'
require 'bundler/gem_tasks'

# returns the source filename for a given JSON build file
# (e.g., "ui.core.jquery.json" returns "jquery.ui.core.js")
def source_file_for_build_file(build_file)
  "jquery.#{build_file.sub('.jquery.json', '')}.js"
end

# returns the source filename for a named file in the 'dependencies'
# array of a JSON build file
# (e.g., if the JSON build file contains
#
#    "dependencies": {
#      "jquery": ">=1.6",
#      "ui.core": "1.9.2",
#      "ui.widget": "1.9.2"
#    },
#
# then "ui.widget" returns "jquery.ui.widget.js")
#
# The only exception is "jquery", which doesn't follow the
# same naming conventions so its a special case.
def source_file_for_dependency_entry(dep_entry)
  # if the dependent file is jquery.js itself, return its filename
  return "jquery.js" if dep_entry == 'jquery'

  # otherwise, tack 'jquery.' on the front
  "jquery.#{dep_entry}.js"
end

# return a Hash of dependency info, whose keys are jquery-ui
# source files and values are Arrays containing the source files
# they depend on
def map_dependencies
  dependencies = {}
  Dir.glob("jquery-ui/*.jquery.json").each do |build_file|
    build_info = JSON.parse(File.read build_file)
    source_file_name = source_file_for_build_file(File.basename(build_file))

    deps = build_info['dependencies'].keys

    # None of jquery.ui files should depend on jquery.js,
    # so we remove 'jquery' from the list of dependencies for all files
    deps.reject! {|d| d == "jquery" }

    deps.map! {|d| source_file_for_dependency_entry d }

    dependencies[source_file_name] = deps
  end
  dependencies
end

DEPENDENCY_HASH = map_dependencies

def version
  JSON.load(File.read('jquery-ui/package.json'))['version']
end

task :submodule do
  sh 'git submodule update --init' unless File.exist?('jquery-ui/README.md')
end

def get_js_dependencies(basename)
  dependencies = DEPENDENCY_HASH[basename]
  if dependencies.nil?
    puts "Warning: No dependencies found for #{basename}"
    dependencies = []
  end
  # Make sure we do not package assets with broken dependencies
  dependencies.each do |dep|
    unless File.exist?("jquery-ui/ui/#{dep}")
      fail "#{basename}: missing #{dep}"
    end
  end
  dependencies
end

def remove_js_extension(path)
  path.chomp(".js")
end

def protect_copyright_notice(source_code)
  # YUI does not minify comments starting with "/*!"
  # The i18n files start with non-copyright comments, so we require a newline
  # to avoid protecting those
  source_code.gsub!(/\A\s*\/\*\r?\n/, "/*!\n")
end

desc "Remove the app directory"
task :clean do
  rm_rf 'app'
end

desc "Generate the JavaScript assets"
task :javascripts => :submodule do
  target_dir = "app/assets/javascripts"
  mkdir_p target_dir
  Rake.rake_output_message 'Generating javascripts'
  Dir.glob("jquery-ui/ui/*.js").each do |path|
    basename = File.basename(path)
    dep_modules = get_js_dependencies(basename).map(&method(:remove_js_extension))
    File.open("#{target_dir}/#{basename}", "w") do |out|
      dep_modules.each do |mod|
        out.write("//= require #{mod}\n")
      end
      out.write("\n") unless dep_modules.empty?
      source_code = File.read(path)
      source_code.gsub!('@VERSION', version)
      protect_copyright_notice(source_code)
      out.write(source_code)
    end
  end

  # process the i18n files separately for performance, since they will not have dependencies
  # https://github.com/joliss/jquery-ui-rails/issues/9
  Dir.glob("jquery-ui/ui/i18n/*.js").each do |path|
    basename = File.basename(path)
    File.open("#{target_dir}/#{basename}", "w") do |out|
      source_code = File.read(path)
      source_code.gsub!('@VERSION', version)
      protect_copyright_notice(source_code)
      out.write(source_code)
    end
  end

  File.open("#{target_dir}/jquery.ui.effect.all.js", "w") do |out|
    Dir.glob("jquery-ui/ui/jquery.ui.effect*.js").sort.each do |path|
      asset_name = remove_js_extension(File.basename(path))
      out.write("//= require #{asset_name}\n")
    end
  end
  File.open("#{target_dir}/jquery.ui.all.js", "w") do |out|
    Dir.glob("jquery-ui/ui/*.js").sort.each do |path|
      asset_name = remove_js_extension(File.basename(path))
      out.write("//= require #{asset_name}\n")
    end
  end
end

desc "Generate the CSS assets"
task :stylesheets => :submodule do
  target_dir = "app/assets/stylesheets"
  mkdir_p target_dir
  Rake.rake_output_message 'Generating stylesheets'

  css_dir = "jquery-ui/themes/base"
  Dir.glob("#{css_dir}/*.css").each do |path|
    basename = File.basename(path)
    source_code = File.read(path)
    source_code.gsub!('@VERSION', version)
    protect_copyright_notice(source_code)
    extra_dependencies = []
    # Is "theme" listed among the dependencies for the matching JS file?
    unless basename =~ /\.(all|base|core)\./
      dependencies = DEPENDENCY_HASH[basename.sub(/\.css/, '.js')]
      if dependencies.nil?
        puts "Warning: No matching JavaScript dependencies found for #{basename}"
        extra_dependencies << 'jquery.ui.core'
      else
        dependencies.each do |dependency|
          dependency = dependency.sub(/\.js$/, '')
          dependent_stylesheet = "#{dependency}.css"
          extra_dependencies << dependency if File.exists?("#{css_dir}/#{dependent_stylesheet}")
        end
        extra_dependencies << 'jquery.ui.theme'
      end
    end
    extra_dependencies.reverse.each do |dep|
      # Add after first comment block
      source_code = source_code.sub(/\A((.*?\*\/\n)?)/m, "\\1/*\n *= require #{dep}\n */\n")
    end
    # Use "require" instead of @import
    source_code.gsub!(/^@import (.*)$/) { |s|
      m = s.match(/^@import (url\()?"(?<module>[-_.a-zA-Z]+)\.css"\)?;/) \
        or fail "Cannot parse import: #{s}"
      "/*\n *= require #{m['module']}\n */"
    }
    # Be cute: collapse multiple require comment blocks into one
    source_code.gsub!(/^( \*= require .*)\n \*\/(\n+)\/\*\n(?= \*= require )/, '\1\2')
    # Replace hard-coded image URLs with asset path helpers
    source_code.gsub!(/url\("?images\/([-_.a-zA-Z0-9]+)"?\)/, 'url(<%= image_path("jquery-ui/\1") %>)')
    File.open("#{target_dir}/#{basename}.erb", "w") do |out|
      out.write(source_code)
    end
  end
end

desc "Generate the image assets"
task :images => :submodule do
  target_dir = "app/assets/images/jquery-ui"
  mkdir_p target_dir
  Rake.rake_output_message 'Copying images'
  FileUtils.cp(Dir.glob("jquery-ui/themes/base/images/*"), target_dir)
end

desc "Convert stylesheets to SCSS"
task :scss do
  base_dir = 'vendor/assets/stylesheets'
  scss_dir = "#{base_dir}/scss"
  variables_hash = {}

  # create scss directory, if necessary
  Dir.mkdir(scss_dir)  unless File.directory?(scss_dir)

  # convert css files to scss
  Dir.glob("#{base_dir}/*.erb").each do |source_file|
    stylesheet_string = File.read(source_file)

    # replace image_path ERB with image-url Sass
    stylesheet_string.gsub!(/<%= image_path\((\S+)\) %>/) { "image-path(#{$1})" }

    # remove sprockets require directives, because they don't work well with Sass variables
    stylesheet_string.gsub! /^\s*\*=\s*require.+$/, ""

    # extract vars
    regex = /(url\(<%= image_path\([\S]+\) %>\)|[\S]+)\/\*{([a-z]+)}\*\//i
    vars = stylesheet_string.scan regex

    # write variables to gobal hash
    vars.each do |var|
      value = var[0]
      name = var[1]
      variables_hash[name] ||= value
    end

    # write destination file
    destination_file_name = File.basename(source_file).gsub(".css.erb", ".css.scss")
    destination_file = File.open "#{scss_dir}/#{destination_file_name}", 'w'
    if destination_file_name == 'jquery.ui.core.css.scss'
      destination_file << "@import 'jquery.ui.variables';\n"
    end
    destination_file << stylesheet_string.gsub(regex) { "$#{$2}" }
    destination_file.close
  end

  # write _jquery.ui.variables.css.scss
  variables_stylesheet = File.open "#{scss_dir}/_jquery.ui.variables.css.scss", 'w'
  variables_hash.each do |name, value|
    variables_stylesheet << "$#{name}: #{value} !default;\n"
  end
  variables_stylesheet.close
end

desc "Clean and then generate everything (default)"
task :assets => [:clean, :javascripts, :stylesheets, :images]

task :build => :assets

task :default => :assets
