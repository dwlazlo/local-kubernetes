# Rstudio Workbench and Launcher on Kubernetes

## Install R

reference: https://docs.rstudio.com/resources/install-r/

A fresh Rocky VM

`sudo yum upgrade`

`rockyserver`
`192.168.3.30`

## Basic dependencies

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install dnf-plugins-core
```
## install R
 
-  https://docs.rstudio.com/resources/install-r/

```
export R_VERSION=4.1.3
 
curl -O https://cdn.rstudio.com/r/centos-8/pkgs/R-${R_VERSION}-1-1.x86_64.rpm
 
sudo yum install R-${R_VERSION}-1-1.x86_64.rpm
```

Verify installation:

`/opt/R/${R_VERSION}/bin/R --version`

For first R installation:

```
sudo ln -s /opt/R/${R_VERSION}/bin/R /usr/bin/R
sudo ln -s /opt/R/${R_VERSION}/bin/Rscript /usr/bin/Rscript
```

## install RStudioWorkbench

- https://docs.rstudio.com/rsw/installation/
 
 

``` 
curl -O https://download2.rstudio.org/server/rhel8/x86_64/rstudio-workbench-rhel-2022.02.2-485.pro2-x86_64.rpm
 
sudo yum install rstudio-workbench-rhel-2022.02.2-485.pro2-x86_64.rpm
 
```
## firewalld
 
Rstudio uses port `8787`

``` 
firewall-cmd --permanent --zone=public --add-port=8787/tcp
firewall-cmd --reload
```

navigate to 
```
http://192.168.3.30:8787/
```
and use your system user credentials to login.

## Problems

I encountered problems on a fresh installation of Rocky 8. 

Searching through logs revealed an SELinux problem.

`systemctl status rstudio-server` showed auth issues related to PAM

Following the feedback available via `journalctl -t setroubleshoot` I was able to create a policy module to workaround the restrictions:

`ausearch -c 'rserver-launche' --raw | audit2allow -M my-rserverlauncher.pp`
`sudo semodule -i my-rserverlaunche.pp`


## Kubernetes Launcher

Next step is to configure the Kubernetes Launcher.
