# Airflow Concepts    

What is Airflow
---------------

Below are some key concepts in Airflow. What is discuss is covered in [this talk](https://drive.google.com/file/d/1DVN4HXtOC-HXvv00sEkoB90mxLDnCIKc/view?usp=sharing).

*   [**Tasks**](https://airflow.apache.org/docs/apache-airflow/stable/concepts/tasks.html) These are the things you want to do; the building blocks of the pipeline. Let's say I want to run a script that does some basic data processing when a specific file is created in a particular location. Once this processing is done, a second script writes a report based on the processed data. You might want to model this as a set of tasks i.e. file_created_sensor -> script_1 -> script_2.
    
*   [**Operator**](https://airflow.apache.org/docs/apache-airflow/stable/concepts/operators.html) An `Operator` is a Task template. There are loads of pre-made operators to carry out common tasks. These can include running python or bash scripts, or launching docker containers based on images which contains the code (this is the most common operator we use, achieved using a Kubernetes cluster).
    
*   [**DAG**](https://airflow.apache.org/docs/apache-airflow/stable/concepts/dags.html) (Directed Acyclic Graph) DAGs define the tasks, the order of operations, and the frequency/schedule that the pipeline is run. DAGs have a schedule interval (a cron-expression detailing when and how often to run the pipeline), and are considered "acyclic" because the sequence flow must never loop back on itself. There can be branches and joins, but no cycles. For example task A -> B -> C -> A is not allowed.
    
*   **Schedule Interval** This is an expression describing the start date and end date _bound to the data ingested by the pipeline_, and the frequency at which to run the pipeline. For example, a `@daily` task defined as 19:00:00, starting on 01-01-2022. This task would be triggered just after 18:59:59 the following day (02-01-2022) after the full day's worth of data exists. As such, the scheduler runs the job at the **end** of each period. 
    
*   [Airflow UI](https://airflow.apache.org/docs/apache-airflow/stable/ui.html) The Airflow UI makes it easy to monitor and troubleshoot your data pipelines. You can also use it to manually trigger your workflow.


What is Kubernetes
------------------

[Kubernetes](https://kubernetes.io/) is an open-source orchestration system for automating the deployment, scaling, and management of containers. You will not have access to the Kubernetes cluster but it's helpful to understand some key concepts.

* [**Container**](https://www.docker.com/resources/what-container/) The term to describe a portable software package of code for an application along with the dependencies it needs at run time. Containers isolate software from its environment and ensure that it works uniformly regardless of underlying infrastructure (e.g. running on Windows vs. Linux).

* [**Image**](https://docs.docker.com/get-started/overview/#images) act as a set of instructions to build a container, like a template.

* **Docker** Software framework for building and running containers.

* **Cluster** A Kubernetes cluster is a set of connected worker machines, called nodes, to run the containers.

* [**Pod**](https://kubernetes.io/docs/concepts/workloads/pods/) a group of one or more containers with shared resources, and a specification for how to run the containers.


Airflow Pipeline
----------------

Within the MoJ Analytical Platform, a typical Airflow pipeline consists of the following process. (Actions inside the grey box are automated)

![](/source/images/airflow/airflow-pipeline.drawio.png)

*   The DAG can be triggered by an analyst through the Airflow UI or through a schedule

*   Instead of including the code within the DAG, we mainly use Kubernetes pod operators and pass in an image which contains the required python/R code. This mean the DAG will be a lightweight script and won’t read or process any data
    
*   The Kubernetes pod operator launches a single-container pod in a Kubernetes cluster
    
*   The pod will need permission to access various AWS resources (i.e. run an Athena query, read-write to a bucket, etc). This is achieved by assuming an [Identity and Access Management (IAM) role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) with the relevant permissions
    
*   The output is then usually saved to an S3 bucket

*   There are two separate Airflow environments, each with it’s own Kubernetes cluster:
    
    *   Dev: for training and testing new/updates to pipelines [UI](https://eu-west-1.console.aws.amazon.com/mwaa/home?region=eu-west-1#environments/dev/sso)
        
    *   Prod: for running production pipelines [UI](https://eu-west-1.console.aws.amazon.com/mwaa/home?region=eu-west-1#environments/prod/sso)
    

The next two sections summarises the process for creating (and maintaining) the DAG, image and IAM roles. This uses two deployment pipelines which are automated using various [Github actions](https://github.com/features/actions) that we have created to facilitate this process.

Image Pipeline
--------------

Note that you can skip this pipeline if you already have a working docker image saved to the Data Engineering ECR.

This deployment pipeline creates a docker image which contains the analytical code (python or R) that will be run in the dockerised task. Actions highlighted in grey are automated.

![](/source/images/airflow/image-pipeline.drawio.png)

The image repo must contain the [build-and-push-to-ecr](https://github.com/moj-analytical-services/.github/blob/master/workflow-templates/data-engineering/build-and-push-to-ecr.yml) Github action to push the docker image to the Data Engineering [Elastic Container Registry](https://aws.amazon.com/ecr/) (ECR). This can be done by:

* copying [template-airflow-python](https://github.com/moj-analytical-services/template-airflow-python) for python images
* copying [template-airflow-r] (under construction) for R images
* copying the Github action to an existing image repo

Please see [Image pipeline](/data-engineering-tools/airflow/instructions/image-pipeline) for more details.

DAG Pipeline
---------------------

This deployment pipeline creates the [Directed Acyclic Graph (DAG)](https://airflow.apache.org/docs/apache-airflow/stable/concepts/dags.html#) which defines the tasks that will be run, as well as the IAM role which the Kubernetes pod will need in order to access relevant AWS resources and services. Actions highlighted in grey are automated.

![](/source/images/airflow/dag-pipeline.drawio.png)

You must add the DAG and role policies to [airflow](https://github.com/moj-analytical-services/airflow) following specific rules. See [DAG pipeline](/data-engineering-tools/airflow/instructions/dag-pipeline) for more details. Once you raise the PR and it is approved by data engineering, various Github actions will automatically:

*   validate the DAG and policies adhere to the rules
*   notify DE through a slack notification
*   create/update the IAM role
*   save the DAG to an S3 bucket which can be accessed by Airflow
    

Component Responsibilities
--------------------------

Since an Airflow pipeline consists of so many moving components, it is helpful to summarise everyone’s remit.

Analysts are responsible for creating/maintaining the following components:

*   DAG which defines the analytical tasks that the pipeline will run
*   IAM policies for the IAM Role that the Kubernetes pod will use
*   Image repo and the code that will be run in the tasks
    

Data Engineering is responsible for maintaining the following components:

*   airflow environments
*   kubernetes clusters
*   github actions for automating the deployment pipelines
*   template image repo to base the image repo from
*   user guidance
    
as well as approving the DAG and IAM Role PR.


When not to use an Airflow
--------------------------

*   to trigger an Analytical Platform app (the Analytical Platform team can create cron jobs for this)
    
*   to trigger a one-time memory-intensive overnight task (it will be quicker to create a cron job for this)