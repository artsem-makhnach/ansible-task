# Ansible playbook to add new user and/or authz_keys



# Roles:

1) **init** - performs initial config of the target host, identifies OS family CentOS/Ubuntu and performs some necessary actions to make Ansible playbook work properly
**./roles/init/tasks/main.yml**

```sh
- raw: grep '^ID=' /etc/os-release
  register: os_type
  ignore_errors: yes
  changed_when: false

- block:
  - name: Update APT Cache
    raw: apt-get update 

  - name: install python
    raw: apt-get install -y python-pip
    args:
      creates: /usr/bin/python
  when: '"ubuntu" in os_type.stdout'

- package: name=sudo

- name: gather facts
  setup:
```

2) **user** - adds user, group and ssh-key to authz_keys according vars which are defined in defaults and vars
**./roles/user/tasks/main.yml**

```sh
- name: group & user & add .ssh folder & add user_authorized_key
  block:
    - group:
        name: "{{ (user_primary_group == None) | ternary(user_name, user_primary_group) }}"       # creates group for the user. If user_primary_group is not defined - takes user_name as groupname. For the groups ID - omit this param if not specified
        gid: "{{ (user_primary_group_id == None) | ternary(omit, user_primary_group_id) }}"
    - user:
        name: "{{ user_name | mandatory }}"    # creates user with user_name (mandatory variable) as name and user_id (if not specified - takes from user_primary_group_id, if user_primary_group_id is empty - omit this parameter)
        uid: "{{ (user_id == None) | ternary(((user_primary_group_id == None) | ternary(omit, user_primary_group_id)), user_id) }}"
        group: "{{ (user_primary_group == None) | ternary(user_name, user_primary_group) }}"
    - name: add username to sudoers           # adds user to sudoers
      template:
        src: user.j2
        dest: "/etc/sudoers.d/{{ user_name }}"
        validate: '/usr/sbin/visudo -cf %s'
        backup: yes
  when: user_name != "root"                  # this block will be skipped if root in user_name

- name: add authz key                        # adds ssh-key if user_authorized_key (path to the *.pub keyfile) is not empty
  authorized_key:
    user: "{{ user_name | mandatory}}"
    key: "{{ lookup('file', '{{ user_authorized_key }}') }}"
  when: user_authorized_key != None

```
**./roles/user/defaults/main.yml**

```sh
user_name: user
user_id:
user_primary_group:
user_primary_group_id:
user_authorized_key: "~/.ssh/id_rsa.pub"
```
**./roles/user/vars/main.yml**
```sh
user_name: user2
user_id: 10000
user_primary_group: user
user_primary_group_id: 10000
user_authorized_key: "~/.ssh/id_rsa.pub"
```
