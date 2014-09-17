# http://nokogiri.org/
require 'nokogiri'
require 'open-uri'

Env = 'env'
Artifacts = "artifacts"
Replacements = "replacements"

ProcCpuInfo = "#{Env}/proc/cpuinfo"
ResolvConf = "#{Env}/etc/resolv.conf"

Stage3Marker = "#{Env}/.stage3_marker"
ReplacementsMarker = "#{Env}/.replacements_marker"
LocaleMarker = "#{Env}/.locale_marker"
PackagerMarker = "#{Env}/.packager_marker"

VirtualRootInsideEnv = "/virtual"
VirtualRoot = "#{Env}#{VirtualRootInsideEnv}"

task :default => :shell


# TODO: sort out some sane deps around the env having been constructed

desc "Run a shell"
task :shell => Env do
  sh "chroot #{Env} /bin/bash"
end

desc "Run a command"
task :cmd, [:cmd] => Env do |t, args|
  if args[:cmd].nil?
    puts "Provide a command"
    exit 1
  end
  sh "chroot #{Env} sh -c '#{args[:cmd]}'"
end

desc "Search the package manager"
task :search, [:pattern] do |t, args|
  if args[:search].nil?
    puts "Provide a search pattern"
    exit 1
  end
  sh "chroot #{Env} eix #{args[:pattern]}"
end

desc "Emerge a package to #{VirtualRoot}"
task :emerge, [:package_spec] => VirtualRoot do |t, args|
  if args[:package_spec].nil?
    puts "Provide a package spec"
    exit 1
  end
  sh "chroot #{Env} emerge --ask --quiet --root=#{VirtualRootInsideEnv} #{args[:package_spec]}"
end

desc "Dockerize a package from #{VirtualRoot}"
task :dockerize, [:package] => VirtualRoot do |t, args|
  if args[:package].nil?
    puts "Provide a package name"
    exit 1
  end
  sh "cd #{VirtualRoot} && tar cp . | docker import - #{args[:package]}"
end

desc "Build the environment"
task :build => PackagerMarker

desc "Destroy #{VirtualRoot}"
task :destroy_virtual do
  rm_f VirtualRoot
end

desc "Unmount and remove the environment"
task :destroy_env => Env do
  sh "umount -l #{Env}/proc || true"
  sh "umount -l #{Env}/dev || true"
  sh "umount -l #{Env}/sys || true"
  sh "umount -l #{Env}/run || true"
  sh "rm -rf --one-file-system #{Env}"
end

Releases = 'http://www.mirrorservice.org/sites/distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/'
releases = Nokogiri::HTML(open(Releases))
stage3 = "#{Releases}#{releases.xpath("//a[contains(@href, 'stage3') and substring(@href, string-length(@href) - 3) = '.bz2']/@href")[0]}"
stage3_file = "#{Artifacts}/#{stage3.sub /^.*\//, ""}"

directory Artifacts
file stage3_file => Artifacts do |task|
  sh "wget --continue --output-document=#{task.name} #{stage3}"
end

directory Env

directory VirtualRoot

file Stage3Marker => [Env, stage3_file] do |task|
  sh "cd #{task.prerequisites[0]} && tar xjpf ../#{task.prerequisites[1]} && cd .. && touch #{task.name}"
end
file ReplacementsMarker => Stage3Marker do |task|
  sh "find replacements -type f | while read r; do cp -v $r #{Env}/`echo $r | sed 's,^replacements/,,'`; done && touch #{task.name}"
end
file ProcCpuInfo => "/proc/cpuinfo" do
  sh "mount -t proc proc #{Env}/proc"
  sh "mount --rbind /sys #{Env}/sys"
  sh "mount --rbind /dev #{Env}/dev"
  sh "mount --rbind /run #{Env}/run"
end
file ResolvConf => '/etc/resolv.conf' do |task|
  cp task.prerequisites[0], task.name
end
file LocaleMarker => [ReplacementsMarker, ResolvConf, ProcCpuInfo] do |task|
  sh "chroot #{Env} emerge-webrsync"
  sh "chroot #{Env} locale-gen"
  sh "touch #{task.name}"
end
file PackagerMarker => LocaleMarker do |task|
  sh "chroot #{Env} emerge --quiet mirrorselect eix gentoolkit"
  sh "chroot #{Env} sh -c 'mirrorselect --http --output >> /etc/portage/make.conf'"
  sh "chroot #{Env} eix-sync -q"
  sh "chroot #{Env} emerge --update @world"
  sh "touch #{task.name}"
end
