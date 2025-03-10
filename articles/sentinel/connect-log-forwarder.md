---
title: Deploy a log forwarder to ingest Syslog and CEF logs to Azure Sentinel | Microsoft Docs
description: Learn how to deploy a log forwarder, consisting of a Syslog daemon and the Log Analytics agent, as part of the process of ingesting Syslog and CEF logs to Azure Sentinel.
services: sentinel
documentationcenter: na
author: batamig
manager: rkarlin
editor: ''

ms.service: azure-sentinel
ms.subservice: azure-sentinel
ms.devlang: na
ms.topic: how-to
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 01/05/2021
ms.author: bagol

---
# Deploy a log forwarder to ingest Syslog and CEF logs to Azure Sentinel

To ingest Syslog and CEF logs into Azure Sentinel, particularly from devices and appliances onto which you can't install the Log Analytics agent directly, you'll need to designate and configure a Linux machine that will collect the logs from your devices and forward them to your Azure Sentinel workspace. This machine can be a physical or virtual machine in your on-premises environment, an Azure VM, or a VM in another cloud. 

This machine has two components that take part in this process:

- A syslog daemon, either **rsyslog** or **syslog-ng**, that collects the logs.
- The **Log Analytics Agent** (also known as the OMS Agent), that forwards the logs to Azure Sentinel.

Using the link provided below, you will run a script on the designated machine that performs the following tasks:

- Installs the Log Analytics agent for Linux (also known as the OMS agent) and configures it for the following purposes:
    - listening for CEF messages from the built-in Linux Syslog daemon on TCP port 25226
    - sending the messages securely over TLS to your Azure Sentinel workspace, where they are parsed and enriched

- Configures the built-in Linux Syslog daemon (rsyslog.d/syslog-ng) for the following purposes:
    - listening for Syslog messages from your security solutions on TCP port 514
    - forwarding only the messages it identifies as CEF to the Log Analytics agent on localhost using TCP port 25226
 
## Prerequisites

Your machine must meet the following requirements:

- **Hardware (physical/virtual)**

    - Your Linux machine must have a minimum of **4 CPU cores and 8 GB RAM**.

        > [!NOTE]
        > - A single log forwarder machine using the **rsyslog** daemon has a supported capacity of **up to 8500 events per second (EPS)** collected.

- **Operating system**

    - CentOS 7 and 8 (not 6), including minor versions (64-bit/32-bit)
    - Amazon Linux 2017.09 (64-bit only)
    - Oracle Linux 7 (64-bit/32-bit)
    - Red Hat Enterprise Linux (RHEL) Server 7 and 8 (not 6), including minor versions (64-bit/32-bit)
    - Debian GNU/Linux 8 and 9 (64-bit/32-bit)
    - Ubuntu Linux 14.04 LTS and 16.04 LTS (64-bit/32-bit), and 18.04 LTS (64-bit only)
    - SUSE Linux Enterprise Server 12, 15 (64-bit only)

- **Daemon versions**

    - Rsyslog: v8
    - Syslog-ng: 2.1 - 3.22.1

- **Packages**
    - You must have **python 2.7** or **3** installed on the Linux machine.<br>Use the `python --version` or `python3 --version` command to check.

- **Syslog RFC support**
  - Syslog RFC 3164
  - Syslog RFC 5424
 
- **Configuration**
    - You must have elevated permissions (sudo) on your designated Linux machine.
    - The Linux machine must not be connected to any Azure workspaces before you install the Log Analytics agent.

- **Data**
    - You may need your Azure Sentinel workspace's **Workspace ID** and **Workspace Primary Key** at some point in this process. You can find them in the workspace settings, under **Agents management**.

### Security considerations

Make sure to configure the machine's security according to your organization's security policy. For example, you can configure your network to align with your corporate network security policy and change the ports and protocols in the daemon to align with your requirements. You can use the following instructions to improve your machine security configuration:  [Secure VM in Azure](../virtual-machines/security-policy.md), [Best practices for Network security](../security/fundamentals/network-best-practices.md).

