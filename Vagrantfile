Vagrant.configure("2") do |config|
    config.env.enable

    # Determinar el valor de os_type basado en el valor de BOX_NAME
    os_type = case ENV["BOX_NAME"]
        when "generic/centos8"
            "centos"
        else
            "ubuntu"
    end

    config.vm.define "database" do |db|
        db.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"  # Utilizamos una imagen de Ubuntu 20.04 por defecto
        db.vm.hostname = "db.unir.mx"
        db.vm.network "private_network", ip: ENV["DB_IP"]

        db.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/database.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end

    config.vm.define "wordpress" do |sitio|
        sitio.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"  # Utilizamos una imagen de Ubuntu 20.04 por defecto
        sitio.vm.hostname = "wordpress.unir.mx"
        sitio.vm.network "private_network", ip: ENV["WP_IP"]

        sitio.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/wordpress.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end

    config.vm.define "proxy" do |proxy|
        proxy.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"  # Utilizamos una imagen de Ubuntu 20.04 por defecto
        proxy.vm.hostname = "wordpress.unir.mx"
        proxy.vm.network "private_network", ip: ENV["PROXY_IP"]

        proxy.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/proxy.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end
end