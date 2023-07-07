# Telia EOC ECM Ansible Playbook

This repository contains Ansible Playbook that deploys latest EOC ECM code from repo.

## Getting Started

These instructions will ensure that you understand how Telia EOC ECM Ansible Playbook works so you can be sure that deployment is executed in right way and all files from repo that are needed for deployment are picked by playbook.

### Prerequisites

This playbook works with repos that have following structure:

```
.
├── 01-SQL
│   ├── ECM
│   └── EOC
├── 02-CATALOG
│   └── PI_Sprint_Latest
├── 03-LIB
├── 04-RESOURCE
├── 05-JAVA
├── 06-METADATA
│   ├── 1-ECM
│   └── 2-EOC
├── 07-MODULES
├── 08-SUPPORT_FILES
├── 09-TEST_CASES
├── 10-ENV_CONFIG
├── 11-OM_DASHBOARD
└── 12-SR_DASHBOARD
```

If there is different structure, it is needed to change some varaibles to match that structure.

### Command to run Ansible Playbook

In Telia we are running Ansible Playbook by giving command line variables while executing playbook. That command line variables are Bamboo variables which are specific for each environment, build,...

Ansible Playbook is executed by this command:

```
ansible-playbook -i inventory-eocecm eoc_ecm_deploy.yml -v --extra-vars 'variable_host_ecm=${bamboo.variable_host_ecm} variable_host_eoc=${bamboo.variable_host_eoc} git_commit_id=${bamboo.planRepository.1.revision} repo_dir_dds=/tmp/working_dir_cicd/repo_metadata_${bamboo.variable_env}_${bamboo.deploy.version} ecm_deploy=${bamboo.variable_ecm_deploy} eoc_deploy=${bamboo.variable_eoc_deploy}'
```

**NOTE**: Inside a folder which is specified by *repo_dir_dds* variable should be *zip* file with metadata.

## Steps included

### Target server preparation

Playbook will transfer metadata repo to target server and create working directory, where it will place all logs, backup, tracking files,...

### Configuration export

Current EOC ECM configuration will be exported to working directory on target server under `backup` directory. File name will be `configExport_<eoc/ecm>_<DATE>_<TIME>_<METADATA GIT COMMIT ID>.xml`

### Metadata deployment

Playbook will deploy metadata for ECM and EOC from their directories `./06-METADATA/1-ECM/` and `./06-METADATA/2-EOC/` using deploy.sh command.

### Configuration import

Playbook will check is there config file in `./10-ENV_CONFIG/` directory and if it is there it will import configuration.

Configuration file should be named `config_[EOC/ECM]_[eocecm_environment].xml`

Possible values for `eocecm_environment` in Norway B2B are:
```
- DEV1_NO_COM
- DEV1_NO_SOM
- DEV2_NO_COM
- DEV2_NO_SOM
- SIT1_NO_COM
- SIT1_NO_SOM
- SIT2_NO_COM
- SIT2_NO_SOM
- DM_NO_COM
- DM_NO_SOM
- AT_NO_COM
- AT_NO_SOM
- Telia_NO_PREPROD_COM
- Telia_NO_PREPROD_SOM
- Telia_NO_PROD_COM
- Telia_NO_PROD_SOM
```

Possible values for `eocecm_environment` in Norway Mobile are:
```
- DEV1_NO_Mobile_SOM
- SIT1_NO_Mobile_SOM
```

Possible values for `eocecm_environment` in Lithuania are:
```
- DEV1_LT
- DEV2_LT
- SIT1_LT
- SIT2_LT
- DM_LT
- UAT_LT
- PROD_LT
```

### Upgrade database

To make the database compatible with the latest metadata, you must upgrade the database. If there is need for upgrade, upgrade sql will be created in server working directory under `upgrade/<EOC/ECM>` directory. Name of file will be `<eoc/ecm>_upgradeSQL_[DATE_AND_TIME].sql`.

If file is created it will be executed on database.

### Tracking files

Tracking files will be written to working directory on target server under `tracking` directory.

