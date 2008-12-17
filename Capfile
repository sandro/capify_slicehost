# IP of your slice
SLICE_IP_ADDRESS = #TODO

# you get this from support@slicehost.com, right after you build a slice
INITIAL_SLICEHOST_ROOT_PASSWORD = #TODO

# your personal public key to use to log in to slice without password
# generate with ssh-keygen -t rsa -f slice_id_rsa
# example:  PRIVATE_KEY_PATH = "#{ENV['HOME']}/.ssh/slice_l4rk_id_rsa"
PRIVATE_KEY_PATH = #TODO

PUBLIC_KEY_PATH  = "#{PRIVATE_KEY_PATH}.pub"

# this is the user you'll log in to your slice with day-to-day
ADMIN_USERNAME = #TODO

# as set in configs/ssh_config
SSH_PORT = 23872

role :slice, SLICE_IP_ADDRESS

set :passwords, {}


namespace :slice do
  desc "Basic configuration baseline."
  task :config do
    # note we only log in as root the very first time we set up slice
    # from then on root ssh is disabled
    set :user, 'root'
    set :password, INITIAL_SLICEHOST_ROOT_PASSWORD
  
    change_root_password
    copy_config_files
    config_iptables
    set_up_admin_user
    upgrade_system
    install_extras
  
    run "shutdown -r now"
  
    puts "Your slice is now updated and secured."
    puts "root pass: %s" % passwords['root']
    puts "Admin pass: %s" % passwords['admin']
  end

  task :change_root_password do
    run "aptitude install -y apg"
    generate_password('root')
    run "passwd", :pty => true do |channel, stream, data|
      2.times { channel.send_data passwords['root']+"\n" }
    end
  end

  task :copy_config_files do
    put(File.read("Configs/sshd_config"), "/etc/ssh/sshd_config")
    put(File.read("Configs/dots/bash_profile"), "/etc/skel/.bash_profile")
    put(File.read("Configs/dots/bashrc"), "/etc/skel/.bashrc")
    put(File.read("Configs/dots/vimrc"), "/etc/skel/.vimrc")
    put(File.read("Configs/iptables.up.rules"), "/etc/iptables.up.rules")
  end

  task :config_iptables do
    run "iptables-restore < /etc/iptables.up.rules"
    run "cat /etc/network/interfaces | sed '/iface lo inet loopback/a\pre-up iptables-restore < /etc/iptables.up.rules' > /root/interfaces"
    run "mv /root/interfaces /etc/network/interfaces"
  end

  task :set_up_admin_user do
    ADMIN_USER_HOME = "/home/#{ADMIN_USERNAME}"
    SSH_PUBLIC_KEY_TEMP_PATH = "#{ADMIN_USER_HOME}/sshpub.#{Time.now.to_i}"

    generate_password('admin')
    run "useradd -G www-data -m -s /bin/bash #{ADMIN_USERNAME}"
    run "passwd #{ADMIN_USERNAME}", :pty => true do |channel, stream, data|
      2.times { channel.send_data passwords['admin']+"\n" }
    end
  
    run "echo '#{ADMIN_USERNAME} ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers"
    run "echo 'AllowUsers #{ADMIN_USERNAME}' >> /etc/ssh/sshd_config"
  
    run "mkdir #{ADMIN_USER_HOME}/.ssh"
    put(File.read(PUBLIC_KEY_PATH), "#{SSH_PUBLIC_KEY_TEMP_PATH}", :mode => 0600)
    run "cat #{SSH_PUBLIC_KEY_TEMP_PATH} >> #{ADMIN_USER_HOME}/.ssh/authorized_keys"
    run "rm #{SSH_PUBLIC_KEY_TEMP_PATH}"
    run "chown -R #{ADMIN_USERNAME}:#{ADMIN_USERNAME} #{ADMIN_USER_HOME}/.ssh"
    run "chmod 700 #{ADMIN_USER_HOME}/.ssh"
    run "chmod 600 #{ADMIN_USER_HOME}/.ssh/authorized_keys"
  end

  task :upgrade_system do
    run "locale-gen en_US.UTF-8 && /usr/sbin/update-locale LANG=en_US.UTF-8"
    run "aptitude update"
    run "aptitude -y safe-upgrade"
    run "aptitude -y full-upgrade"
    run "aptitude -y install build-essential"
  end

  desc "installs extra little utilities"
  task :install_extras do
    sudo "aptitude -y install curl"
    sudo "aptitude -y install screen"
    sudo "aptitude -y install subversion"
    sudo "aptitude -y install vim"
  end

  namespace :install do
    set :user, ADMIN_USERNAME
    ssh_options.merge! :port => SSH_PORT, :auth_methods => ['publickey'], :keys => [PRIVATE_KEY_PATH]

    desc "Installs the Rails stack"
    task :rails_stack do
      # mysql
      # postfix # replaces exim which mysql installed with postfix
      # ruby
      # rubygems
      rails
      mongrel
      nginx
  
      puts 'Note:  run dpkg-reconfigure postfix to set up postfix.'
      puts "Mysql pass: %s" % passwords['mysql']
    end

    desc "install mysql"
    task :mysql do
      sudo "aptitude install -y mysql-server mysql-client libmysqlclient15-dev libmysql-ruby1.8" do |channel, stream, data|
        if data =~/^New password/
          generate_password('mysql')
          channel.send_data passwords['mysql']+"\n"
        end
      end
    end

    desc "install postfix"
    task :postfix do
      run "export DEBIAN_FRONTEND=noninteractive"
      sudo "apt-get install -y postfix" do |channel, stream, data|
        if data =~/^General type of mail/
          channel.send_data "1\n"
        end
      end
    end

    desc "install ruby"
    task :ruby do
      sudo "aptitude install -y ruby1.8-dev ruby1.8 ri1.8 rdoc1.8 irb1.8 libreadline-ruby1.8 libruby1.8 libopenssl-ruby"
      sudo "ln -s /usr/bin/ruby1.8 /usr/local/bin/ruby"
      sudo "ln -s /usr/bin/ri1.8 /usr/local/bin/ri"
      sudo "ln -s /usr/bin/rdoc1.8 /usr/local/bin/rdoc"
      sudo "ln -s /usr/bin/irb1.8 /usr/local/bin/irb"
      sudo "aptitude install -y imagemagick librmagick-ruby1.8 librmagick-ruby-doc libfreetype6-dev xml-core -y"
    end
    
    desc "install rubygems from source, since aptitude is stuck at 0.9.4"
    task :rubygems do
      # TODO:  you know, we ought to just have this in a bash script, push that script to slice, and execute it there
      run "if [ ! -d ~/src ]; then mkdir ~/src; fi"
      run "cd ~/src && curl --silent --remote-name http://files.rubyforge.mmmultiworks.com/rubygems/rubygems-1.0.1.tgz"
      run "cd ~/src && tar xzf rubygems-1.0.1.tgz"
      run "cd ~/src/rubygems-1.0.1 && sudo ruby setup.rb"
      run "cd /usr/local/bin && sudo ln -s /usr/bin/gem1.8 gem"
    end

    desc "install rails"
    task :rails do
      sudo "gem install rails"
    end

    desc "install mongrel"
    task :mongrel do
      sudo "gem install mongrel_cluster"
      sudo "useradd -G www-data mongrel"
      sudo "ln -s /usr/bin/ruby1.8 /usr/bin/ruby"
    end

    desc "install nginx"
    task :nginx do
      # sudo "aptitude install -y libpcre3 libpcre3-dev libpcrecpp0 libssl-dev zlib1g-dev"
      # run "mkdir -p ~/sources"
      # package = "nginx-0.5.33.tar.gz"
      # package_dir = package.split(".tar",2)[0]
      # run "cd ~/sources/; wget -q http://sysoev.ru/nginx/%s; tar xzf %s; rm %s" % [package, package, package]
      # #run "tar xzf %s" % package
      # #run "rm %s" % package
      # c = "--prefix=/etc/nginx/ --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --pid-path=/var/run/nginx.pid --lock-path=/var/log/nginx/nginx.lock --error-log-path=/var/log/nginx/nginx.error --http-log-path=/var/log/nginx/nginx.log --http-client-body-temp-path=/var/lib/nginx/body --http-proxy-temp-path=/var/lib/nginx/proxy --http-fastcgi-temp-path=/var/lib/nginx/temp --user=www-data --group=www-data --with-debug --with-http_ssl_module --with-sha1=/usr/lib"
      # run "cd ~/sources/%s&& ./configure %s&& make -s" % [package_dir, c] 
      # sudo "/bin/sh -c 'cd ~/sources/%s; make install -s'" % package_dir 
      put(File.read("Configs/nginx.conf"), "/home/%s/nginx.conf" % user)
      put(File.read("Configs/nginx_initd"), "/home/%s/nginx_initd" % user)
      # sudo "mv ~/nginx_initd /etc/init.d/nginx"
      # sudo "chmod +x /etc/init.d/nginx"
      put(File.read("Configs/nginx_logrotate"), "/home/%s/nginx_logrotate" % user)
      # sudo "mv ~/nginx_logrotate /etc/logrotate.d/nginx"
      sudo "aptitude install -y nginx"
    end
  end

  def generate_password(pw_for)
    run 'apg -a 0 -n 1 -M snlc -m 8 -x 12 -q -c `date`' do |channel, stream, data|
      puts data
      passwords[pw_for] = data.strip
      puts passwords.inspect
    end
  end
end
