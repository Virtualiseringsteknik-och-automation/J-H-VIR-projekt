


Vagrant.configure("2") do |config|
  
  LOADBALANCER_IP ="192.168.56.10"
  WEBSERVER1_IP ="192.168.56.11"
  WEBSERVER2_IP ="192.168.56.12"
  DATABASE_IP ="192.168.56.13"

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
      #Anger hur myckket ramminne VM får anävnda från värdmaskinen.
      vb.memory = VM_MEMORY
      #Anger hur myckket CPU VM får anävnda från värdmaskinen.
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
end