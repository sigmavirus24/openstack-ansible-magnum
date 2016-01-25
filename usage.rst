=========================================
 Using the openstack-ansible-magnum role
=========================================

Below is some very high-level documentation for using this role.

Install Role
============

Create a file called ``requirements.yml`` with the contents:

.. code-block:: yaml

    - name: elasticsearch
      src: Stouts.elasticsearch

    - name: openstack-ansible-magnum
      src: https://github.com/sigmavirus24/openstack-ansible-magnum

To install the role then do:

.. code::

    $ ansible-galaxy install -r requirements.yml

Alternatively simply use

.. code::

    $ ansible-galaxy install \
        git+https://github.com/sigmavirus24/openstack-ansible-magnum

Using the role with OpenStack Ansible
=====================================

An example playbook might look like:

.. code-block:: yaml

    # tests/os-magnum-install.yml
    ---
    # Copyright 2016, Ian Cordasco
    # Copyright 2016, Rackspace
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    - name: Install magnum server
      hosts: magnum_all
      max_fail_percentage: 20
      user: root
      pre_tasks:
        - name: Use the lxc-openstack aa profile
          lxc_container:
            name: "{{ container_name }}"
            container_config:
              - "lxc.aa_profile=lxc-openstack"
          delegate_to: "{{ physical_host }}"
          when: not is_metal | bool
          register: container_config
          tags:
            - lxc-aa-profile
        - name: Wait for container ssh
          wait_for:
            port: "22"
            delay: "{{ ssh_delay }}"
            search_regex: "OpenSSH"
            host: "{{ ansible_ssh_host }}"
          delegate_to: "{{ physical_host }}"
          when: >
                (container_config is defined and container_config | changed) or
                (container_extra_config is defined and container_config | changed)
          register: ssh_wait_check
          until: ssh_wait_check | success
          retries: 3
          tags:
            - ssh-wait
        - name: Sort the rabbitmq servers
          dist_sort:
            value_to_lookup: "{{ container_name }}"
            ref_list: "{{ groups['magnum_all'] }}"
            src_list: "{{ rabbitmq_servers }}"
          register: servers
        - name: Set rabbitmq servers
          set_fact:
            rabbitmq_servers: "{{ servers.sorted_list }}"
        - name: Create log dir
          file:
            path: "{{ item.path }}"
            state: directory
          with_items:
            - { path: "/openstack/log/{{ inventory_hostname }}-magnum" }
          when: is_metal | bool
          tags:
            - magnum-logs
            - magnum-log-dirs
        - name: Create log aggregation links
          file:
            src: "{{ item.src }}"
            dest: "{{ item.dest }}"
            state: "{{ item.state }}"
            force: "yes"
          with_items:
            - { src: "/openstack/log/{{ inventory_hostname }}-magnum", dest: "/var/log/magnum", state: "link" }
          when: is_metal | bool
          tags:
            - magnum-logs
      roles:
        - role: "openstack-ansible-magnum"
          magnum_galera_address: "{{ galera_address }}"
          magnum_venv_tag: "{{ openstack_release }}"
          magnum_venv_download_url: "{{ openstack_repo_url }}/venvs/{{ openstack_release }}/{{ ansible_distribution | lower }}/magnum-{{ openstack_release }}.tgz"
          tags:
            - "os-magnum"
        - { role: "openstack_openrc", tags: [ "openstack-openrc" ] }
        - role: "system_crontab_coordination"
          tags:
            - "system-crontab-coordination"
      vars:
        galera_address: "{{ internal_lb_vip_address }}"
        ansible_hostname: "{{ container_name }}"
        is_metal: "{{ properties.is_metal|default(false) }}"

If you're using this role with the OpenStack Ansible, you'll want to add a 
line to your ``/etc/openstack_deploy/user_secrets.yml`` and rerun the password 
generation script:

.. code-block:: yaml

    magnum_service_password:
    magnum_galera_password:
    magnum_rabbitmq_password:

You will need to make a file named ``magnum.yml`` in 
``/etc/openstack_deploy/env.d/`` with the following contents:

.. code-block:: yaml

    # /etc/openstack_deploy/env.d/magnum.yml
    ---
    component_skel:
      magnum_api:
        belongs_to:
          - magnum_all

    container_skel:
      magnum_container:
        belongs_to:
          - infra_containers
          - os-infra_containers
        contains:
          - magnum_api
        properties:
          service_name: magnum
          contianer_release: trusty

Note that under properties you can also specify whether magnum runs in a 
container or on bare metal with ``is_metal:`` set to ``true`` or ``false``.

Finally, just run the playbook above and you should have a functional 
Searchlight service.
