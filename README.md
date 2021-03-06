# Домашнее задание к занятию "08.03 Использование Yandex Cloud"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.

## Основная часть
1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.
4. Приготовьте свой собственный inventory файл `prod.yml`.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.

добавил в hosts.yml
```
---
all:
  hosts:
    el-instance:
      ansible_host: 51.250.15.133
    kb-instance:
      ansible_host: 62.84.115.143
    fb-instance:
      ansible_host: 51.250.7.254
  vars:
    ansible_connection: ssh
    ansible_user: derpanter
elasticsearch:
  hosts:
    el-instance:
kibana:
  hosts:
    kb-instance:
filebeat:
  hosts:
    fb-instance:  

```


добавил переменные в playbook/inventory/prod/group_vars/

kibana.yml
```
 ---
kbn_stack_version: "7.14.0"
```

filebeat.yml 

```
---
flb_stack_version: "7.14.0"
```




Добавил файл в Temlpate kibana.yml.j2

```
server.host: 0.0.0.0
elasticsearch.hosts: ["http://{{ hostvars['el-instance']['ansible_facts']['default_ipv4']['address'] }}:9200"]
kibana.index: ".kibana" 


```


Добавил файл в Temlpate filebeat.yml.j2

```

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
output.elasticsearch:
 hosts: ["http://{{ hostvars['el-instance']['ansible_facts']['default_ipv4']['address'] }}:9200"]
setup.kibana:
  host: "http://{{ hostvars['kb-instance']['ansible_facts']['default_ipv4']['address'] }}:5601"

```

Добавил таски в site.yml

```

-name: Install Kibana
  hosts: kibana
  handlers:
    - name: restart kibana
      become: true
      service:
        name: kibana
        state: restarted
  tasks:
    - name: "Download Kibana's rpm"
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kbn_stack_version }}-x86_64.rpm"
        dest: "/tmp/kibana-{{ kbn_stack_version }}-x86_64.rpm"
      register: download_kibana
      until: download_kibana is succeeded
    - name: Install Kibana
      become: true
      yum:
        name: "/tmp/kibana-{{ kbn_stack_version }}-x86_64.rpm"
        state: present
    - name: Configure Kibana
      become: true
      template:
        src: kibana.yml.j2
        dest: /etc/kibana/kibana.yml
      notify: restart kibana
- name: Install filebeat
  hosts: filebeat
  handlers:
    - name: restart filebeat
      become: true
      service:
        name: filebeat
        state: restarted
  tasks:
    - name: "Download Filebeat's rpm"
      get_url:
        url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ flb_stack_version }}-x86_64.rpm"
        dest: "/tmp/filebeat-{{ flb_stack_version }}-x86_64.rpm"
      register: download_filebeat
      until: download_filebeat is succeeded
    - name: Install filebeat
      become: true
      yum:
        name: "/tmp/filebeat-{{ flb_stack_version }}-x86_64.rpm"
        state: present
    - name: Configure Filebeat
      become: true
      template:
        src: filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: restart filebeat
    - name: Set filebeat systemwork
      become: true
      command:
        cmd: filebeat modules enable system
        chdir: /usr/share/filebeat/bin
      register: filebeat_modules
      changed_when: filebeat_modules.stdout != 'Module system is already enabled'
    - name: Load Kibana dashboard
      become: true
      command:
        cmd: filebeat setup
        chdir: /usr/share/filebeat/bin
      register: filebeat_setup
      changed_when: false
      until: filebeat_setup is succeeded


```





6. Попробуйте запустить playbook на этом окружении с флагом `--check`.


```

derpanter@Panters-MBP15 playbook % ansible-playbook -i inventory/prod/ site.yml --check

PLAY [Install Elasticsearch] ***************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [el-instance]

TASK [Download Elasticsearch's rpm] ********************************************************************************************************
ok: [el-instance]

TASK [Install Elasticsearch] ***************************************************************************************************************
ok: [el-instance]

TASK [Configure Elasticsearch] *************************************************************************************************************
ok: [el-instance]

PLAY [Install Kibana] **********************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [kb-instance]

TASK [Download Kibana's rpm] ***************************************************************************************************************
ok: [kb-instance]

TASK [Install Kibana] **********************************************************************************************************************
ok: [kb-instance]

TASK [Configure Kibana] ********************************************************************************************************************
ok: [kb-instance]

PLAY [Install filebeat] ********************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [fb-instance]

TASK [Download Filebeat's rpm] *************************************************************************************************************
ok: [fb-instance]

TASK [Install filebeat] ********************************************************************************************************************
ok: [fb-instance]

TASK [Configure Filebeat] ******************************************************************************************************************
ok: [fb-instance]

TASK [Set filebeat systemwork] *************************************************************************************************************
skipping: [fb-instance]

TASK [Load Kibana dashboard] ***************************************************************************************************************
skipping: [fb-instance]

PLAY RECAP *********************************************************************************************************************************
el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
fb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
kb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```



7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.

```

derpanter@Panters-MBP15 playbook % ansible-playbook -i inventory/prod/ site.yml --diff 

PLAY [Install Elasticsearch] ***************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [el-instance]

TASK [Download Elasticsearch's rpm] ********************************************************************************************************
ok: [el-instance]

TASK [Install Elasticsearch] ***************************************************************************************************************
ok: [el-instance]

TASK [Configure Elasticsearch] *************************************************************************************************************
ok: [el-instance]

PLAY [Install Kibana] **********************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [kb-instance]

TASK [Download Kibana's rpm] ***************************************************************************************************************
ok: [kb-instance]

TASK [Install Kibana] **********************************************************************************************************************
ok: [kb-instance]

TASK [Configure Kibana] ********************************************************************************************************************
ok: [kb-instance]

PLAY [Install filebeat] ********************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [fb-instance]

TASK [Download Filebeat's rpm] *************************************************************************************************************
ok: [fb-instance]

TASK [Install filebeat] ********************************************************************************************************************
ok: [fb-instance]

TASK [Configure Filebeat] ******************************************************************************************************************
ok: [fb-instance]

TASK [Set filebeat systemwork] *************************************************************************************************************
ok: [fb-instance]

TASK [Load Kibana dashboard] ***************************************************************************************************************
ok: [fb-instance]

PLAY RECAP *********************************************************************************************************************************
el-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
fb-instance                : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
kb-instance                : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

