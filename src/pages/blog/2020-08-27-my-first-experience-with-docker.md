---
templateKey: blog-post
title: My First Experience with Docker
date: 2020-08-27T03:21:56.824Z
description: We have a .NET application that has been running for years, but
  once a week, the application fails to recover and needs the server needs to be
  rebooted.  To preemptively reboot the EC2 instance nightly we decided to use
  Docker and ECS  scheduled tasks.
featuredpost: true
featuredimage: /img/461260-docker-containers.jpg
tags:
  - Devops
  - Dockers
  - CI/CD
  - Python
  - Cloud
---
<!--StartFragment-->

Here is what the finished Dockerfile looks like:

<!--EndFragment-->

<!--StartFragment-->

```dockerfile
FROM amazonlinux:latest

RUN yum -y update
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python get-pip.py
RUN pip install boto
COPY ./win_reboot.py /root/
RUN chmod +x /root/win_reboot.py

CMD ["/root/win_reboot.py"]
```



```

```

![]()

<!--EndFragment-->

<!--StartFragment-->

ECS has a role with the correct access to run reboot.

Here is what the python script looks like:

<!--EndFragment-->

<!--StartFragment-->

```python
#!/usr/bin/env python

import boto.ec2
import os

conn = boto.ec2.connect_to_region("us-east-1")
instance_id_list = []
for instance in os.environ['WINDOWS_EC2'].split("|"):
for r in conn.get_all_instances(filters={"tag:Name" : instance}):
	[instance_id_list.append(i.id) for i in r.instances]
conn.reboot_instances(instance_ids=instance_id_list, dry_run=False)
```

<!--EndFragment-->

<!--StartFragment-->

We need to import `boto` and `os` here. `boto` is to do the AWS magic of rebooting the servers and `os` to use an environment variable. Here we use a pipe delimiter to target multiple EC2 instances. This will help out while testing the container on your local machine because you can pass in environment variables on the command line using the -e flag. There is probably a more efficient and elegant way to go about this, but this works for us.

Keep these files in the same directory and run:

<!--EndFragment-->

<!--StartFragment-->

```shell
docker build -t test-name:latest .
```

If you it builds successfully, you can try to run it:

```shell
docker run -it test-name:latest /bin/bash
```

This will run your container interactively and drop you into a bash shell. Alternatively, try running with environment variables passed in.

```shell
docker run test-name:latest -e WINDOWS_EC2='EC2-instance-tagName' -e AWS_DEFAULT_REGION='aws region'-e AWS_ACCESS_KEY_ID='ID GOES HERE' -e AWS_SECRET_ACCESS_KEY='KEY GOES HERE'
```

If all of this is working as expected, you can go to ECS in AWS and create your task definition. It will provide you commands to push your image to ECR.

Maybe I'll add some screenshots here.

After that you might find you have some extra docker images and containers to clean up locally. The following commands should help. Consult the docker documentation for more information. <https://docs.docker.com/engine/reference/commandline/rmi/>

````shell
for i in ```docker images | grep '<none>'| awk '{ print $3 }'```; do docker rmi -f $i; done
for i in `docker container ls --all | awk '{ print $1 }'`; do docker container rm $i; done
````

<!--EndFragment-->