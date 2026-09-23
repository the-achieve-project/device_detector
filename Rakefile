# frozen_string_literal: true

require 'rake'
require 'rake/testtask'

$LOAD_PATH.unshift 'lib'
require 'device_detector'

Rake::TestTask.new do |t|
  t.pattern = 'spec/**/*_spec.rb'
  t.libs.push 'spec'
end

task default: :test

desc 'generate detectable names output for README'
task :detectable_names do
  require 'date'

  bot_names = DeviceDetector::Bot.new('').send(:regexes)
                                 .map { |r| r[:name] }.uniq.sort_by(&:downcase)
  bot_names.delete('$1')
  client_names = DeviceDetector::Client.new('').send(:regexes)
                                       .map { |r| r[:name] }.uniq.sort_by(&:downcase)
  client_names.delete('$1')
  device = DeviceDetector::Device.new('')
  device_paths = device.send(:filepaths)
  device_regexes = device.send(:load_regexes, device_paths)
  device_names = device_regexes.flat_map { |dn| dn[1].keys }.uniq.sort_by(&:downcase)

  today = Date.today.strftime

  puts '## Detectable clients, bots and devices'
  puts
  puts "Updated on #{today}"
  puts
  puts '### Bots'
  puts
  puts bot_names.join(', ')
  puts
  puts '### Clients'
  puts
  puts client_names.join(', ')
  puts
  puts '### Devices'
  puts
  puts device_names.join(', ')
  puts
end

PIWIK_REPO_URL = 'https://github.com/matomo-org/device-detector'
PIWIK_CHECKOUT_LOCATION = ENV.fetch('MATOMO_CHECKOUT', '/tmp/matomo_device_detector')

def get_latest_piwik_checkout
  if File.exist?(PIWIK_CHECKOUT_LOCATION)
    system "cd #{PIWIK_CHECKOUT_LOCATION}; git reset --hard HEAD; git pull origin master"
  else
    system "git clone --depth 1 #{PIWIK_REPO_URL} #{PIWIK_CHECKOUT_LOCATION}"
  end
end

desc 'update regex database from piwik project'
task :update_regexes do
  top = File.expand_path(__dir__)
  get_latest_piwik_checkout
  system "cp -R #{PIWIK_CHECKOUT_LOCATION}/regexes/* #{top}/regexes"
end

desc 'update fixtures from piwik project'
task :update_fixtures do
  top = File.expand_path(__dir__)
  get_latest_piwik_checkout

  fixture_mappings = [
    { target_path: "#{top}/spec/fixtures/detector", source_path: 'Tests/fixtures/*.yml' },
    { target_path: "#{top}/spec/fixtures/client",
      source_path: 'Tests/Parser/Client/fixtures/*.yml' },
    { target_path: "#{top}/spec/fixtures/parser", source_path: 'Tests/Parser/fixtures/*.yml' },
    { target_path: "#{top}/spec/fixtures/device",
      source_path: 'Tests/Parser/Device/fixtures/*.yml' }
  ]

  fixture_mappings.each do |mapping|
    source_path = mapping.fetch(:source_path)
    target_path = mapping.fetch(:target_path)
    system "cp -R #{PIWIK_CHECKOUT_LOCATION}/#{source_path} #{target_path}"
  end
end

# Matomo keeps the OS tables in PHP, not YAML, so update_regexes does not carry
# them. Rewrites the matching constants in lib/device_detector/os.rb.
desc 'update OS name, family, desktop and version-mapping tables from piwik project'
task :update_os_tables do
  top = File.expand_path(__dir__)
  get_latest_piwik_checkout
  php = File.read("#{PIWIK_CHECKOUT_LOCATION}/Parser/OperatingSystem.php")
  commit = `git -C #{PIWIK_CHECKOUT_LOCATION} rev-parse HEAD`.strip
  source = "https://github.com/matomo-org/device-detector/blob/#{commit}/Parser/OperatingSystem.php"

  php_array = lambda do |name|
    php[/\$#{name} = \[(.*?)\n    \];/m, 1] or raise "#{name} not found in OperatingSystem.php"
  end
  php_strings = ->(body) { body.scan(/'((?:[^'\\]|\\.)*)'/).flatten.map { |s| s.gsub("\\'", "'") } }
  php_pairs = ->(body) { php_strings.call(body).each_slice(2).to_a }
  quote = ->(s) { "'#{s.gsub("'", "\\\\'")}'" }

  hash_constant = lambda do |const, pairs|
    lines = pairs.map { |k, v| "      #{quote.call(k)} => #{quote.call(v)}" }
    "    #{const} = {\n#{lines.join(",\n")}\n    }.freeze"
  end

  families = php_array.call('osFamilies').scan(/'([^']+)'\s*=>\s*\[(.*?)\]/m).map do |family, codes|
    rows = php_strings.call(codes).each_slice(10).map { |row| "        #{row.join(' ')}" }
    "      #{quote.call(family)} => %w[\n#{rows.join("\n")}\n      ]"
  end
  desktop = php_strings.call(php_array.call('desktopOsArray')).map { |s| quote.call(s) }

  replacements = {
    'OPERATING_SYSTEMS' => hash_constant.call('OPERATING_SYSTEMS', php_pairs.call(php_array.call('operatingSystems'))),
    'OS_FAMILIES' => "    OS_FAMILIES = {\n#{families.join(",\n")}\n    }.freeze",
    'FIRE_OS_VERSION_MAPPING' => hash_constant.call('FIRE_OS_VERSION_MAPPING', php_pairs.call(php_array.call('fireOsVersionMapping'))),
    'LINEAGE_OS_VERSION_MAPPING' => hash_constant.call('LINEAGE_OS_VERSION_MAPPING', php_pairs.call(php_array.call('lineageOsVersionMapping'))),
    'DESKTOP_OSS' => "    DESKTOP_OSS = Set.new(\n      [\n        #{desktop.join(', ')}\n      ]\n    )"
  }

  path = "#{top}/lib/device_detector/os.rb"
  ruby = File.read(path)
  replacements.each do |const, code|
    pattern = /    # https:\/\/github\.com\/matomo-org[^\n]*\n(?:    #[^\n]*\n|\n)*    #{const} = (?:\{.*?\n    \}\.freeze|Set\.new\(.*?\n    \))/m
    ruby.sub!(pattern) { "    # #{source}\n#{code}" } or raise "#{const} not found in os.rb"
  end
  File.write(path, ruby)
end
