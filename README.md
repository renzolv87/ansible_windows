- [Instalar en Windows](#instalar-en-windows)
  * [En el cliente Windows](#en-el-cliente-windows)
  * [En Ansible Master](#en-ansible-master)

# Instalar en Windows

## En el cliente Windows
* **Documentación oficial:** https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html
* **Preguntas Frecuentes:** https://docs.ansible.com/ansible/latest/user_guide/windows_faq.html

* Validar PowerShell Version
<pre>
PS C:\Users\rlujan> $PSVersionTable.PSVersion

Major  Minor  Build  Revision
-----  -----  -----  --------
5      1      17763  592


PS C:\Users\rlujan>
</pre>

* Abrir como administrador Windows PowerShell ISE

* Ejecutamos este power shell que upgradea power shell si es necesario y hace un reinicio:
<pre>
$url = "https://raw.githubusercontent.com/jborean93/ansible-windows/master/scripts/Upgrade-PowerShell.ps1"
$file = "$env:temp\Upgrade-PowerShell.ps1"
$username = "rlujan"
$password = "rlujan"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force

#Version can be 3.0, 4.0 or 5.1
&$file -Version 5.1 -Username $username -Password $password -Verbose
</pre>

* Remove auto logon and set the execution policy back to the default of Restricted. You can do this with the following PowerShell commands:
<pre>
#This isn't needed but is a good security practice to complete
Set-ExecutionPolicy -ExecutionPolicy Restricted -Force

$reg_winlogon_path = "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
Set-ItemProperty -Path $reg_winlogon_path -Name AutoAdminLogon -Value 0
Remove-ItemProperty -Path $reg_winlogon_path -Name DefaultUserName -ErrorAction SilentlyContinue
Remove-ItemProperty -Path $reg_winlogon_path -Name DefaultPassword -ErrorAction SilentlyContinue
</pre>

* **OMITIR:** Si tuviéramos versión de power shell 3 hay que ejecutar el WinRM Memory Hotfix
<pre>
$url = "https://raw.githubusercontent.com/jborean93/ansible-windows/master/scripts/Install-WMF3Hotfix.ps1"
$file = "$env:temp\Install-WMF3Hotfix.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
powershell.exe -ExecutionPolicy ByPass -File $file -Verbose
</pre>

* WinRM Setup
<pre>
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)

powershell.exe -ExecutionPolicy ByPass -File $file
</pre>

* Para ver los WinRM Listeners:
<pre>
winrm enumerate winrm/config/Listener
</pre>

* Validamos que funciona la comunicación:
<pre>
winrs -r:http://windows:5985/wsman -u:rlujan -p:rlujan ipconfig
</pre>

## En Ansible Master
* https://www.vultr.com/docs/how-to-install-and-configure-ansible-on-centos-7-for-use-with-windows-server
* https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html

* ansible.cfg:
<pre>
[root@ansible ansible_windows]# cat ansible.cfg
[defaults]
roles_path    = /etc/ansible/roles:/usr/share/ansible/roles
inventory     = ./hosts
[privilege_escalation]
[paramiko_connection]
[ssh_connection]
[persistent_connection]
[accelerate]
[selinux]
[colors]
[diff]
[root@ansible ansible_windows]# 
</pre>

* En el servidor master de ansible instalamos pywinrm:
<pre>
yum install epel-release
yum -y install python-pip pip
pip install pywinrm
</pre>

* Configuramos las variables de conexión en el groupvars:
<pre>
mkdir group_vars
cd group_vars
ansible-vault create nodes.yml

ansible-vault edit nodes.yml 
ansible_ssh_user: rlujan
ansible_ssh_pass: rlujan
ansible_ssh_port: 5986
ansible_connection: winrm
ansible_winrm_server_cert_validation: ignore
</pre>

* Validamos que llegamos al servidor windows:
<pre>
echo test | telnet windows 5986

echo test | telnet windows 5985
</pre>

* Ejecutamos un ping:
<pre>
[root@ansible ansible_windows]# pwd
/etc/ansible_windows
[root@ansible ansible_windows]# ansible -m win_ping windows
windows | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
[root@ansible ansible_windows]# 
</pre>

* Ansible windows modules:
 * **Online:** https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html
 * **On server:** 
<pre>
ansible-doc win_copy
</pre>

* Si tuviéramos encriptado las variables:
<pre>
ansible -m win_ping windows --ask-vault-pass
</pre>

* Facters:
<pre>
ansible -m setup nodes

ansible -m win_disk_facts windows | more
</pre>

* En el cliente windows vamos a Services, buscamos Print Spooler y vemos que esta Running y lo pararemos desde ansible:
<pre>
ansible -m win_service -a "name=spooler state=stopped" windows

ansible -m win_service -a "name=spooler state=started" windows
</pre>

* Playbooks:
<pre>
[root@ansible ansible_windows]# pwd
/etc/ansible_windows
[root@ansible ansible_windows]# ls -l ansible.cfg hosts
-rw-r--r-- 1 root root 184 May 24 01:03 ansible.cfg
-rw-r--r-- 1 root root  16 May 24 00:06 hosts
[root@ansible ansible_windows]# 

ansible-playbook playbooks/demo.yml --check
ansible-playbook playbooks/demo.yml 
</pre>

* Roles:
<pre>
ansible-galaxy init apache
</pre>

* En cliente windows para ver usuarios:
<pre>
Server Manager -> Tools -> Computer Management -> Local Users and Groups -> Users
</pre>

* Rol run:
<pre>
ansible-playbook masterplaybooks/win_apache.yml --extra-vars="hosts=windows" --tags=notepad --check
ansible-playbook masterplaybooks/win_apache.yml --extra-vars="hosts=windows" --tags=notepad

ansible-playbook masterplaybooks/win_apache.yml --extra-vars="hosts=windows"
</pre>

* En cmd:
<pre>
C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin>httpd.exe -v
Server version: Apache/2.2.25 (Win32)
Server built:   Jul 10 2013 01:52:12

C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin>

"C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin\httpd.exe" -k stop
"C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin\httpd.exe" -k uninstall
del "C:\Program Files (x86)\Apache Software Foundation\Apache2.2\htdocs\index.html"
</pre>

* Probar que funciona, abrir navegador:
<pre>
http://localhost/
</pre>

* Más Ejemplos:
  * https://geekflare.com/ansible-playbook-windows-example/
