require 'rubygems'
require 'io/console'
require 'sshkit'
require 'sshkit/dsl'
require 'json'
require 'yaml'

if ENV['debug'] == 'true' then
  require 'byebug'
end

$:.unshift File.dirname(__FILE__)

require 'lib/crypti_netssh'
require 'lib/crypti_kit'
require 'lib/crypti_api'

require 'lib/list'
require 'lib/server_list'
require 'lib/account_list'

require 'lib/key_manager'
require 'lib/account_manager'
require 'lib/dependency_manager'

require 'lib/loading_status'
require 'lib/forging_status'
require 'lib/account_balance'

kit = CryptiKit.new('config.yml')

desc 'List available servers'
task :list_servers do
  run_locally do
    info 'Listing available server(s)...'
    kit.config['servers'].values.each do |server|
      info kit.server_info(server)
    end
    info 'Done.'
  end
end

desc 'Add servers to config'
task :add_servers do
  run_locally do
    info 'Adding server(s)...'
    list = ServerList.new(kit.config)
    list.add_all(ENV['servers'])
    info 'Updating configuration...'
    list.save
    info 'Done.'
  end
  Rake::Task['list_servers'].invoke
end

desc 'Remove servers from config'
task :remove_servers do
  run_locally do
    info 'Removing server(s)...'
    list = ServerList.new(kit.config)
    list.remove_all(ENV['servers'])
    info 'Updating configuration...'
    list.save
    info 'Done.'
  end
  Rake::Task['list_servers'].invoke
end

desc 'Add your public ssh key'
task :add_key do
  kit.servers(ENV['servers']).each do |server|
    run_locally do
      dep = DependencyManager.new(self, kit)
      next unless dep.check_local('ssh', 'ssh-copy-id', 'ssh-keygen')

      manager = KeyManager.new(self, kit)
      unless test 'cat', kit.deploy_key then
        manager.gen_key
      end
      if test 'cat', kit.deploy_key then
        manager.add_key(server)
      end
    end
  end
end

desc 'Log into servers directly'
task :log_into do
  kit.servers(ENV['servers']).each do |server|
    run_locally do
      dep =  DependencyManager.new(self, kit)
      next unless dep.check_local('ssh')

      info "Logging into #{server}..."
      system("ssh #{kit.deploy_user_at_host(server)}")
      info 'Done.'
    end
  end
end

desc 'Install dependencies'
task :install_deps do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'apt-get', 'curl')

    as kit.deploy_user do
      info 'Adding repository...'
      execute 'curl', '-sL', 'https://deb.nodesource.com/setup', '|', 'bash', '-'
      info 'Installing apt dependencies...'
      execute 'apt-get', 'install', '-f', '--yes', kit.apt_dependencies
      info 'Installing npm dependencies...'
      execute 'npm', 'install', '-g', kit.npm_dependencies
      info 'Done.'
    end
  end
end

desc 'Install crypti nodes'
task :install_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever', 'npm', 'wget', 'unzip')

    as kit.deploy_user do
      info 'Stopping all processes...'
      execute 'forever', 'stopall', '||', ':'
      info 'Setting up...'
      execute 'rm', '-rf', kit.deploy_path
      execute 'mkdir', '-p', kit.deploy_path
      within kit.deploy_path do
        info 'Downloading crypti...'
        execute 'wget', kit.app_url
        info 'Installing crypti...'
        execute 'unzip', kit.zip_file
        info 'Cleaning up...'
        execute 'rm', kit.zip_file
      end
      within kit.install_path do
        info 'Installing node modules...'
        execute 'npm', 'install'
        info 'Downloading blockchain...'
        execute 'wget', kit.blockchain_url
        info 'Starting crypti node...'
        execute 'forever', 'start', 'app.js', '||', ':'
        info 'Done.'
      end
    end
  end
end

desc 'Uninstall crypti nodes'
task :uninstall_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever')

    as kit.deploy_user do
      info 'Stopping all processes...'
      execute 'forever', 'stopall', '||', ':'
      info 'Removing crypti...'
      execute 'rm', '-rf', kit.deploy_path
      info 'Done.'
    end
  end
end

desc 'Start crypti nodes'
task :start_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever')

    as kit.deploy_user do
      within kit.install_path do
        info 'Starting crypti node...'
        execute 'forever', 'start', 'app.js', '||', ':'
        info 'Done.'
      end
    end
  end
end

desc 'Restart crypti nodes'
task :restart_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever')

    as kit.deploy_user do
      within kit.install_path do
        info 'Restarting crypti node...'
        execute 'forever', 'restart', 'app.js', '||', ':'
        info 'Done.'
      end
    end
  end
end

desc 'Rebuild crypti nodes (using new blockchain only)'
task :rebuild_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever', 'wget')

    as kit.deploy_user do
      within kit.install_path do
        info 'Stopping all processes...'
        execute 'forever', 'stopall', '||', ':'
        info 'Removing old blockchain...'
        execute 'rm', '-f', 'blockchain.db*'
        info 'Removing old log file...'
        execute 'rm', '-f', 'logs.log'
        info 'Downloading blockchain...'
        execute 'wget', kit.blockchain_url
        info 'Starting crypti node...'
        execute 'forever', 'start', 'app.js', '||', ':'
        info 'Done.'
      end
    end
  end
end

desc 'Stop crypti nodes'
task :stop_nodes do
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'forever')

    as kit.deploy_user do
      within kit.install_path do
        info 'Stopping crypti node...'
        execute 'forever', 'stop', 'app.js', '||', ':'
        info 'Done.'
      end
    end
  end
end

desc 'Start forging on crypti nodes'
task :start_forging do
  puts 'Starting forging...'
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'curl')

    api = CryptiApi.new(self)
    kit.get_passphrase(server) do |passphrase|
      api.post '/forgingApi/startForging', passphrase
      api.post '/api/unlock', passphrase do |json|
        manager = AccountManager.new(self, kit)
        manager.add_account(json, server)
      end
    end
  end
  Rake::Task['get_forging'].invoke
end

desc 'Stop forging on crypti nodes'
task :stop_forging do
  puts 'Stopping forging...'
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'curl')

    api = CryptiApi.new(self)
    kit.get_passphrase(server) do |passphrase|
      api.post '/forgingApi/stopForging', passphrase do |json|
        manager = AccountManager.new(self, kit)
        manager.remove_account(json, server)
      end
    end
  end
  Rake::Task['get_forging'].invoke
end

desc 'Get loading status'
task :get_loading do
  puts 'Getting loading status...'
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'curl')

    api  = CryptiApi.new(self)
    json = api.get '/api/getLoading'
    info kit.server_info(server)
    puts LoadingStatus.new(json)
  end
end

desc 'Get forging status'
task :get_forging do
  puts 'Getting forging status...'
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'curl')

    api  = CryptiApi.new(self)
    json = api.get '/forgingApi/getForgingInfo'
    info kit.server_info(server) + ForgingStatus.new(json).to_s
  end
end

desc 'Get account balances'
task :get_balances do
  puts 'Getting account balances...'
  on kit.servers(ENV['servers']), in: :sequence, wait: kit.server_delay do |server|
    dep = DependencyManager.new(self, kit)
    next unless dep.check_remote(server, 'curl')

    list = AccountList.new(kit.config)
    key  = kit.server_key(server)
    if address = list[key] then
      api  = CryptiApi.new(self)
      json = api.get '/api/getBalance', { address: address }
      info kit.server_info(server) + ": #{address}"
      puts AccountBalance.new(json)
    end
  end
end