hdb_cockpit-registration role
===============

Introduction
------------

THis role is created to let the pipeline take care of automatic registration of new HANA instances into our centralized HANA cockpit. It used the following api calls:

* /uaa-security/oauth/token: getting access token
* resource/RegisteredResourcesGet: getting list of already registered resources
* registration/SystemRegister: take care of registration in HCP

It takes care of the following activities:

- Create user `HANACOCKPIT` in systemDB and tenantDB (if it not exists yet)
- Registers the systemDB and tenantDB in the cockpit (it checks if it already registered)

System databases are registered with naming convention SYSTEMDB@'HANASID'.
The HANASID variabele is read from the ec2 tag called `ec2status.tags.Sap_hdb_sid`, so it is important to use unique HANA database names, otherwise registration will fail because an existing resource already being registerd.

The HANACOCKPIT user is being created using a template script (templates/user_creation.sql.j2). This way it is easy to modify the permissions of this user if this is needed.

```
-- Check if the user exists before creating
DO BEGIN
  DECLARE user_exists INT;
  SELECT COUNT(*) INTO user_exists FROM USERS WHERE USER_NAME = '{{ hana_user }}';
  IF :user_exists = 0 THEN
    EXEC 'CREATE USER {{ hana_user }} PASSWORD "{{ hana_cockpit_user_password }}";';
    EXEC 'GRANT DATA ADMIN TO {{ hana_user }};';
    EXEC 'GRANT CATALOG READ TO {{ hana_user }};';
    EXEC 'GRANT SELECT ON SCHEMA _SYS_STATISTICS TO {{ hana_user }};';
    EXEC 'ALTER USER {{ hana_user }} DISABLE PASSWORD LIFETIME;';
  END IF;
END;
```

As you can see the user is created with a default password {{ hana_cockpit_user_password }} variabele. THis is stored in the Ansible vault.

Please make sure to add above attributes.

Example
-------
YOu can run the playbook manually:
`ansible-playbook -i ansible/inventory-prod ansible/hana-cockpit-register.yml -e aws_env=prod`

```
- name: Register HANA Database in Cockpit
  hosts: ec2hosts
  become: yes
  vars_files:
    - vault.yml
  roles:
    - role: hdbcockpit-registration
      vars:
        hana_sid: "{{ ec2status.tags.Sap_hdb_sid | upper }}"
        hana_systemDB_password: "'{{ master_password }}'" # systemDB needs '' around the password
        hana_tenantDB_password: "'{{ master_password }}'" # tenantDB needs '' around the password
        hana_host: "{{ ec2status.tags.Sap_virtual_db_name }}"
        hana_user: "HANACOCKPIT"
```

Author
------
Initial version by Jaap Hellemons - June 2024