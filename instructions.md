What is Needed In This Job?
An image that gets us ready to run Ansible.
A filter to only run this job on master branch.
Our ssh keys/key pair for our EC2 instance.
Install the Ansible CLI in the job.
An inventory file with the hostname or IP address of our EC2 instance in it.
An Ansible playbook that configures the instance and copies files.
A step that executes Ansible.
A dependency in our workflow to make sure the infrastructure creation job finishes before this job runs.
In case you haven't added your SSH keys to Circle CI yet, check out these instructions in the doc to do that now.

https://circleci.com/docs/2.0/workflows#sequential-job-execution-example


https://docs.ansible.com/ansible/2.5/user_guide/intro_getting_started.html#host-key-checking

https://support.circleci.com/hc/en-us/articles/360056463852-Can-I-split-a-config-into-multiple-files-#:~:text=It%20is%20not%20possible%20to,make%20them%20more%20human%2Dreadable

https://circleci.com/docs/2.0/local-cli#packing-a-config