If your devices are sending Syslog and CEF logs over TLS (because, for example, your log forwarder is in the cloud), you will need to configure the Syslog daemon (rsyslog or syslog-ng) to communicate in TLS. See the following documentation for details:  
- [Encrypting Syslog traffic with TLS – rsyslog](https://www.rsyslog.com/doc/v8-stable/tutorials/tls_cert_summary.html)
- [Encrypting log messages with TLS – syslog-ng](https://support.oneidentity.com/technical-documents/syslog-ng-open-source-edition/3.22/administration-guide/60#TOPIC-1209298)

## Run the deployment script
 
1. From the Azure Sentinel navigation menu, select **Data connectors**. Select the connector for your product from the connectors gallery (or the **Common Event Format (CEF)** if your product isn't listed), and then the **Open connector page** button on the lower right. 

1. On the connector page, in the instructions under **1.2 Install the CEF collector on the Linux machine**, copy the link provided under **Run the following script to install and apply the CEF collector**.  
If you don't have access to that page, copy the link from the text below (copying and pasting the **Workspace ID** and **Primary Key** from above in place of the placeholders):

    ```bash
    sudo wget -O cef_installer.py https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/DataConnectors/CEF/cef_installer.py&&sudo python cef_installer.py [WorkspaceID] [Workspace Primary Key]
    ```
1. Paste the link or the text into the command line on your log forwarder, and run it.

1. While the script is running, check to make sure you don't get any error or warning messages.
    - You may get a message directing you to run a command to correct an issue with the mapping of the *Computer* field. See the [explanation in the deployment script](#mapping-command) for details.

1. [Configure your device to send CEF messages](connect-common-event-format.md#configure-your-device).

    > [!NOTE]
    > **Using the same machine to forward both plain Syslog *and* CEF messages**
    >
    > If you plan to use this log forwarder machine to forward [Syslog messages](connect-syslog.md) as well as CEF, then in order to avoid the duplication of events to the Syslog and CommonSecurityLog tables:
    >
    > 1. On each source machine that sends logs to the forwarder in CEF format, you must edit the Syslog configuration file to remove the facilities that are being used to send CEF messages. This way, the facilities that are sent in CEF won't also be sent in Syslog. See [Configure Syslog on Linux agent](../azure-monitor/agents/data-sources-syslog.md#configure-syslog-on-linux-agent) for detailed instructions on how to do this.
    >
    > 1. You must run the following command on those machines to disable the synchronization of the agent with the Syslog configuration in Azure Sentinel. This ensures that the configuration change you made in the previous step does not get overwritten.<br>
    > `sudo su omsagent -c 'python /opt/microsoft/omsconfig/Scripts/OMS_MetaConfigHelper.py --disable'`

## Deployment script explained

The following is a command-by-command description of the actions of the deployment script.

Choose a syslog daemon to see the appropriate description.

# [rsyslog daemon](#tab/rsyslog)

1. **Downloading and installing the Log Analytics agent:**

    - Downloads the installation script for the Log Analytics (OMS) Linux agent.

        ```bash
        wget -O onboard_agent.sh https://raw.githubusercontent.com/Microsoft/OMS-Agent-for-Linux/
            master/installer/scripts/onboard_agent.sh
        ```

    - Installs the Log Analytics agent.
    
        ```bash
        sh onboard_agent.sh -w [workspaceID] -s [Primary Key] -d opinsights.azure.com
        ```

1. **Setting the Log Analytics agent configuration to listen on port 25226 and forward CEF messages to Azure Sentinel:**

    - Downloads the configuration from the Log Analytics agent GitHub repository.

        ```bash
        wget -O /etc/opt/microsoft/omsagent/[workspaceID]/conf/omsagent.d/security_events.conf
            https://raw.githubusercontent.com/microsoft/OMS-Agent-for-Linux/master/installer/conf/
            omsagent.d/security_events.conf
        ```

1. **Configuring the Syslog daemon:**

    - Opens port 514 for TCP communication using the syslog configuration file `/etc/rsyslog.conf`.

    - Configures the daemon to forward CEF messages to the Log Analytics agent on TCP port 25226, by inserting a special configuration file `security-config-omsagent.conf` into the syslog daemon directory `/etc/rsyslog.d/`.

        Contents of the `security-config-omsagent.conf` file:

        ```bash
        if $rawmsg contains "CEF:" or $rawmsg contains "ASA-" then @@127.0.0.1:25226 
        ```

1. **Restarting the Syslog daemon and the Log Analytics agent:**

    - Restarts the rsyslog daemon.
    
        ```bash
        service rsyslog restart
        ```

    - Restarts the Log Analytics agent.

        ```bash
        /opt/microsoft/omsagent/bin/service_control restart [workspaceID]
        ```

1. **Verifying the mapping of the *Computer* field as expected:**

    - Ensures that the *Computer* field in the syslog source is properly mapped in the Log Analytics agent, using the following command: 

        ```bash
        grep -i "'Host' => record\['host'\]"  /opt/microsoft/omsagent/plugin/filter_syslog_security.rb
        ```
    - <a name="mapping-command"></a>If there is an issue with the mapping, the script will produce an error message directing you to **manually run the following command** (applying the Workspace ID in place of the placeholder). The command will ensure the correct mapping and restart the agent.
    
        ```bash
        sudo sed -i -e "/'Severity' => tags\[tags.size - 1\]/ a \ \t 'Host' => record['host']" -e "s/'Severity' => tags\[tags.size - 1\]/&,/" /opt/microsoft/omsagent/plugin/filter_syslog_security.rb && sudo /opt/microsoft/omsagent/bin/service_control restart [workspaceID]
        ```

# [syslog-ng daemon](#tab/syslogng)

1. **Downloading and installing the Log Analytics agent:**

    - Downloads the installation script for the Log Analytics (OMS) Linux agent.

        ```bash
        wget -O onboard_agent.sh https://raw.githubusercontent.com/Microsoft/OMS-Agent-for-Linux/
            master/installer/scripts/onboard_agent.sh
        ```

    - Installs the Log Analytics agent.
    
        ```bash
        sh onboard_agent.sh -w [workspaceID] -s [Primary Key] -d opinsights.azure.com
        ```

1. **Setting the Log Analytics agent configuration to listen on port 25226 and forward CEF messages to Azure Sentinel:**

    - Downloads the configuration from the Log Analytics agent GitHub repository.

        ```bash
        wget -O /etc/opt/microsoft/omsagent/[workspaceID]/conf/omsagent.d/security_events.conf
            https://raw.githubusercontent.com/microsoft/OMS-Agent-for-Linux/master/installer/conf/
            omsagent.d/security_events.conf
        ```

1. **Configuring the Syslog daemon:**

    - Opens port 514 for TCP communication using the syslog configuration file `/etc/syslog-ng/syslog-ng.conf`.

    - Configures the daemon to forward CEF messages to the Log Analytics agent on TCP port 25226, by inserting a special configuration file `security-config-omsagent.conf` into the syslog daemon directory `/etc/syslog-ng/conf.d/`.

        Contents of the `security-config-omsagent.conf` file:

        ```bash
        filter f_oms_filter {match(\"CEF\|ASA\" ) ;};destination oms_destination {tcp(\"127.0.0.1\" port(25226));};
        log {source(s_src);filter(f_oms_filter);destination(oms_destination);};
        ```

1. **Restarting the Syslog daemon and the Log Analytics agent:**

    - Restarts the syslog-ng daemon.
    
        ```bash
        service syslog-ng restart
        ```

    - Restarts the Log Analytics agent.

        ```bash
        /opt/microsoft/omsagent/bin/service_control restart [workspaceID]
        ```

1. **Verifying the mapping of the *Computer* field as expected:**

    - Ensures that the *Computer* field in the syslog source is properly mapped in the Log Analytics agent, using the following command: 

        ```bash
        grep -i "'Host' => record\['host'\]"  /opt/microsoft/omsagent/plugin/filter_syslog_security.rb
        ```
    - <a name="mapping-command"></a>If there is an issue with the mapping, the script will produce an error message directing you to **manually run the following command** (applying the Workspace ID in place of the placeholder). The command will ensure the correct mapping and restart the agent.
    
        ```bash
        sed -i -e "/'Severity' => tags\[tags.size - 1\]/ a \ \t 'Host' => record['host']" -e "s/'Severity' => tags\[tags.size - 1\]/&,/" /opt/microsoft/omsagent/plugin/filter_syslog_security.rb && sudo /opt/microsoft/omsagent/bin/service_control restart [workspaceID]
        ```
---

## Next steps

In this document, you learned how to deploy the Log Analytics agent to connect CEF appliances to Azure Sentinel. To learn more about Azure Sentinel, see the following articles:

- Learn about [CEF and CommonSecurityLog field mapping](cef-name-mapping.md).
- Learn how to [get visibility into your data, and potential threats](get-visibility.md).
- Get started [detecting threats with Azure Sentinel](./detect-threats-built-in.md).
