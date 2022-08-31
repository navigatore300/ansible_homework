# Домашнее задание к занятию "08.02 Работа с Playbook"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.  
---
__Ответ:__  
Подготовил папку для выполнения работы.
---  
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.  
---
__Ответ:__    
Выполнено.
---  
3. Подготовьте хосты в соотвтествии с группами из предподготовленного playbook.  
---
__Ответ:__    
Подготовил хост при помощи vagrant для выполнения работы.
---  
4. Скачайте дистрибутив [java](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) и положите его в директорию `playbook/files/`. 
---
__Ответ:__    
```
ls ~/PycharmProjects/ansible_homework/dz_8_2/playbook/files/ -l
total 1006568
-rw-rw-r-- 1 userr userr 318801277 авг 30 21:37 elasticsearch-7.10.1-linux-x86_64.tar.gz
-rw-r--r-- 1 userr userr 145144564 авг 26 21:29 jdk-11.0.16_linux-x64_bin.deb
-rw-r--r-- 1 userr userr 151626416 авг 26 21:31 jdk-11.0.16_linux-x64_bin.rpm
-rw-r--r-- 1 userr userr 168905265 авг 26 21:29 jdk-11.0.16_linux-x64_bin.tar.gz
-rw-rw-r-- 1 userr userr 246227690 авг 30 21:21 kibana-7.10.1-linux-x86_64.tar.gz
---
```  

## Основная часть
1. Приготовьте свой собственный inventory файл `prod.yml`.  
---
__Ответ:__    
```---
elasticsearch:
  hosts:
    vm1:
      ansible_host: 192.168.56.20
      ansible_connection: ssh
      ansible_user: root
      ansible_ssh_private_key_file: ssh_key/id_rsa
```
---  
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.  
---
__Ответ:__    
Загрузка через url работает, но для ускорения процесса изменен под копирование архива из локальной папки.
```angular2html
- name: Install Kibana
  hosts: elasticsearch
  tasks:
#    - name: Get Kibana
#      get_url:
#        url: https://mirrors.huaweicloud.com/kibana/{{ elastic_version }}/kibana-{{ elastic_version }}-linux-x86_64.tar.gz
##        url: https://artifacts.elastic.co/downloads/kibana/kibana-{{ elastic_version }}-linux-x86_64.tar.gz
#        dest: "/tmp/kibana-{{ elastic_version }}-linux-x86_64.tar.gz"
#        mode: 0755
#        timeout: 60
#        force: true
#        validate_certs: false
#      register: get_kibana
#      until: get_kibana is succeeded
#      tags: kibana
    - name: Upload tar.gz Kibana from local storage
      copy:
        src: "{{ kibana_package }}"
        dest: "/tmp/{{ kibana_package }}"
        mode: 0644
      register: get_kibana
      until: get_kibana is succeeded
      tags: kibana
    - name: Create directrory for Kibana
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}"
        mode: 0755
      tags: kibana
    - name: Extract Kibana in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
      tags: kibana
    - name: Set environment Kibana
      become: true
      template:
        src: templates/kibana.sh.j2
        dest: /etc/profile.d/kibana.sh
        mode: 0755
      tags: kibana
```
---  
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.  
---
__Ответ:__    
Описаны в выше.
---  
4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.  
---
__Ответ:__    
Описаны в выше.
---  
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
```angular2html
 ansible-lint site.yml
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml
WARNING  Listing 6 violation(s) that are fatal
risky-file-permissions: File permissions unset or incorrect
site.yml:9 Task/Handler: Upload .tar.gz file containing binaries from local storage

risky-file-permissions: File permissions unset or incorrect
site.yml:16 Task/Handler: Ensure installation dir exists

risky-file-permissions: File permissions unset or incorrect
site.yml:32 Task/Handler: Export environment variables

risky-file-permissions: File permissions unset or incorrect
site.yml:52 Task/Handler: Create directrory for Elasticsearch

risky-file-permissions: File permissions unset or incorrect
site.yml:67 Task/Handler: Set environment Elastic

yaml: no new line character at the end of file (new-line-at-end-of-file)
site.yml:109

You can skip specific rules or tags by adding them to your configuration file:
# .ansible-lint
warn_list:  # or 'skip_list' to silence them completely
  - experimental  # all rules tagged as experimental
  - yaml  # Violations reported by yamllint

Finished with 1 failure(s), 5 warning(s) on 1 files.
```
В модулях для обеспечения безопасности необходимо указать: 0644 для архивов и 0755 для папок и исполняемых файлов.

