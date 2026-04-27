# VARIABLER TILL ATT SETTA UPP VMAR.
  LOADBALANCER_IP ="192.168.56.10"
  WEBSERVER1_IP ="192.168.56.11"
  WEBSERVER2_IP ="192.168.56.12"
  DATABASE_IP ="192.168.56.13"
  ANSIBLE_REPO ="git@github.com:TofflanSec/Project_J-H.git"
  VM_MEMORY ="512"
  VM_CPUS ="1"

  Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"


  #==========Lastbalanseraren==========
  config.vm.define "lb" do |lb|
    lb.vm.hostname = "lb"
    lb.vm.network "private_network", ip: LOADBALANCER_IP
    lb.vm.network "forwarded_port", guest: 80, host: 8080
    lb.vm.provider "virtualbox" do |vb|
      vb.name = "loadbalancer"
      vb.memory = VM_MEMORY
      vb.cpus = VM_CPUS
    end

    lb.vm.provision "shell", inline: <<-SHELL
        # lastbalanseraren genererar ett ssh lyckelpar och sparar dem i rätt fil.
          sudo -u vagrant ssh-keygen -t ed25519 -f /home/vagrant/.ssh/id_ed25519 -N "" -C "ansible-kontroll"
      # vmen kopierar sedan nyckelparet och lägger dem sedan i en delat mapp så att de andra vmarna kommer åt dem
      cp /home/vagrant/.ssh/id_ed25519.pub /vagrant/ansible_id_ed25519.pub
      # notera att även den privata nyckeln läggs i den delade mappen så att databasen (som startar sist)
      # kan hämta den för sedan kunna exicvera ansibleplaybooks
      cp /home/vagrant/.ssh/id_ed25519 /vagrant/ansible_id_ed25519
      # den publika nyckeln lägs även till i authorized_keys för att göra vmarna mottagliga för ssh trafik
      cat /home/vagrant/.ssh/id_ed25519.pub >> /home/vagrant/.ssh/authorized_keys
      # här ändras rättigheterna på filen så att de andra vmarna har tillgång till att hämta den publika nyckeln
      chmod 600 /home/vagrant/.ssh/authorized_keys
      #när den privata nyckeln är flyttad till den delade mappen så raderas den från lastbalanseraren
      rm /home/vagrant/.ssh/id_ed25519
    SHELL
  end

end
