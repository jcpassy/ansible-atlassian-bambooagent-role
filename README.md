Ansible Atlassian BambooAgent role
==================================

This role performs the necessary for running an Atlassian Bamboo remote agent on a target machine.


The role performs the following actions:

* creates the user running the bamboo agent,
* installs the certificate of the bamboo server such that it is possible
  to download the bamboo agent `jar` file directly from the Server without bypassing any security (optional).
* creates the startup scripts for the bamboo agent, populates additional paths (build tools) and additional options
  (either from the system like `CUDA_VISIBLE_DEVICES` or additional Bamboo Agent configurations)
* registers the startup script of the bamboo agent for launching the agent at boot time.
* registers an auto start service on the operating system
* populates the capabilities of the agent


Requirements
------------

Java should be installed on the target operating system. Consider using the `ansible-atlassian-bambooagent-oracle-java` role for this.


Role Variables
--------------


The following variable are defined and can be overridden:

| variable | default | meaning |
|----------|---------|---------|
| `bambooagent_user` | bambooagent | the user running the bamboo agent|
| `bambooagent_group`| bambooagent | the group which the bamboo agent user is in|
| `bambooagent_service_name` | bambooagent| the name of the service running the bamboo agent. This will appear as the service for starup-shutdown admin commands|
| `bambooagent_userhome`| `/home/bambooagent` | the home folder of the bamboo agent user|
| `bambooagent_build_home`| `{{ bambooagent_userhome }}/bamboo-agent-home` | the build folder |
| `bambooagent_version` | 5.11.1.1 | the version of the agent |
| `bamboo_java_jar_file` | "" (empty string) | The final `.jar` of the Bamboo ageng launcher. If empty (default), the role will attempt to fetch this file from the Bamboo server directly.|
| `bambooagent_jar_filename` | `atlassian-bamboo-agent-installer-{{ bambooagent_version }}.jar` | the jar file of the agent |
| `bambooagent_jar_filename_full_path`| `{{ bambooagent_userhome }}/{{bambooagent_jar_filename}}` | the full path location of the jar file **on the remote** |
| `bambooagent_capability_file`| `{{ bambooagent_build_home }}/bin/bamboo-capabilities.properties` | the location of the capabilities file on the remote |
| `bambooagentjava_additional_options`| <ul><li>`-Djava.awt.headless=true`</li><li>`-Dbamboo.home={{ bambooagent_build_home }}`</li></ul> |additional options passed to the Java virtual machine. This should be a list|
| `bambooagent_additional_environment`| `[]` (empty list) | additional environment variables set before running the bamboo agent (eg. `CUDA_VISIBLE_DEVICES=1`). This should be a list |

### Java
The version of the agent should work well with the installed Java. For instance,version 5.11 of the Bamboo agent require Java 8.

### Capabilities
Some specific capabilities may be declared automatically by the agent using a feature of Bamboo: a capability file. This capability file is inside the
installation folder.

The capabilities files will receive the capabilities that are declared by running the playbook. This is a list of pairs (dictionary) indicating the name
of the capability along with its value.

```
- name: '[BAMBOO] empty capabilities declaration'
  set_fact:
    bamboo_capabilities: {}
```

and then as a post_task populates the capability:

```
- name: '[BAMBOO] Flush the capabilities into the agents file'
  lineinfile:
    dest="{{ bambooagent_capability_file }}"
    state=present
    line="{{item.key}}={{item.value}}"
  with_dict: "{{bamboo_capabilities}}"
```

### HTTPS certificate to the service
The certificate should be in the variable `certificate_files` (a list of certificates which are alias/filename pairs) like the following:

```
- certificate_files:
  - alias: "bamboo_server_certificate_alias"
    file: "{{some_root_location}}/bamboo_server_certificate.crt"
```


Dependencies
------------

No additional dependency.

Example Playbook
----------------


