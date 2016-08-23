---
layout: post
title: Vulnerabilities audit with vulners.com and audit
---

Recently [vulners.com](https://blog.vulners.com/linux-vulnerability-audit-in-vulners/) announced cool new API to check your installed packages for vulnerabilities. They also developed [PoC](https://github.com/videns/vulners-scanner) for scanning your system.

I decided that this was perfect case for automation with Ansible. For those who don't know, Ansible is agentless automation+configuration management system. I'm sure what I'm describing could be also done with any other system, like Puppet, Chef, SaltStack and others.

First, I developed custom ansible module to query vulners.api, which can be found [here](https://github.com/lctrcl/vulners-scanner/commit/3bccbccf638cd614656d3e003c867a8b9be2dca2).

But then I found the easier way, which is to use native `uri` module of Ansible itself:

{% raw %}
```yaml
- hosts: centos
#  strategy: debug
  vars: [
      packages_json: "{
         'version': '{{ansible_distribution_major_version}}',
         'os': '{{ansible_distribution}}',
         'package': {{packages.stdout_lines}}
       }",
  ]

  tasks:
   - name: dpkg-query
     shell: dpkg-query -W -f='${Package} ${Version} ${Architecture}\n'
     register: packages
     when: ansible_distribution == 'Debian' or ansible_distribution == 'Ubuntu'

   - name: rpm-query
     shell: rpm -qa
     register: packages
     when: ansible_distribution == 'CentOS' or ansible_distribution == 'RedHat'

   - name: vulners-check
     uri:
      url: https://vulners.com/api/v3/audit/audit/
      return_content: yes
      method: POST
      body: "{{packages_json}}"
      body_format: json
      HEADER_Content-Type: "application/json"
     register: json_response
     when: packages
   - debug: var=json_response.json
```
{% endraw %}

Sample output of it running against freshly installed CentOS 6.8:

```
...
...
...
                    "ntpdate-4.2.6p5-10.el6.centos.x86_64": {
                        "CESA-2016:1141": [
                            {
                                "OSVersion": "6",
                                "bulletinPackage": "ntpdate-4.2.6p5-10.el6.centos.1.x86_64",
                                "bulletinVersion": "4.2.6p5-10.el6.centos.1",
                                "operator": "lt",
                                "providedPackage": "ntpdate-4.2.6p5-10.el6.centos.x86_64",
                                "providedVersion": "0:4.2.6p5-10.el6.centos",
                                "result": true
                            }
                        ]
                    },
                    "openssl-1.0.1e-48.el6.x86_64": {
                        "CESA-2016:0996": [
                            {
                                "OSVersion": "6",
                                "bulletinPackage": "openssl-1.0.1e-48.el6_8.1.x86_64",
                                "bulletinVersion": "1.0.1e-48.el6_8.1",
                                "operator": "lt",
                                "providedPackage": "openssl-1.0.1e-48.el6.x86_64",
                                "providedVersion": "0:1.0.1e-48.el6",
                                "result": true
                            }
                        ]
                    }
                },
                "vulnerabilities": [
                    "CESA-2016:1292",
                    "CESA-2016:0996",
                    "CESA-2016:1141",
                    "CESA-2016:1547",
                    "CESA-2013:0620",
                    "CESA-2016:1406"
                ]
            },
            "result": "OK"
        }
    }
}
```

Sample configuration and playbook can be found [here](https://github.com/lctrcl/vulners-scanner/tree/master/examples/ansible-sample-playbook)

If you already have some automation system in place (like Ansible, Puppet...), I would recommend you checking this vulners.com API and figuring how to implement this in your system. However, it must be noted that you are basically sending every packet you have installed "in the cloud", so use it with this caution in mind -). And yes, patch early and patch often.

# Ansible vulnerabilities audit 101

Install `ansible`:

```
brew install ansible            # MacOS
sudo yum install ansible        # RHEL/CENTOS
sudo apt-get install ansible    # Debian/Ubuntu
```

If you already have ssh keys and configs for remote hosts in `~/.ssh/config`, ansible will pick them.

Example `~/.ssh/config` would be:

{% raw %}
```
Host host1
    hostname 192.168.x.x
    user ubuntu
    IdentityFile ~/.ssh/host1key
Host host2
    hostname 192.168.x.x
    user admin
    IdentityFile ~/.ssh/host2key
```
{% endraw %}

Make `hosts` file with your hosts and groups. You can also define private SSH keys and users here.

{% raw %}
```
host1 ansible_ssh_host=192.168.x.x ansible_ssh_user=ubuntu
host2 ansible_ssh_host=192.168.x.x ansible_ssh_user=admin
host3 ansible_ssh_host=192.168.x.x ansible_ssh_user=admin ansible_ssh_private_key_file=~/.ssh/hostkey

[prod]
host1

[dev]
host2

[misc]
host3
```
{% endraw %}


Make `ansible.cfg` file:

{% raw %}
```
[defaults]
inventory = hosts
retry_files_save_path = "./"
```
{% endraw %}


Get sample ansible playbook, review it, edit and execute it:

```
wget https://raw.githubusercontent.com/lctrcl/vulners-scanner/master/examples/ansible-sample-playbook/vulners-check.yml
vi vulners-check.yml
```

Edit first field `hosts` to match your defined groups. Multiple groups can be separated with `:`

{% raw %}
```
- hosts: prod:dev:misc
```
{% endraw %}

You're good to go, first check that ansible can communicate with hosts

```
ansible all -m

host1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
host2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
host3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Execute `vulners-check.yml` playbook:

```
ansible-playbook  vulners-check-uri.yml
```

If you seeing the vulnerable packages, you can make sure your system is up-to-date with following command:  

```
# Debian/Ubuntu
ansible all -s -m apt -a 'upgrade=full update_cache=yes' --ask-become-pass --become-method=sudo

# RHEL/centos
ansible all -m yum -a "name=* state=latest" --ask-become-pass --become-method=sudo
```

That's it!
