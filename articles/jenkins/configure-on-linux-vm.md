---
title: Quickstart - Configure Jenkins using Azure CLI
description: Learn how to install Jenkins on an Azure Linux virtual machine and build a sample Java application.
keywords: jenkins, azure, devops, portal, linux, virtual machine
ms.topic: quickstart
ms.date: 08/21/2020
ms.custom: devx-track-jenkins, devx-track-azurecli
---

# Quickstart: Configure Jenkins using Azure CLI

This quickstart shows how to install [Jenkins](https://jenkins.io) on an Ubuntu Linux VM with the tools and plug-ins configured to work with Azure.

In this quickstart, you'll complete these tasks:

> [!div class="checklist"]
> * Create a setup file that downloads and installs Jenkins
> * Create a resource group
> * Create a virtual machine with the setup file
> * Open port 8080 in order to access Jenkins on the virtual machine
> * Connect to the virtual machine via SSH
> * Configure a sample Jenkins job based on a sample Java app in GitHub
> * Build the sample Jenkins job

## Prerequisites

[!INCLUDE [open-source-devops-prereqs-azure-subscription.md](../includes/open-source-devops-prereqs-azure-subscription.md)]

## Troubleshooting

If you encounter any problems configuring Jenkins, refer to the [Cloudbees Jenkins installation page](https://www.jenkins.io/doc/book/installing/) for the latest instructions and known issues.

## Create a virtual machine

1. Sign in to the [Azure portal](https://portal.azure.com).

1. Open [Azure Cloud Shell](/azure/cloud-shell/overview) and - if not done already - switch to **Bash**.

1. Create a file named `cloud-init-jenkins.txt`.

    ```bash
    code cloud-init-jenkins.txt
    ```

1. Paste the following code into the new file:

    ```json
    #cloud-config
    package_upgrade: true
    runcmd:
      - apt install openjdk-8-jdk -y
      - wget -qO - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
      - sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
      - apt-get update && apt-get install jenkins -y
      - service jenkins restart
    ```

1. Save the file (**&lt;Ctrl>S**) and exit the editor (**&lt;Ctrl>Q**).

1. Create a resource group using [az group create](/cli/azure/group#az_group_create). You might need to replace the `--location` parameter with the appropriate value for your environment.

    ```azurecli
    az group create \
    --name QuickstartJenkins-rg \
    --location eastus
    ```

1. Create a virtual machine using [az vm create](/cli/azure/vm#az_vm_create).

    ```azurecli
    az vm create \
    --resource-group QuickstartJenkins-rg \
    --name QuickstartJenkins-vm \
    --image UbuntuLTS \
    --admin-username "azureuser" \
    --generate-ssh-keys \
    --custom-data cloud-init-jenkins.txt
    ```

1. Verify the creation (and state) of the new virtual machine using [az vm list](/cli/azure/vm#az_vm_list).

    ```azurecli
    az vm list -d -o table --query "[?name=='QuickstartJenkins-vm']"
    ```

1. By default, Jenkins runs on port 8080. Therefore, open port 8080 on the new virtual machine using [az vm open](/cli/azure/vm#az_vm_open_port).

    ```azurecli
    az vm open-port \
    --resource-group QuickstartJenkins-rg \
    --name QuickstartJenkins-vm  \
    --port 8080 --priority 1010
    ```

## Configure Jenkins

1. Get the public IP address for the sample virtual machine using [az vm show](/cli/azure/vm#az_vm_show).

    ```azurecli
    az vm show \
    --resource-group QuickstartJenkins-rg \
    --name QuickstartJenkins-vm -d \
    --query [publicIps] \
    --output tsv
    ```

    **Notes**:

    - The `--query` parameter limits the output to the public IP addresses for the virtual machine.

1. Using the IP address retrieved in the previous step, SSH into the virtual machine. You'll need to confirm the connection request.

    ```azurecli
    ssh azureuser@<ip_address>
    ```

    **Notes**:

    - Upon successful connection, the Cloud Shell prompt includes the user name and virtual machine name: `azureuser@QuickstartJenkins-vm`.

1. Verify that Jenkins is running by getting the status of the Jenkins service.

    ```bash
    service jenkins status
    ```

1. Get the autogenerated Jenkins password.

    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```

1. Using the IP address, open the following URL in a browser: `http://<ip_address>:8080`

1. Enter the password you retrieved earlier and select **Continue**.

    ![Initial page to unlock Jenkins](./media/configure-on-linux-vm/unlock-jenkins.png)

1. Select **Select plug-in to install**.

    ![Select the option to install selected plug-ins](./media/configure-on-linux-vm/select-plugins.png)

1. In the filter box at the top of the page, enter `github`. Select the GitHub plug-in and select **Install**.

    ![Install the GitHub plug-ins](./media/configure-on-linux-vm/install-github-plugin.png)

1. Enter the information for the first admin user and select **Save and Continue**.

    ![Enter information for first admin user](./media/configure-on-linux-vm/create-first-user.png)

1. On the **Instance Configuration** page, select **Save and Finish**.

    ![Confirmation page for instance configuration](./media/configure-on-linux-vm/instance-configuration.png)

1. Select **Start using Jenkins**.

    ![Jenkins is ready!](./media/configure-on-linux-vm/start-using-jenkins.png)

## Create your first job

1. On the Jenkins home page, select **Create a job**.

    ![Jenkins console home page](./media/configure-on-linux-vm/jenkins-home-page.png)

1. Enter a job name of `mySampleApp`, select **Freestyle project**, and select **OK**.

    ![New job creation](./media/configure-on-linux-vm/new-job.png)

1. Select the **Source Code Management** tab. Enable **Git** and enter the following URL for the **Repository URL** value: `https://github.com/spring-guides/gs-spring-boot.git`. Then change the **Branch Specifier** to `*/main`.

    ![Define the Git repo](./media/configure-on-linux-vm/source-code-management.png)

1. Select the **Build** tab, then select **Add build step**

    ![Add a new build step](./media/configure-on-linux-vm/add-build-step.png)

1. From the drop-down menu, select **Invoke Gradle script**.

    ![Select the Gradle script option](./media/configure-on-linux-vm/invoke-gradle-script-option.png)

1. Select **Use Gradle Wrapper**, then enter `complete` in **Wrapper location** and `build` for **Tasks**.

    ![Gradle script options](./media/configure-on-linux-vm/gradle-script-options.png)

1. Select **Advanced** and enter `complete` in the **Root Build script** field.

    ![Advanced Gradle script options](./media/configure-on-linux-vm/root-build-script.png)

1. Scroll to the bottom of the page, and select **Save**.

## Build the sample Java app

1. When the home page for your project displays, select **Build Now** to compile the code and package the sample app.

    ![Project home page](./media/configure-on-linux-vm/project-home-page.png)

1. A graphic below the **Build History** heading indicates that the job is being built.

    ![Job-build in progress](./media/configure-on-linux-vm/job-currently-building.png)

1. When the build completes, select the **Workspace** link.

    ![Select the workspace link](./media/configure-on-linux-vm/job-workspace.png)

1. Navigate to `complete/build/libs` to see that the `.jar` file was successfully built.

    ![The target library verifies that the build succeeded](./media/configure-on-linux-vm/successful-build.png)

1. Your Jenkins server is now ready to build your own projects in Azure!

## Next steps

> [!div class="nextstepaction"]
> [Jenkins on Azure](./index.yml)
