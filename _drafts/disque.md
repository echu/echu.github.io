---
layout:     post
title:      Disque
summary:    Getting it to work and some design patterns.
categories: tech queue
---

I've been testing [disque](http://github.com/antirez/disque) for some of my use cases at work. We currently have a REST API that uses [rq](http://python-rq.org) to launch asynchronous jobs for text analysis. However, `rq` uses Python for both the worker and the client (_i.e._, the one submitting jobs).

Since we wanted to document our REST API using [swagger](http://swagger.io), we needed to either generate a Python server or use a language-agnostic queue library. While we did consider using standard queue libraries such as [zeromq](http://zeromq.org) to build our own queue, I really liked the plug-and-play nature of `rq`--ensure that Redis is availble and you're good to go. I didn't want to design my own topologies and think about brokers and so on.

While it is possible to implement a work queue in Redis, `disque` seems to have implemented the primitives I needed while retaining the convenience of a Redis server. Its only downside: extremely cutting-edge technology with a small chance of support or future development.

Still, that never stopped a curious mind. So, let's dive in.

I wanted my code to be easily deployed and modified, so for these experiments, I provisioned virtual machines using [terraform](https://terraform.io) and [ansible](http://ansible.com). With `terraform`, I spun up three instances on [Google Cloud Engine](https://cloud.google.com/). After setting up my GCE accounts, here's the terraform file I used (via `terraform apply`):

    resource "google_compute_instance" "disque-node" {
      name          = "disque-${count.index}"
      description   = "The VM hosting a disque server"
      machine_type  = "n1-standard-1"
      zone          = "us-central1-a"
      tags          = ["disque"]

      disk {
          image = "debian-8-jessie-v20150915"
          size  = 100 # for disque persistence
      }

      # this is for ansible to gain access to the machines
      metadata {
        sshKeys = "ansible:${file("keys/gce.pub")}"
      }

      network_interface {
          network = "default"
          access_config {}
      }

      count = 3
    }

With these machines, I installed `disque` with ansible and used `supervisord` to monitor the `disque-server` on each machine. I used the dynamic inventory from [CiscoCloud](https://github.com/CiscoCloud/terraform.py), though I'm hoping Terraform will have Ansible support in the future. Here's my ansible file, which I ran with `ansible-playbook -i terraform.py -s -u ansible --private-key=PATH/TO/PRIVATE/KEY disque.yml `

{% highlight yaml+jinja %}
{% raw %}
- hosts: all
  tasks:
    - name: Create disque node groups
      group_by: key={{ ansible_hostname | regex_replace('-[\d]+', '')}}

- hosts: disque-*
  tasks:
    - name: Install git
      apt: pkg=git state=installed update_cache=true

    - name: Install build tools
      apt: pkg=build-essential state=installed update_cache=true

    - name: Clone disque source
      git: repo=https://github.com/antirez/disque.git dest=/home/ansible/disque

    - name: Build and install disque
      command: make install chdir=/home/ansible/disque

    - name: Install supervisord
      apt: pkg=supervisor state=installed update_cache=true

    - name: Synchronize supervisor configuration
      synchronize: src=disque.conf dest=/etc/supervisor/conf.d/disque.conf
      notify:
        - Restart disque

    - name: Enable supervisord
      service: name=supervisor enabled=yes

    - name: Start supervisord
      service: name=supervisor state=started

  handlers:
    - name: Restart disque
      service: name=supervisor state=restarted

- hosts: disque-0
  tasks:
    - name: Configure disque cluster meets
      command: disque cluster meet {{ hostvars[item].ansible_eth0.ipv4.address }} 7711
      with_items: "{{ groups.disque }}"
      when: "'disque' in item and item != 'disque-0'"
{% endraw %}
{% endhighlight %}

The very last task in my Ansible file is used to tell one of my disque nodes to connect to the others in a cluster. Once I got to this step, I had a running disque cluster on Google Cloud Engine!

I used some simple Python code to test my cluster. For submitting jobs, I used the Python disque client [disq](https://github.com/ryansb/disq). Here's a rough sketch of my job client (in py3):

{% highlight python3 lineanchors %}
import disq
import json

conn = disq.Disque()

def process_job(record):
    job_id = conn.addjob('worker', json.dumps(record))
    print('submitted job {}'.format(job_id))

    # wait for result
    result = get_result(job_id)
    return json.dumps(result)

def get_result(job_id):
    qname, result_id, result = conn.getjob(job_id)[0]
    conn.fastack(result_id)
    return (qname, result_id, json.loads(result.decode('utf8')))

if __name__ == "__main__":
    i = 0
    while True:
      print(process_job({'work': i}))
      i += 1
{% endhighlight %}

Obviously, this code could be made parallel; as is, the code is serial and blocks until some work is done. Furthermore, the work submitted would contain more information, but the biggest challenge I had was figuring out how to return the results of a job. In `rq`, we could write the result into Redis and check the key (_i.e._, the job id), but with `disque`, the only method for communication I had was via queues. My initial solution was to put all the results on a "results" queue, but this would only work if the job client didn't care to receive the results of its job (and only wanted to receive *any* results). Instead, at the suggestion of my coworkers, I made a separate queue for each job id, which did the trick.

In a separate process, my workers were implemented as follows:

{% highlight python3 lineanchors %}
import disq
import json

conn = disq.Disque()

def process_record():
    qname, job_id, job = conn.getjob('worker')[0]
    print("got job {}".format(job_id))
    try:
        result = do_stuf_with(job)
    finally:
        conn.fastack(job_id)
    conn.addjob(job_id, json.dumps(result))

if __name__ == "__main__":
    while True:
        process_record()
{% endhighlight %}

My initial implementation attempted to connect to my GCE instances from my local laptop. However, since the disque `HELLO` command returns the *local* IP address of the cluster, my disque client could not connect to the private IPs. This problem arises with all distributed infrastructure, but a simple solution would be to issue the `CLUSTER MEET` command with public IPs or, better yet, use DNS to resolve hostnames.

Unfortunately, one small issue with disque is that its `CLUSTER MEET` command expects an IP address and gives an error when a hostname is used instead. I believe this should be improved, and the hostnames for the cluster returned as part of the `HELLO` command. One could argue that `disque` is meant to be used in a firewalled environment, so allowing public, remote access is frowned upon.

In any case, I hope this post was helpful and sufficient for folks just getting started with disque.
