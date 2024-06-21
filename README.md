# RPZ Server Management

This document explains how the RPZ server is implemented and operates within the PDNS environment. It also details the steps to update zone files and validate that updates have been successfully propagated to DNS servers.

For ease of management, the entire process has been automated. The RPZ server updates the zone files it serves whenever a change is merged into the `main` branch of this Git repository.

## Prerequisites
- Directory structure on the Management host created
- RPZ Server automation tools deployed using the existing Ansible playbook
- Zone files available within the Git repository monitored for updates
- Kubernetes cluster hosting the RPZ server exposed as a LoadBalancer service type

## Zone Update & Propagation Workflow

1. Clone the repository with the relevant zone files
2. Create a branch to make necessary updates
3. Merge your changes to the `main` branch (new zone or existing zone update)
4. Wait for the cron job on the Management host to pick up your changes (5-minute check interval)
5. If a zone was added, notify the DNS server to watch for the new zone (Telco Portal or `values.yaml`)
6. Wait for DNS servers to update zones from the RPZ server (5-minute check interval)
7. Validate that the update was successfully propagated to the DNS server

## RPZ Server Automation Tools

An [Ansible playbook](https://simpan.ipdc.cc/AntareX/pldt-pdns/src/branch/main/netprotect/ansible/60-install-rpz-server.yml) performs the necessary configuration changes on the Management host (adding a user, setting up directory structure, creating update scripts, defining cron jobs, etc.) to enable the automated process of RPZ zone updates in the remote Kubernetes cluster.

*Dedicated user account for automated RPZ server management:*

```
$ id rpz_admin
uid=1150(rpz_admin) gid=1150(rpz_admin) groups=1150(rpz_admin)
```

```
$ grep rpz_admin /etc/passwd
rpz_admin:x:1150:1150:RPZ Deployment Automation:/home/rpz_admin:/bin/false
```

*Files and directories related to automation:*

To enable `rpz_admin` to read the repository for changes, it is necessary to store the required git configuration within this user's home directory.

```
/home/rpz_admin/
├── .gitconfig
└── .git-credentials
```

The RPZ Server Kubernetes manifest ([dns-rpz-server.yaml](https://simpan.ipdc.cc/AntareX/pldt-pdns/src/branch/main/netprotect/ansible/templates/rpz/dns-rpz-server.yaml)) and the script for reloading the RPZ server upon zone update ([deploy_rpz_server.sh](https://simpan.ipdc.cc/AntareX/pldt-pdns/src/branch/main/netprotect/ansible/templates/rpz/deploy_rpz_server.sh)) are deployed to a predefined directory using an [Ansible playbook](https://simpan.ipdc.cc/AntareX/pldt-pdns/src/branch/main/netprotect/ansible/60-install-rpz-server.yml) as part of setting up the automated zone update process.

```
/opt/rpz
├── bin
│   └── deploy_rpz_server.sh    --> Checks for updates and trigger zone update on RPZ Server
└── conf
    └── dns-rpz-server.yaml     --> Kubernetes definition of RPZ Server deployment
```

*Cron job triggering the zone update:*

```
$ sudo crontab -l -u rpz_admin
#Ansible: RPZ Server Deployment
*/5 * * * * /opt/rpz/bin/deploy_rpz_server.sh >/dev/null 2>&1
```

## RPZ Zone Update Process

The following section describes the recommended process for updating zone files on the RPZ server.

### 1. Clone the repository

Skip this step if you have already cloned the repository. Instead, ensure you have pulled the latest main branch into your local repository.

```
git clone https://simpan.ipdc.cc/AntareX/pldt-rpz.git
```

```
git checkout main
```

```
git pull
```

### 2. Perform changes to Zone files

With an up-to-date copy of the `main` branch, create a new branch to hold your changes.

```
git checkout -b rpz/zone_update
```

After making all necessary changes, such as adding or removing URLs from zone files or adding a new zone file, commit your changes and push them to the Git repository.

```
git status
```

Add all the files you have modified. You can use `git diff` to review the changes made to each file.

```
git add <file name>
```

Commit the changes with a descriptive message detailing the modifications you made.

```
git commit -m "Remove URLs from zone file abc.rpz"
```

Push the changes to the Git repository.

```
git push origin rpz/zone_update
```

After pushing your branch to the origin, create a Pull Request using the Git Server UI. Based on your repository settings, either wait for your changes to be reviewed and approved, or merge those directly into the `main` branch.

### 3. Validate change propagation

Once the updates are merged into the `main` branch, the cron job will detect the changes within its defined interval and execute the update script. If existing files were modified or new files were created, these changes will be recognized and propagated to the RPZ server. Each execution of the update script is logged to syslog, where you can verify that the script has recognized your update.

```
$ sudo journalctl --since "1 hour ago" -t rpz -f
Jun 24 12:45:03 vpqdnsmgmt rpz[1709933]: Zones were not updated; no action required.
Jun 24 12:50:02 vpqdnsmgmt rpz[1709982]: Zones were not updated; no action required.
Jun 24 12:55:02 vpqdnsmgmt rpz[1710031]: Zones were not updated; no action required.
Jun 24 13:00:02 vpqdnsmgmt rpz[1710082]: Zones were not updated; no action required.
Jun 24 13:05:02 vpqdnsmgmt rpz[1710195]: Zones were updated; reloading RPZ server.
Jun 24 13:05:02 vpqdnsmgmt rpz[1710196]: Deleting RPZ Zone ConfigMap.
Jun 24 13:05:02 vpqdnsmgmt rpz[1710197]: Creating RPZ Zone ConfigMap.
Jun 24 13:05:02 vpqdnsmgmt rpz[1710198]: Restarting RPZ Server Deployment.
Jun 24 13:10:02 vpqdnsmgmt rpz[1710242]: Zones were not updated; no action required.
Jun 24 13:15:02 vpqdnsmgmt rpz[1710291]: Zones were not updated; no action required.
Jun 24 13:20:02 vpqdnsmgmt rpz[1710344]: Zones were not updated; no action required.
Jun 24 13:25:02 vpqdnsmgmt rpz[1710393]: Zones were not updated; no action required.
```

RPZ zones are deployed to a Kubernetes cluster hosting the RPZ server as a ConfigMap resource. When the ConfigMap is updated, the RPZ Server Pod needs to be restarted to mount the updated data. You can verify this by checking the workloads running in the `rpz-server` namespace and by checking the `AGE` of the Pod.

```
$ kubectl -n rpz-server get pod,svc,deploy
NAME                                  READY   STATUS    RESTARTS   AGE
pod/dns-rpz-server-5f74d9d564-rh7hh   1/1     Running   0          3h9m

NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
service/dns-rpz-server-svc   LoadBalancer   172.30.33.193   10.151.45.79   53:31441/TCP   8d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dns-rpz-server   1/1     1            1           8d
```

It is also recommended to check the logs of the Pod.

```
$ kubectl -n rpz-server logs pod/dns-rpz-server-5f74d9d564-rh7hh | egrep '^Zone|^Creating'
Creating empty zone 'csog-ioc'
Zone 'csog-ioc' exists already, replacing contents
Creating empty zone 'e13-illegal-gambling'
Zone 'e13-illegal-gambling' exists already, replacing contents
Creating empty zone 'ra10175-phishing'
Zone 'ra10175-phishing' exists already, replacing contents
Creating empty zone 'ra11479-anti-terrorism'
Zone 'ra11479-anti-terrorism' exists already, replacing contents
Creating RPZ zone csog-ioc with count 68074
Creating RPZ zone e13-illegal-gambling with count 3000
Creating RPZ zone ra10175-phishing with count 96
Creating RPZ zone ra11479-anti-terrorism with count 54
```


### 4. Notify DNS Server (new RPZ zone added)

If you have only added or removed URLs in a specific zone file, it is recommended to randomly check DNS Pods to ensure the zone files were refreshed. However, if you have added a new zone file on the RPZ server, you need to instruct the DNS pods to pull the zone from the RPZ server by either editing the `values.yaml` file or updating the Telco Portal.

To update RPZ zones declaratively via the `values.yaml` file, provide the necessary details as shown in the example below.

```
dnsProxy:
  rpz:
    - endpoints: ["10.151.45.79:53"]
      zone: "csog-ioc"
      authName: "NAME"
      authKey: "KEY"
      authAlgo: "hmac-sha256"
      expectedLength: 100
      refreshSeconds: 300
    - endpoints: ["10.151.45.79:53"]
      zone: "ra11479-anti-terrorism"
      authName: "NAME"
      authKey: "KEY"
      authAlgo: "hmac-sha256"
      expectedLength: 100
      refreshSeconds: 300
```

After updating the file, redeploy NetProtect to apply the changes. This will result in a restart of the DNS Pods.

Alternatively, you can use the Telco Portal UI to notify the DNS pods to synchronize the RPZ zones.

### 4. Troubleshooting

The following steps are recommended for verifying the operational status of the RPZ server or troubleshooting potential issues that may arise at different layers of deployment.

- Ensure that the zone files in the main branch of Git contain the list of URLs
- Confirm that the user exists on the Management host (`id rpz_admin`)
- Check that the automation scripts are available (`tree /opt/rpz`)
- Verify that `rpz_admin` can fetch the git repository (`sudo su - rpz_admin -s /bin/bash`, `cd ~/zones-repo`, `git fetch -v`)
- Make sure a cron job is defined for rpz-admin (`sudo crontab -l -u rpz_admin`)
- Confirm that the script executes at predefined intervals (`grep 'rpz:' /var/log/syslog`)
- Ensure the RPZ server is running (`kubectl -n rpz-server get pod`)
- Verify that the RPZ server is exposed on an External IP (`kubectl -n rpz-server get service`)
- Check that the ConfigMap exists within the RPZ server namespace (`kubectl -n rpz-server get configmaps rpz-lists`)
- Ensure there are no errors in the logs of the RPZ server (`kubectl -n rpz-server logs -l app.kubernetes.io/name=dns-rpz-server`)
- Confirm that the DNS server can reach the RPZ server (`kubectl -n netprotect-vc2dns01 logs -l app.kubernetes.io/name=dns-proxy-main -c dns-proxy`)