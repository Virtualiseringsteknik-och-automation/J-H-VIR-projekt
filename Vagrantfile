


Vagrant.configure("2") do |config|
  
  #Variabler för VMarnas IPv4-adresser
  LOADBALANCER_IP ="192.168.56.10"
  WEBSERVER1_IP ="192.168.56.11"
  WEBSERVER2_IP ="192.168.56.12"
  WEBSERVER3_IP ="192.168.56.14"  
  DATABASE_IP ="192.168.56.13"
  #Variabler för VMarnas prestanda
  VM_MEMORY ="512"
  VM_CPUS ="1"
  
  config.vm.box = "ubuntu/jammy64"

  #==========Database========== (Definierar en ny VM. Databasen kommer även att aggera ansible kontrollnod.)
  config.vm.define "db" do |db|
    #Tilldelar ett namn till vm i hostmaskinen.
    db.vm.hostname = "db"
    #Tilldelar en ipadress till VM i ett privat nätverk.
    db.vm.network "private_network", ip: DATABASE_IP
    #Anger vilket värdprogrram som ska köra VM.
    db.vm.provider "virtualbox" do |vb|
      #Tilldelar internt namn i VM
      vb.name = "database"
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket CPU VM får använda från värdmaskinen.
      vb.cpus = VM_CPUS
    end
    
    #Läser in secretsfilen där token till github är lagrad.
    secrets = {}
    File.foreach("secrets.env") do |line|
    key, value = line.strip.split("=")
    secrets[key] = value
    end

    #Bootstrapscript
    db.vm.provision "shell", inline: <<-SHELL
      #Uppdatera ubuntu
      apt-get update -y

      #Installera ansible och git
      apt-get install -y ansible git

      #Klona repot till "/home/vagrant/"
      git clone https://TofflanSec:#{secrets["GITHUB_TOKEN"]}@github.com/Virtualiseringsteknik-och-automation/J-H-VIR-projekt.git /home/vagrant/ansible
      chown -R vagrant:vagrant /home/vagrant/ansible
      chmod 755 /home/vagrant/ansible

      #Skapa en SSH-nyckel utan lösenord som sparas i VM:s .ss-mapp för användaren vagrant.
      sudo -u vagrant ssh-keygen -t ed25519 \
      -f /home/vagrant/.ssh/id_ed25519 \
      -N "" -C "ansible-kontroll"

      #Kopierar den publika delen av ssh-nyckeln som nyss skapades till den delade vagrantmappen
      cp /home/vagrant/.ssh/id_ed25519.pub\
      /vagrant/ansible_id_ed25519.pub
      #Kopierar in själva nyckeln till listan med auktoriserade ssh-nycklar
      cat /home/vagrant/.ssh/id_ed25519.pub >> /home/vagrant/.ssh/authorized_keys
      #Ändrar rättigheter på .ssh-mappen
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh

      echo ===db klar===
    SHELL
  end
  #==========Lastbalanseraren==========(Definierar en ny VM.)
  config.vm.define "lb" do |lb|
    #Tilldelar ett namn till vm i hostmaskinen.
    lb.vm.hostname = "lb"
    #Tilldelar en ipadress till VM i ett privat nätverk.
    lb.vm.network "private_network", ip: LOADBALANCER_IP
    #Anger att VM ska använda portforward för att kunna prata med värdmaskinen.
    lb.vm.network "forwarded_port", guest: 80, host: 8080
    #Anger vilket värdprogrram som ska köra VM.
    lb.vm.provider "virtualbox" do |vb|
      #Tilldelar internt namn i VM
      vb.name = "loadbalancer"
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.cpus = VM_CPUS
    end
    lb.vm.provision "shell", inline: <<-SHELL
      #Uppdatera ubuntu
      apt-get update -y

      #Skapar mappen ".ssh" i användarmappen "vagrant"
      mkdir -p /home/vagrant/.ssh

      #Kopierar in den publika ssh-nyckeln till listan med auktoriserade ssh-nycklar
      cat /vagrant/ansible_id_ed25519.pub \
        >> /home/vagrant/.ssh/authorized_keys

      #Ändrar rättigheter på .ssh-mappen
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
    echo ===lb klar===
    SHELL
  end

#==========Webserver1==========
  config.vm.define "web1" do |web1|
    #Tilldelar ett namn till vm i hostmaskinen.
    web1.vm.hostname = "web1"
    #Tilldelar en ipadress till VM i ett privat nätverk.
    web1.vm.network "private_network", ip: WEBSERVER1_IP
    #Anger vilket värdprogrram som ska köra VM.
    web1.vm.provider "virtualbox" do |vb|
      #Tilldelar internt namn i VM
      vb.name = "webserver1"
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.cpus = VM_CPUS
    end
    web1.vm.provision "shell", inline: <<-SHELL
      #Uppdatera ubuntu
      apt-get update -y      
      #Skapar mappen ".ssh" i användarmappen "vagrant"
      mkdir -p /home/vagrant/.ssh

      #Kopierar in den publika ssh-nyckeln till listan med auktoriserade ssh-nycklar
      cat /vagrant/ansible_id_ed25519.pub \
        >> /home/vagrant/.ssh/authorized_keys

      #Ändrar rättigheter på .ssh-mappen
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
    echo ===web1 klar===
    SHELL
  end

#==========Webserver2==========
  config.vm.define "web2" do |web2|
    #Tilldelar ett namn till vm i hostmaskinen.
    web2.vm.hostname = "web2"
    #Tilldelar en ipadress till VM i ett privat nätverk.
    web2.vm.network "private_network", ip: WEBSERVER2_IP
    #Anger vilket värdprogrram som ska köra VM.
    web2.vm.provider "virtualbox" do |vb|
      #Tilldelar internt namn i VM
      vb.name = "webserver2"
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.cpus = VM_CPUS
    end
    web2.vm.provision "shell", inline: <<-SHELL
      #Uppdatera ubuntu
      apt-get update -y      
      #Skapar mappen ".ssh" i användarmappen "vagrant"
      mkdir -p /home/vagrant/.ssh

      #Kopierar in den publika ssh-nyckeln till listan med auktoriserade ssh-nycklar
      cat /vagrant/ansible_id_ed25519.pub \
        >> /home/vagrant/.ssh/authorized_keys

      #Ändrar rättigheter på .ssh-mappen
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
    echo ===web2 klar===
    SHELL
  end

  #==========Webserver3==========
  config.vm.define "web3" do |web3|
    #Tilldelar ett namn till vm i hostmaskinen.
    web3.vm.hostname = "web3"
    #Tilldelar en ipadress till VM i ett privat nätverk.
    web3.vm.network "private_network", ip: WEBSERVER3_IP
    #Anger vilket värdprogrram som ska köra VM.
    web3.vm.provider "virtualbox" do |vb|
      #Tilldelar internt namn i VM
      vb.name = "webserver3"
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket ramminne VM får använda från värdmaskinen.
      vb.cpus = VM_CPUS
    end
    web3.vm.provision "shell", inline: <<-SHELL
      #Uppdatera ubuntu
      apt-get update -y      
      #Skapar mappen ".ssh" i användarmappen "vagrant"
      mkdir -p /home/vagrant/.ssh

      #Kopierar in den publika ssh-nyckeln till listan med auktoriserade ssh-nycklar
      cat /vagrant/ansible_id_ed25519.pub \
        >> /home/vagrant/.ssh/authorized_keys

      #Ändrar rättigheter på .ssh-mappen
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
      chown -R vagrant:vagrant /home/vagrant/.ssh
    echo ===web3 klar===
    SHELL
  end
end