This are the files that will be inside `tracking` directory:
```
<ecm/eoc>_last_successful_build_details.txt
<ecm/eoc>_last_successful_build.txt
<ecm/eoc>_second_to_last_successful_build_details.txt
<ecm/eoc>_second_to_last_successful_build.txt
```

Files `<ecm/eoc>_last_successful_build.txt` and `<ecm/eoc>_second_to_last_successful_build.txt` will only contain GIT Commit ID.

And files `<ecm/eoc>_last_successful_build_details.txt` and `<ecm/eoc>_second_to_last_successful_build_details.txt` contain following:
```
This deployment was successful with metadata with Git Commit ID: XXXXXXX2.
It was executed on YYYY/MM/DD at HHhMMminSSs.
```
If needed we can add more information to this tracking file.

### Execute SQL created by developers

If there are SQLs that are created by developers they should be in `./01-SQL/ECM` or `./01-SQL/EOC`.

Some SQL files can be executed multiple times and such files you should list in `ECM_main.sql` or `EOC_main.sql`.

Here is list of example content of `./01-SQL/EOC` directory:
```
Telia_1_SoF_COMMON.sql
Telia_2_SoF_LT.sql
Telia_3_codeTables_COMMON.sql
Telia_4_validate_request_attributes.sql
Telia_5_EHF_Insert.sql
Telia_6_validate_request_against_SR.sql
Telia_EHF_Fault_Insert.sql
Telia_EHF_Fault_Update_Failure.sql
Telia_EHF_Fault_Update_Success.sql
Telia_Notification_Sequence2_OneTimeExecution.sql
Telia_Notification_Sequence_OneTimeExecution.sql
```

In this case you would create file `EOC_main.sql` with following content:
```
\ir Telia_1_SoF_COMMON.sql

\ir Telia_2_SoF_LT.sql

\ir Telia_3_codeTables_COMMON.sql

\ir Telia_4_validate_request_attributes.sql

\ir Telia_5_EHF_Insert.sql

\ir Telia_6_validate_request_against_SR.sql

\ir Telia_EHF_Fault_Insert.sql

\ir Telia_EHF_Fault_Update_Success.sql
```
...and it would execute all this scripts in order of apperance in `EOC_main.sql` file.

### Execute SQL created by developers - OneTimeExecution.sql

If there are SQLs in `./01-SQL/ECM` or `./01-SQL/EOC`, and they have `_OneTimeExecution.sql` in their filename, they will be executed only once on that environment. Playbook will execute them alphabetically, so if order is needed make sure you name them correctly.  

### Deploy Order Manager Dashboard

Playbook will check is there Order Manager Dashboard file in `./11-OM_DASHBOARD/` directory and if it is there it will deploy that file to target server. If there is existing *om.war* file it will backup it in `[WORKING_DIR]/backup/om`.

Order Manager Dashboard file should be named `om_[eocecm_environment].war`.

For possible values for `eocecm_environment` in Norway and Lithuania please see explanation in *Configuration Import* section.  

### Deploy Service Registry Dashboard

Playbook will check is there Service Registry Dashboard file in `./12-SR_DASHBOARD/` directory and if it is there it will deploy that file to target server. If there is existing *sr.war* file it will backup it in `[WORKING_DIR]/backup/sr`.

Service Registry Dashboard file should be named `sr_[eocecm_environment].war`.

For possible values for `eocecm_environment` in Norway and Lithuania please see explanation in *Configuration Import* section.  

### Restart JBoss

Playbook will here restart all EOC ECM managed servers.

### Catalog import

On ECM nodes there is additional step which includes catalog import. It will check in folder `./02-CATALOG/PI_Sprint_Latest` is there `exportedCatalog.zip` and it will import that catalog file.

## Logs

Logs are located on targert server in `[WORKING_DIR]/logs`. Inside that folder you will find directories:
- `log_eocecm`
    - In `log_eocecm` you will find logs of **EOC ECM commands** such as: *deploy.sh*, *generateUpgradeSQL*,...
- `log_jboss`
    - In `log_jboss` you will find **JBoss logs** for *EOC ECM* managed servers and its modules.
