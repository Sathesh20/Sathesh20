- name: Ensure libvirt and necessary tools are installed
      apt:
        name:
          - qemu-kvm
          - libvirt-daemon-system
          - libvirt-clients
          - bridge-utils
          - virtinst
          - virt-manager
        state: present
        update_cache: yes
 
    - name: Ensure libvirt service is started and enabled
      systemd:
        name: libvirtd
        state: started
        enabled: yes
 
    - name: Create storage pool if it doesn't exist
      command: "virsh pool-define-as --name default --type dir --target /var/lib/libvirt/images"
      ignore_errors: yes
 
    - name: Start the storage pool
      command: "virsh pool-start default"
      ignore_errors: yes
 
    - name: Autostart the storage pool
      command: "virsh pool-autostart default"
      ignore_errors: yes
 
    - name: Define the virtual machine
      command: >
        virt-install
        --name={{ MYSQL }}
        --ram={{ 128000 }}
        --vcpus={{ 16 }}
        --disk path={{ vm_image }},size={{ vm_disk }}
        --os-variant=ubuntu20.04
        --import
        --network network=default
        --noautoconsole
      register: result
      ignore_errors: yes
 
    - name: Wait for the VM to boot up (10 seconds)
      wait_for:
        timeout: 10
 
    - name: Install necessary software packages
      apt:
        name: "{{ software_packages }}"
        state: present
        update_cache: yes
 
    - name: Ensure Docker service is started and enabled
      systemd:
        name: docker
        state: started
        enabled: yes
 
    - name: Create a user for Docker and add to Docker group
      user:
        name: dockeruser
        state: present
        groups: docker
        append: yes
 
    - name: Install Docker Compose via pip
      pip:
        name: docker-compose
        state: present
 
    - name: Ensure nginx service is started and enabled
      systemd:
        name: nginx
        state: started
        enabled: yes
 
    - name: Create a simple index.html for nginx
      copy:
        dest: /var/www/html/index.html
        content: "<html><body><h1>Welcome to your VM!</h1></body></html>"
 
    - name: Restart nginx to apply changes
      systemd:
        name: nginx
        state: restarted
