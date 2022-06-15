---
layout: post
title:  "How we servce Airflow as self-service data orchestrator tool"
date:   2022-06-15 17:00:00 +0700
categories: data-engineering
tag: [devops,kubernetes]
---

## Business requirements

Our company member are decentralized that each business team can have their own Data Analyst or Data Engineer. The old data engineering team are transfer to build data platform or support for specific tasks rather than build up all data pipelines.

When data pipeline are distributed to multiple teams then the data platform need to adjust for operation purpose and Airflow is one of demonstration.

## Airflow supported features:

- Login: Not much orchestration tool support login and role management such as Airflow. We always want our application are secure, limit access rather than expose to all the people in the networks.
- RBAC: By default, Airflow using the FAB to limit access right to some functions (DAGs access, connections access,…)

### What we want to add:

- A utilities to handle the access control to DAG, secret level and the resource for each team/role.
- The Python DAG may not familiar to some business user and hard to control/operation to Airflow administrator. That’s why we planned to build a tool called Dag render to render the Python dag from a configuration file.

## How we organize our Airflow:

### Airflow user role:
![center-aligned-image](/images/self-service-airflow/airflow-user-management.png){: .align-center}
- Each Airflow user can be belong to one or multiple groups then for each group will have some dedicated resource such as Gitlab DAG manifest repo, Hashicorp Vault secret space and Airflow permissions.
- We integrate the Airflow user with our company Active directory so the user role should be manage by the IT System Admin instead of Airflow team.
![center-aligned-image](/images/self-service-airflow/ad-role-granter.png)
- The Role Granter is an application was written by ourselves, there are several functions:
    - Trigger daily or by Gitlab CI.
    - Get all Airflow user in Airflow AD trees and search their assigned group.
    - Map user AD group to Airflow roles
    - Grant user airflow role to Airflow instance through Airflow API.
    

### Dag & resource management:

By default, Airflow consider all dag file in Dag Bag at same level which equality for user. When serve Airflow as a service, we want each team should manage the DAG by their own: Web UI access, trigger, using dedicated dag repo,…

To integrate the dag management requirements to Airflow instance, we had to customize some function at database level and append secret backend layers to control the access control on dag, secret layer.

### Dag render:

- The Dag render is an dedicated application written by ourselves, almost in Python to:
    - Take responsibility to render the python dag file from dag config files (we called dag manfiest).
    - While rendering, it’s also validate the configuration based on our defined policies:
        - DAG id convention: `team_id.dag_id` or `project_id.dag_id`
        - User are not allowed to config some arguments related to the role, resource permission such as: `access_control`, `queue`,..
        - Test the rendered DAG with Dag Bag test.

### Team/Project manifest repo:

- Each team or dedicated project will have their own Gitlab repository to store their DAG manifest files.
- Those manifest files will be render and test by Dag render through CI pipeline.
- If the rendered files are passed the pipeline then they will be ingested to central DAG repo.
![center-aligned-image](/images/self-service-airflow/airflow-dag-flow.png)

- By using DAG central repo, the Airflow administrator will easily track or audit the changes on all Airflow dags.
- The Git sync service are ran in kubernetes will take the responsibility to sync all DAGs in central repo to shared k8s volume with Airflow services.

## Customized Airflow:

To allowed fully self-service as mentioned above, we have to customize the Airflow:DAG context is an important variable to control the permissions. The DAG context contains valuable information: dag id, dag owner, dag run time,…. they can be get during the dag execution. With these above information, we can add more control login in the secret and xcom backend to control the secret/xcom access.

### Hashicorp Vault and Customize secret backend

By default, Airflow using it’s meta database to store secrets (connection, variable). They are all encoded by a fernet key but there’s a bit risk in database and fernet key backup, rotation. Moreover, the secret management is not friendly and are not support by FAB, to insert a new secret, the user must have admin rights on secret manager and they can manage all secret or nothing. 

Hashicorp Vault is a great tool to manage the secrets and reuse-able for many projects. Each team can manage their secret their selves without request to Airflow admin. To integrate with Hashicorp Vault, we will using custom secret backend and add more control logic: The dag are only allowed to access their owner secret space. With this feature, the user can not use/access other team/project secrets.

### Customize XCOM backend:

When build-up data pipelines, you would like to sharing data between tasks. By default, there are two options:

- Store the output data to a storage system then access that data in downstream task.
- Share data through Airflow XCOM. This option has a limitation: the data size. Airflow stored XCOM to meta database so to store a large data (GB) to database is not a good idea.

To resolve this issue, we apply customized secret backend which store the XCOM data to disk storage rather than meta database.