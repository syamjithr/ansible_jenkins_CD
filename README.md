# ansible_jenkins_CD
## A CD pipeline using ansible, docker and jenkins.
Here I used an ansible playbook to build a flask web_application on docker container. The playbook uses two hosts build and prod. In build server, the playbook creates a docker image by pulling a GitHub repository (which contains Dockerfile and flask web application file) The playbook will clone the GitHub repository and then make a docker image with that particular Dockerfile. The docker image tags will be given according to the tags given in GitHub commits. The playbook will create a docker image called syamjith/jdocker:(2,3,4 according to the tags given by the developer) It will also add the latest tag to the image and push it to the docker hub https://hub.docker.com/repository/docker/syamjith/jdocker. The next session of the playbook creates a docker container with the above docker image on the prod server.

The image build will only happen when a change is occured in the repository no matter how many times you manually run the playbook.

#### By using Jenkins we can create a CD pipeline. We should create a jobs to execute the playbook. The sourcecode of the job should be set as the github repo (https://github.com/syamjithr/jdocker.git). In build area give the playbook play.yml and set the build_trigger (Github hook tigger for GITScm polling). Also setup the webhook with the jenkins server on the github repository.
play.yaml
```
---
- name: "Docker Image/Build and Image/Push"
  hosts: build
  become: true
  vars_files:
    - docker.vars
  vars:
    repo_url: "https://github.com/syamjithr/jdocker.git"
    repo_dest: "/var/repository/"
    image_name: "syamjithr/jdocker"
  tasks:
    
    - name: "Build-Setp - Docker Installation"
      shell: 'amazon-linux-extras install docker -y'
      args:
       warn: no
 
    - name: "Build-Setp - Additional package Installation"
      yum:
        name: git,python-pip
        state: present


    - name: "Build-Setp - python docker extension installation"
      pip:
        name: docker-py

    - name: "Build-Setp - Docker service restart/enable"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Build-Setp - Cloning Repository"
      git:
        repo: "{{ repo_url }}"
        dest: "{{ repo_dest }}"
      register: repo_status



    - name: "Build-Setp - Login to remote Repository"
      docker_login:
        username: syamjithr
        password: "{{ docker_password }}"

            
            
    - name: "Build-Setp - Building image"
      docker_image:
        source: build
        build:
          path: "{{ repo_dest }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ repo_status.after }}"
        - latest
    

    - name: "Build-Setp - removing image"
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ repo_status.after }}"
        - latest



- name: "Docker Run Container"
  hosts: prod
  become: true
  vars:
    image_name: "syamjithr/jdocker"
    
  tasks:
    
    - name: "Deployment - Docker Installation"
      shell: 'amazon-linux-extras install docker -y'
      args:
        warn: no

            
    - name: "Deployment - Additional package Installation"
      yum:
        name: python-pip
        state: present


    - name: "Deployment - python docker extension installation"
      pip:
        name: docker-py

    - name: "Deployment - Docker service restart/enable"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Deployment - Run Container"
      docker_container:
        name: webserver
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:80"
```        
#### Inventory file
 vim hosts
 ```
 [build]
private_ip ansible_user="ec2-user"

[prod]
private_ip ansible_user="ec2-user"
```
#### Variable file
vim docker.vars
```
---
docker_password: dockerhub_password
```
