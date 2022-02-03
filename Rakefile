# encoding: utf-8
require 'json'

################################
# Rake configuration
################################

# Paths
DERIVED_DATA_DIR = File.join('.build').freeze
RELEASES_ROOT_DIR = File.join('releases').freeze

EXECUTABLE_NAME = 'XCRemoteCache'
EXECUTABLE_NAMES = ['xclibtool', 'xcpostbuild', 'xcprebuild', 'xcprepare', 'xcswiftc', 'xcld'] 
PROJECT_NAME = 'XCRemoteCache'

SWIFTLINT_ENABLED = true
SWIFTFORMAT_ENABLED = true

################################
# Tasks
################################

task :prepare do
  Dir.mkdir(DERIVED_DATA_DIR) unless File.exists?(DERIVED_DATA_DIR)
end

desc 'lint'
task :lint => [:prepare] do
  puts 'Run linting'

  system("swiftformat --lint --config .swiftformat --cache ignore .") or abort "swiftformat failure" if SWIFTFORMAT_ENABLED
  system("swiftlint lint --config .swiftlint.yml --strict") or abort "swiftlint failure" if SWIFTLINT_ENABLED
end

task :autocorrect => [:prepare]  do 
  puts 'Run autocorrect'

  system("swiftformat --config .swiftformat --cache ignore .") or abort "swiftformat failure" if SWIFTFORMAT_ENABLED
  system("swiftlint autocorrect --config .swiftlint.yml") or abort "swiftlint failure" if SWIFTLINT_ENABLED
end

desc 'build package artifacts'
task :build, [:configuration, :arch, :sdks, :is_archive] do |task, args|
  # Set task defaults
  args.with_defaults(:configuration => 'debug', :sdks => ['macos'])

  unless args.configuration == 'Debug'.downcase || args.configuration == 'Release'.downcase
    fail("Unsupported configuration. Valid values: ['Debug', 'Release']. Found '#{args.configuration}''")
  end

  # Clean data generated by SPM
  # FIXME: dangerous recursive rm
  system("rm -rf #{DERIVED_DATA_DIR} > /dev/null 2>&1")

  # Build
  build_paths = []
  args.sdks.each do |sdk|
    spm_build(args.configuration, args.arch)

    # Path of the executable looks like: `.build/(debug|release)/XCRemoteCache`
    build_path_base = File.join(DERIVED_DATA_DIR, args.configuration)
    sdk_build_paths = EXECUTABLE_NAMES.map {|e| File.join(build_path_base, e)}

    build_paths.push(sdk_build_paths)
  end

  puts "Build products: #{build_paths}"

  if args.configuration == 'Release'.downcase
    puts "Creating release zip"
    create_release_zip(build_paths[0])
  end
end

desc 'run tests with SPM'
task :test do
  # Running tests
  spm_test()
end

desc 'run E2E tests with CocoaPods plugin'
task :e2e => [:build, :e2e_only]

