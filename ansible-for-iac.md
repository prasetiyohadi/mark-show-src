# Ansible sharing session

Thursday, August 12th, 2021

---
## Ansible best practices

[Tips and tricks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

General tips:
- keep it simple
- use version control

---
## Some issues

- playbook is too complex/too long
- excessive use of `tags`
- "creative" use of inventory
 
---
## Some strategies

- define variables according data structure
- separate inventories for:
  - infrastructure/platform
  - services (service-a, service-b, etc)
- separate playbook for creating platform and for managing platform (e.g. DNS server is a platform)
- use include_tasks, loop, and other patterns

---
### Define variables according data structure

*staging/services/service-a/inventory/groups/all.yml*

```yaml
app: service-a
env: staging
subscription_id: 00000000-0000-0000-0000-000000000000
resource_group: service-a
dns:
  zones:
    - name: example.com
      records:
        - name: www
          ttl: 60
          type: A
          value: 127.0.0.1
_tags:
  app: "{{ app }}"
  env: "{{ env }}"
```

---
### Use include_tasks, loops, & other patterns (1)

*playbooks/dns_records.yml*

```yaml
- name: create dns records
  hosts: localhost
  connection: local
  tasks:
    - name: create dns record
      include_tasks: tasks/dns-zone.yml
      loop: "{{ dns.zones }}"
      loop_control:
        loop_var: zone
```

---
### Use include_tasks, loops, and other patterns (2)

*playbooks/tasks/dns_zone.yml*

```yaml
- name: create dns zone
  # TODO: task to create DNS zone

- name: create dns records for this zone
  include_tasks: dns_record.yml
  loop: "{{ zone.records }}"
  loop_control:
    loop_var: record
```

---
### Use include_tasks, loops, and other patterns (3)

*playbooks/tasks/dns_record.yml*

```yaml
- name: create dns record
  azure.azcollection.azure_rm_privatednsrecordset:
    resource_group: "{{ resource_group }}"
    subscription_id: "{{ subscription_id }}"
    zone_name: "{{ zone.name }}"
    time_to_live: "{{ record.ttl }}"
    relative_name: "{{ record.name }}"
    record_type: "{{ record.type }}"
    records:
      # TODO: check for different record_type
      - entry: "{{ record.value }}"
    tags: "{{ _tags }}"
```

---
## Playbooks limitation

- difficult to reuse
- changes directly impacts deployment
- difficult for versioning

---
## Migrate to collections/roles (1)

- easier to reuse/reproduce similar environment
- enable versioning
- decouple infrastructure code (collections/roles) and config repository (inventory)

---
## Migrate to collections/roles (2)

[Developing collections](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html)

*requirements.yml*

```yaml
collections:
  - name: my.azure_dns
    source: git@REPOSITORY_URL:ansible/collections/my.azure_dns.git
    type: git
    version: 0.1.0
```

---
## Migrate to collections/roles (3)

Run playbook from the collection

```sh
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i staging/services/service-a/inventory \
    my.azure_dns.dns_records -vC
```

---
## Better alternative

Use **terraform** for creating cloud infrastructure. Use **ansible** only for its intended use, that is configuration management.

---
# Thank you
