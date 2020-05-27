- [Windows](#windows)
  * [Preparar el cliente Windows](#preparar-el-cliente-windows)
  * [En Ansible Master](#en-ansible-master)
  * [Variables a nivel de inventario](#variables-a-nivel-de-inventario)
  * [Ansible vault para encriptar y guardar información sensible](#ansible-vault-para-encriptar-y-guardar-informaci-n-sensible)
  * [Ansible Windows Modules](#ansible-windows-modules)
  * [Playbooks](#playbooks)
  * [Roles](#roles)

# Windows
## Preparar el cliente Windows
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
* **Soportado solo en OS Linux**
* https://blog.deiser.com/es/primeros-pasos-con-ansible
* https://www.vultr.com/docs/how-to-install-and-configure-ansible-on-centos-7-for-use-with-windows-server
* https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html

* ansible.cfg:
<pre>
[root@ansible ansible_windows]# cat ansible.cfg
[defaults]
roles_path    = ./roles
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

* En el servidor master de ansible instalamos pywinrm (necesario para interactuar con winrm):
<pre>
yum install epel-release
yum -y install python-pip pip
pip install pywinrm
</pre>

## Variables a nivel de inventario
  * https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html
<pre>
group_vars
host_Vars
</pre>

* Configuramos las variables de conexión en el groupvars.

## Ansible vault para encriptar y guardar información sensible
<pre>
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

## Ansible Windows Modules
 * **Todos:** https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html
 * **Online:** https://docs.ansible.com/ansible/latest/modules/win_copy_module.html
 * **On server:** ansible-doc win_copy

* **Si he de lanzar modulos con yamls encriptados:**
<pre>
ansible -m win_ping windows --ask-vault-pass
</pre>

* **Facters:**
<pre>
ansible -m setup nodes

ansible -m win_disk_facts windows | more
</pre>

* **Ejecutar modulos Ad-hoc:**
  * En el cliente windows vamos a Services, buscamos Print Spooler y vemos que esta Running y lo pararemos desde ansible:
<pre>
ansible -m win_service -a "name=spooler state=stopped" windows

ansible -m win_service -a "name=spooler state=started" windows
</pre>

## Playbooks
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

* **Idempotencia:** es la propiedad para realizar una acción determinada varias veces y aun así conseguir el mismo resultado que se obtendría si se realizase una sola vez.

## Roles
<pre>
ansible-galaxy init apache
</pre>

* **Componentes de un rol:**
  * **defaults**: Data sobre el rol / aplicación (variables por defecto).
  * **files**: Poner ficheros estáticos aquí. Ficheros que copiaremos a los clientes.
  * **handlers**: Tareas que se basan en algunas acciones. Disparadores (Triggers). Ex: si cambias httpd.conf -> reinicia el servicio.
  * **meta**: Metadatos/Información sobre el rol (Autor, plataformas soportadas, dependencias, etc).
  * **tasks**: Core lógico o código. Ex: Instala paquetes, copia ficheros, configura, etc.
  * **templates**: similar a files pero soportan modificaciones (ficheros dinámicos no estáticos) -> Jinja2 template language.
  * **vars**: Tanto vars como defaults guardan variables pero las variables en vars tienen mayor prioridad.
  * Todos tienen el main.yml que es donde inicia la lectura de cada código.

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

* Validar que funciona, abrir navegador:
<pre>
http://localhost/
</pre>

* Cómo configurar apache como servicio en linea de comandos:
<pre>
C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin>httpd.exe -v
Server version: Apache/2.2.25 (Win32)
Server built:   Jul 10 2013 01:52:12

C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin>

"C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin\httpd.exe" -k stop
"C:\Program Files (x86)\Apache Software Foundation\Apache2.2\bin\httpd.exe" -k uninstall
del "C:\Program Files (x86)\Apache Software Foundation\Apache2.2\htdocs\index.html"
</pre>

