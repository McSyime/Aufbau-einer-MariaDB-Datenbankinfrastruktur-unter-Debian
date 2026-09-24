Vagrant.configure("2") do |config|
  # Image Debian 12 officielle pour Vagrant
  config.vm.box = "debian/bookworm64"

  # Nom de la machine virtuelle
  config.vm.hostname = "db-server"

  # Réseau privé accessible depuis la machine hôte
  config.vm.network "private_network", ip: "192.168.56.20"

  # Accès à phpMyAdmin depuis l'hôte via http://localhost:8080
  config.vm.network "forwarded_port",
                    guest: 80,
                    host: 8080,
                    host_ip: "127.0.0.1",
                    auto_correct: true

  config.vm.provider "virtualbox" do |vb|
    vb.name = "database-server"
    vb.memory = 2048
    vb.cpus = 2
  end

  # Configuration minimale de Debian au premier démarrage
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y curl vim openssh-server
    systemctl enable --now ssh
  SHELL
end