desc 'run E2E tests without building the XCRemoteCache binary'
task :e2e_only do
  # Build a plugin
  cocoapods_dir = 'cocoapods-plugin'
  Dir.chdir(cocoapods_dir) do
    gemspec_path = "cocoapods-xcremotecache.gemspec"
    gemfile_path = "cocoapods-xcremotecache.gem"
    system("gem build #{gemspec_path} -o #{gemfile_path}")
    system("gem install #{gemfile_path}")
  end

  # Start nginx server
  system('nginx -c $PWD/e2eTests/nginx/nginx.conf')

  current_branch = 'e2e-test-branch'
  producer_configuration = %{xcremotecache({
    'cache_addresses' => ['http://localhost:8080/cache/pods'], 
    'primary_repo' => '.',
    'primary_branch' => '#{current_branch}',
    'mode' => 'producer',
    'final_target' => 'XCRemoteCacheSample',
    'artifact_maximum_age' => 0
  })}
  consumer_configuration = %{xcremotecache({
    'cache_addresses' => ['http://localhost:8080/cache/pods'], 
    'primary_repo' => '.',
    'primary_branch' => '#{current_branch}',
    'mode' => 'consumer',
    'final_target' => 'XCRemoteCacheSample',
    'artifact_maximum_age' => 0
  })}
  # Configure remote 
  system("git checkout -b #{current_branch}")
  system('git remote add self . && git fetch self')

  log_name = "xcodebuild.log"
  # initalize Pods
  for podfile_path in Dir.glob('e2eTests/**/*.Podfile')
    p "****** Scenario: #{podfile_path}"
    # Revert any local changes
    system('git clean -xdf e2eTests/XCRemoteCacheSample')
    # Link prebuild binaries to the Project
    system('ln -s $(pwd)/releases e2eTests/XCRemoteCacheSample/XCRC')


    # Create producer Podfile
    File.open('e2eTests/XCRemoteCacheSample/Podfile', 'w') do |f|
      # Copy podfile
      File.foreach(podfile_path) { |line| f.puts line }

      f.write(producer_configuration)
    end

    Dir.chdir('e2eTests/XCRemoteCacheSample') do
      system('pod install')
      p "Building producer ..."
      system("xcodebuild -workspace 'XCRemoteCacheSample.xcworkspace' -scheme 'XCRemoteCacheSample' -configuration 'Debug' -sdk 'iphonesimulator' -destination 'generic/platform=iOS Simulator' -derivedDataPath ./DerivedData EXCLUDED_ARCHS='arm64 i386' clean build > #{log_name}")
      
      # reset stats
      system('XCRC/xcprepare stats --reset --format json')
      # clean DerivedData
      system('rm -rf ./DerivedData')
    end


    # Create consumer Podfile
    File.open('e2eTests/XCRemoteCacheSample/Podfile', 'w') do |f|
      # Copy podfile
      File.foreach(podfile_path) { |line| f.puts line; p  }

      f.write(consumer_configuration)
    end

    Dir.chdir('e2eTests/XCRemoteCacheSample') do
      system('pod install')
      p "Building consumer ..."
      system("xcodebuild -workspace 'XCRemoteCacheSample.xcworkspace' -scheme 'XCRemoteCacheSample' -configuration 'Debug' -sdk 'iphonesimulator' -destination 'generic/platform=iOS Simulator' -derivedDataPath ./DerivedData2 EXCLUDED_ARCHS='arm64 i386' clean build > #{log_name}")
    
      # clean DerivedData
      system('rm -rf ./DerivedData2')

      # validate 100% hit rate
      stats_json_string = JSON.parse(`XCRC/xcprepare stats --format json`)
      misses = stats_json_string.fetch('miss_count', 0)
      hits = stats_json_string.fetch('hit_count', 0)
      all_targets = misses + hits
      raise "Failure: XCRemoteCache is disabled" if all_targets == 0
      hit_rate = hits * 100 / all_targets
      raise "Failure: Hit rate is only #{hit_rate}%" if misses != 0
      p "hit rate: #{hit_rate}% (#{hits}/#{all_targets})"
    end

    # Destroy a cache
    system('rm -rf /tmp/cache')
  end

  # Revert remote 
  system('git remote remove self')

  # Stop nginx
  system('nginx -s stop')
end


################################
# Helper functions
################################

def spm_build(configuration, arch)
  spm_cmd = "swift build "\
              "-c #{configuration} "\
              "#{arch.nil? ? "" : "--triple #{arch}"} "
  system(spm_cmd) or abort "Build failure"
end

def bash(command)
  system "bash -c \"#{command}\""
end

def spm_test()
  tests_output_file = File.join(DERIVED_DATA_DIR, 'tests.log')
  # Redirect error stream with to a file and pass to the second stream output 
  spm_cmd = "swift test --enable-code-coverage 2> >(tee #{tests_output_file})"
  test_succeeded = bash(spm_cmd)
  
  abort "Test failure" unless test_succeeded
end

def create_release_zip(build_paths)
  release_dir = RELEASES_ROOT_DIR
  
  # Create and move files into the release directory
  mkdir_p release_dir
  build_paths.each {|p|
    cp_r p, release_dir
  }
  
  output_artifact_basename = "#{PROJECT_NAME}.zip"

  Dir.chdir(release_dir) do
    # -X: no extras (uid, gid, file times, ...)
    # -x: exclude .DS_Store
    # -r: recursive
    system("zip -X -x '*.DS_Store' -r #{output_artifact_basename} .") or abort "zip failure"
    # List contents of zip file
    system("unzip -l #{output_artifact_basename}") or abort "unzip failure"
  end
end