Предупреждение "Overriding detected file kind" устраняется путем переименования site.yml в playbook.yml

После исправления ошибок и предупреждений не выдается.

---
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.  

`ansible-playbook -i inventory/prod.yml playbook.yml --check`  
Падает с ошибкой на задаче:  
```angular2html
TASK [Extract java in the installation directory] **************************************************************************************************************
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: NoneType: None
fatal: [vm1]: FAILED! => {"changed": false, "msg": "dest '/opt/jdk/11.0.16' must be an existing dir"}
```
Без флага выполняется без проблем. Затем если еще раз выполнить с `--check:  
```angular2html
userr@HP-ProBook:~/PycharmProjects/ansible_homework/dz_8_2/playbook$ ansible-playbook -i inventory/prod.yml playbook.yml --check

PLAY [Install Java] ***************************************************************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************************************************************
ok: [vm1]

TASK [Set facts for Java 11 vars] *************************************************************************************************************************************************************************
ok: [vm1]

TASK [Upload .tar.gz file containing binaries from local storage] *****************************************************************************************************************************************
ok: [vm1]

TASK [Ensure installation dir exists] *********************************************************************************************************************************************************************
changed: [vm1]

TASK [Extract java in the installation directory] *********************************************************************************************************************************************************
skipping: [vm1]

TASK [Export environment variables] ***********************************************************************************************************************************************************************
ok: [vm1]

PLAY [Install Elasticsearch] ******************************************************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************************************************************
ok: [vm1]

TASK [Upload tar.gz Elasticsearch from local storage] *****************************************************************************************************************************************************
ok: [vm1]

TASK [Create directrory for Elasticsearch] ****************************************************************************************************************************************************************
ok: [vm1]

TASK [Extract Elasticsearch in the installation directory] ************************************************************************************************************************************************
skipping: [vm1]

TASK [Set environment Elastic] ****************************************************************************************************************************************************************************
ok: [vm1]

PLAY [Install Kibana] *************************************************************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************************************************************
ok: [vm1]

TASK [Upload tar.gz Kibana from local storage] ************************************************************************************************************************************************************
ok: [vm1]

TASK [Create directrory for Kibana] ***********************************************************************************************************************************************************************
ok: [vm1]

TASK [Extract Kibana in the installation directory] *******************************************************************************************************************************************************
skipping: [vm1]

TASK [Set environment Kibana] *****************************************************************************************************************************************************************************
ok: [vm1]

PLAY RECAP ************************************************************************************************************************************************************************************************
vm1                        : ok=13   changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0  
```
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.  
---
__Ответ:__    
Для результативности вывода надо внести какие нибудь изменения. Меняем 644 на 664 и видим в выводе:
```angular2html
TASK [Upload tar.gz Kibana from local storage] ************************************************************************************************************************************************************
--- before
+++ after
@@ -1,4 +1,4 @@
 {
-    "mode": "0644",
+    "mode": "0664",
     "path": "/tmp/kibana-7.10.1-linux-x86_64.tar.gz"
 }

changed: [vm1]
```
---  
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.  
---
__Ответ:__    
Повторный запуск не показывает изменений.
---
9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
10. Готовый playbook выложите в свой репозиторий, в ответ предоставьте ссылку на него.

## Необязательная часть

1. Приготовьте дополнительный хост для установки logstash.
2. Пропишите данный хост в `prod.yml` в новую группу `logstash`.
3. Дополните playbook ещё одним play, который будет исполнять установку logstash только на выделенный для него хост.
4. Все переменные для нового play определите в отдельный файл `group_vars/logstash/vars.yml`.
5. Logstash конфиг должен конфигурироваться в части ссылки на elasticsearch (можно взять, например его IP из facts или определить через vars).
6. Дополните README.md, протестируйте playbook, выложите новую версию в github. В ответ предоставьте ссылку на репозиторий.