```
- hosts: bambooagents

  vars:
    - program_location: /folder/containing/installers/
    - bamboo_server_url: https://my.local.network/bamboo/agentServer

    # Those are the certificates to install to make Java work with the bamboo server
    - certificate_files:
      - alias: "my.certificate.authority.crt"
        file: "{{program_location}}/common/my.certificate.authority.crt"

    # the home folder of the Bamboo agent (for examples, should be different on Linux/OSX/etc)
    - bambooagent_userhome: "/Users/bambooagent"

    # The operating system that will be declared as a capability
    # This is also used for locating the installation files and fixing the paths on the fly
    - bamboo_operating_system: "{% if ansible_distribution=='MacOSX' %}osx{% elif ansible_distribution=='Ubuntu' %}linux{% elif ansible_os_family=='Windows' %}windows{% endif %}"


    # Additional capabilities to declare (example, this should be Linux specific)
    - bamboo_capabilities_to_declare:
      - { key: 'system.builder.command.gcc-4.4',    value: '/usr/bin/gcc-4.4' }
      - { key: 'system.builder.command.gcc-4.6',    value: '/usr/bin/gcc-4.6' }
      - { key: 'system.builder.command.gcc-5.4',    value: '/usr/bin/gcc-5' }
      - { key: 'system.builder.command.python3.4',  value: '/usr/bin/python3.4' }


  pre_tasks:

    - name: '[BAMBOO] empty capabilities declaration'
      set_fact:
        bamboo_capabilities: {}

  roles:

    # Installs the bamboo agent
    # If bamboo_java_jar_file is given, copies from local, otherwise fetches from server
    - role: bamboo-agent-installation
      bamboo_java_jar_file: "{{ program_location }}/common/{{bambooagent_jar_filename}}"

    # Installs OSX command line tools (as defined in the inventory), fixes the paths on the fly
    # (not part of this role)
    - role: ansible-atlassianbamboo-agent-install-dmg
      dmg_to_install:
        - "{{ bamboo_xcode | default({}) }}"
      when: xcode_needs_install|bool

    # Installs cmake (not part of this role)
    - role: ansible-atlassian-bambooagent-cmake
      cmake_installation: "{{ bamboo_cmake_installation }}"

  tasks:


    # Declares default capabilities (example)
    - name: '[BAMBOO] default capabilities'
      set_fact:
        bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
      with_items:
        - key: 'operating_system'
          value: "{{ bamboo_operating_system }}"
        - key: agent_name
          value: "{{ ansible_fqdn }}"
        - key: osversion
          value: "{{ ansible_distribution_version.split('.')[:2] | join('.') }}"
      tags:
      - capability


    # Declares python capabilities (example only)
    - block:
      - name: '[BAMBOO] running python'
        command: python -c "import sys; print('%d.%d\n%s' % (sys.version_info.major, sys.version_info.minor, sys.executable))"
        register: bamboo_capability_python_version

      - name: '[BAMBOO] register python'
        set_fact:
          bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
        with_items:
          - key: 'system.builder.command.python{{bamboo_capability_python_version.stdout_lines.0}}'
            value: '{{ bamboo_capability_python_version.stdout_lines.1 }}'
      tags:
      - capability

    #
    # Install packages and registers capabilities (Ubuntu, example only)
    #
    - block:
      - name: '[BAMBOO] install standard packages'
        apt: name="{{ item }}" update_cache=yes state=present cache_valid_time=3600
        with_items: "{{ bamboo_packages_to_install | default() }}"

      - name: '[BAMBOO] check capabilities into fact'
        set_fact:
          bamboo_capabilities: "{{ bamboo_capabilities | combine({item.key:item.value}) }}"
        with_items: "{{ bamboo_capabilities_to_declare }}"
        when: "{{ bamboo_capabilities_to_declare | default() }}"

      when: (ansible_distribution=="Ubuntu") and
            ("{{ bamboo_packages_to_install | default() }}")
      tags:
      - capability

  post_tasks:

    - name: Display all capabilities for an agent
      debug: msg="{{bamboo_capabilities}}"
      tags:
      - capability

    - name: '[BAMBOO] Flush the capabilities into the agents file'
      lineinfile:
        dest="{{ bambooagent_capability_file }}"
        state=present
        regexp="^{{item.key}}="
        line="{{item.key}}={{item.value}}"
      with_dict: "{{bamboo_capabilities}}"
      tags:
      - capability:write

```

License
-------

BSD

Author Information
------------------

Any comments on the Ansible, PR or bug reports are welcome from the corresponding Github project.