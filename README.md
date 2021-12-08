# TUTO DLC - Configuration Management Tools and Infrastructure as Code


The goal of this tutorial is to deploy an infrastructure using code (**IaC**).
IaC for Infrastructure as Code, is a type of IT configuration that allows to automatically manage and provision IT infrastructure through code without going through manual processes [1].
To do this, we have a multitude of tools supporting IaC. These can be grouped into two categories according to the languages they support: Declarative Language (Only specify what to do) et Procedural Language (Specify in detail how to do the task).
Thus, we have choosen two tools based on categories presented : 
- [**Ansible**]https://docs.ansible.com/ansible/latest/index.html), based on Declarative language and 
- [**Terraform**](https://www.terraform.io/docs/index.html), based on Procedural Language.

For this tutorial needs, we will use an existing application project named **DOODLE** (https://github.com/selabs-ur1/doodle). 

### Description
The doodle app is made up by :
- a **back** which is developed using the [quarkus.io](https://quarkus.io/) framework
- a **front** which is developed in [angular](https://angular.io/) using the [primeng](https://www.primefaces.org/primeng/)  angular UI component library and the [fullcalendar](https://fullcalendar.io/) graphical component

Our Infrastructure will be composed by two machines myVM4DoodleFrontEnd for the front and myVM4DoodleBackEnd for the back. The work area on this app will be limited on the front end part. We will only deploy a machine, by the mean of IaC tools, which will support the front end of the app. This front vm, to launch properly the front component of the doodle app will needs : 
- a Webserver app; we chose Apache
- to allow connection from internet on 80 port (http ones) 
- to have in a scpecific path (the webserver one) the front end doodle app code compiled

### Requirements
To realize this tutorial, you must have :
- a professional mail adress (from your school) to create a free account to use some cloud providers services. *No needs of Credit card*



## Ansible
*Useful resources for Ansible overview:*
- [Introduction](https://www.youtube.com/watch?v=Cisg9bLhLkk&t=1s&ab_channel=xavki)
- [Notion_and_definitions](https://www.youtube.com/watch?v=tirjpYSMkkM&ab_channel=xavki)

To deploy the infrastructure needed to launch the front end part of doodle app, will have chosen to use the cloud provider [Azure from Microsoft](https://azure.microsoft.com/fr-fr/free/search/?OCID=AID2200187_SEM_48fa2f35e22c1b66c2984e9a8add5db6:G:s&ef_id=48fa2f35e22c1b66c2984e9a8add5db6:G:s&msclkid=48fa2f35e22c1b66c2984e9a8add5db6). 

**Step 1: Create your free Azure acount**
Follow the *start free* instructions from [Azure student page](https://azure.microsoft.com/en-us/free/students/) to create your azure account.
As reminder, normallly no needs to fill any credit card section if you selected the creation of student account which is totally free.
Once the account created, move to `Step 2`.

**Step 2: Create a Principal of service**
On Azure, a Principal of service provides you with a dedicated account to manage Azure resources with Ansible.
Launch **Azure Cloud Shell** on Bash environnement, then accept the **creation of storage** on your Azure student account.
Once the terminal prompt available, run the command `az ad sp create-for-rbac --name myServicePrincipal` to create a principal of service.
Keep the output of the command, particularly values of `appID`, `password` and `tenant`.
Then, run `az account show` to get details on your subscription. As the past one, keep the output information, espacially `id`.

In fact, to authenticate to Azure with the service principal, you need the following:
- SubscriptionID from your account
- ApplicationId fron Principal of service
- Password of the Principal of service
- Tenant of the Principal of Service

Now, construct the pattern of the credential file to connect to your azure principal of service
<p style='text-align: center'>
    <img src='images/Principale_service.PNG' alt="Exemple de rendu indisponible" width='400'/>
</p>

Keep it somewhere, we will use it later.

**Step 3: Create the first machine which will be the Ansible Management principal node**
On **Azure Cloud Shell** on Bash/Powershell environnement, you have already access **ansible**. Run `ansible --version` to verify it.
Create an ansible playbook named `vm1_playbook.yml` in which we will design how to create the main machine which role will be the Ansible Server.
- Content of `vm1_playbook.yml`
```shell script
- name: Create Azure VM which role will be the Ansible Server
  hosts: localhost
  connection: local
  vars_prompt:
    - name: password
      prompt: "Enter the new VM local administrator password"
  tasks:
  - name: Create resource group
    azure_rm_resourcegroup:
      name: tutoDLCAnsible-rg
      location: eastus
  - name: Create virtual network
    azure_rm_virtualnetwork:
      resource_group: tutoDLCAnsible-rg
      name: myVnet
      address_prefixes: "178.0.0.0/16"
  - name: Add subnet
    azure_rm_subnet:
      resource_group: tutoDLCAnsible-rg
      name: mySubnet
      address_prefix: "178.0.10.0/24"
      virtual_network: myVnet
  - name: Create public IP address
    azure_rm_publicipaddress:
      resource_group: tutoDLCAnsible-rg
      allocation_method: Static
      name: myPublicIP
    register: output_ip_address
  - name: Public IP of VM
    debug:
      msg: "The public IP is {{ output_ip_address.state.ip_address }}."
  - name: Create Network Security Group that allows ONLY SSH
    azure_rm_securitygroup:
      resource_group: tutoDLCAnsible-rg
      name: myNetworkSecurityGroup
      rules:
        - name: SSH
          protocol: Tcp
          destination_port_range: 22
          access: Allow
          priority: 1001
          direction: Inbound
  - name: Create Network Security Group to allow Doodle app works
    azure_rm_securitygroup:
      resource_group: tutoDLCAnsible-rg
      name: myNetworkSecurityGroup1
      rules:
        - name: SSH
          protocol: Tcp
          destination_port_range: 22
          access: Allow
          priority: 1001
          direction: Inbound
       - name: port_80
          protocol: any
          destination_port_range: 80
          access: Allow
          priority: 1011
          direction: Inbound
  - name: Create virtual network interface card
    azure_rm_networkinterface:
      resource_group: tutoDLCAnsible-rg
      name: myNIC
      virtual_network: myVnet
      subnet: mySubnet
      public_ip_name: myPublicIP
      security_group: myNetworkSecurityGroup
      private_ip_allocation_method: Static
      private_ip_address: 178.0.10.210
  - name: Create VM
    azure_rm_virtualmachine:
      resource_group: tutoDLCAnsible-rg
      name: myVM4AnsibleServer
      vm_size: Standard_DS1_v2
      admin_username: azureuser
      admin_password: "{{ password }}"
      network_interfaces: myNIC
      image:
        offer: CentOS
        publisher: OpenLogic
        sku: '7.5'
        version: latest
```

As explanation: 
1. The task named `Create resource group` will create a groupe of resouces to which the `vm1` will be part of. In Azure, the concept of **resource group** is a container that holds related resources (Virtual machines, storage accounts, web apps, databases, and virtual networks...); even Resource groups, subscriptions, management groups, and tags are also examples of resources. A good practice in azure is to affect to resource groups resources to well manage them. [useful_link](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview#resource-groups)
2. The task named `Create virtual network`





## Liens utiles
- [1] https://www.lebigdata.fr/infrastructure-as-code-definition