---
title: Create and assign an autoscale scaling plan for Azure Virtual Desktop
description: How to create and assign an autoscale scaling plan to optimize deployment costs.
author: Heidilohr
ms.topic: how-to
ms.date: 01/16/2024
ms.author: helohr
manager: femila
ms.custom: references_regions
---
# Create and assign an autoscale scaling plan for Azure Virtual Desktop

Autoscale lets you scale your session host virtual machines (VMs) in a host pool up or down according to schedule to optimize deployment costs.

To learn more about autoscale, see [Autoscale scaling plans and example scenarios in Azure Virtual Desktop](autoscale-scenarios.md).

>[!NOTE]
> - Azure Virtual Desktop (classic) doesn't support autoscale. 
> - Autoscale doesn't support Azure Virtual Desktop for Azure Stack HCI.
> - You can't use autoscale and [scale session hosts using Azure Automation and Azure Logic Apps](scaling-automation-logic-apps.md) on the same host pool. You must use one or the other.
> - Autoscale is available in Azure and Azure Government.

For best results, we recommend using autoscale with VMs you deployed with Azure Virtual Desktop Azure Resource Manager templates or first-party tools from Microsoft.

## Prerequisites

To use scaling plans, make sure you follow these guidelines:

- Scaling plan configuration data must be stored in the same region as the host pool configuration. Deploying session host VMs is supported in all Azure regions.
- When using autoscale for pooled host pools, you must have a configured *MaxSessionLimit* parameter for that host pool. Don't use the default value. You can configure this value in the host pool settings in the Azure portal or run the [New-AzWvdHostPool](/powershell/module/az.desktopvirtualization/new-azwvdhostpool) or [Update-AzWvdHostPool](/powershell/module/az.desktopvirtualization/update-azwvdhostpool) PowerShell cmdlets.
- You must grant Azure Virtual Desktop access to manage the power state of your session host VMs. You must have the `Microsoft.Authorization/roleAssignments/write` permission on your subscriptions in order to assign the role-based access control (RBAC) role for the Azure Virtual Desktop service principal on those subscriptions. This is part of **User Access Administrator** and **Owner** built in roles.
- If you want to use personal desktop autoscale with hibernation (preview), you will need to [self-register your subscription](../virtual-machines/hibernate-resume.md) and enable the hibernation feature when [creating VMs](deploy-azure-virtual-desktop.md) for your personal host pool. For the full list of prerequisites for hibernation, see [Prerequisites to use hibernation](../virtual-machines/hibernate-resume.md).

    > [!IMPORTANT]
    > Hibernation is currently in PREVIEW.
    > See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## Assign the Desktop Virtualization Power On Off Contributor role with the Azure portal

Before creating your first scaling plan, you'll need to assign the *Desktop Virtualization Power On Off Contributor* RBAC role to the Azure Virtual Desktop service principal with your Azure subscription as the assignable scope. Assigning this role at any level lower than your subscription, such as the resource group, host pool, or VM, will prevent autoscale from working properly. You'll need to add each Azure subscription as an assignable scope that contains host pools and session host VMs you want to use with autoscale. This role and assignment will allow Azure Virtual Desktop to manage the power state of any VMs in those subscriptions. It will also let the service apply actions on both host pools and VMs when there are no active user sessions. 

To learn how to assign the *Desktop Virtualization Power On Off Contributor* role to the Azure Virtual Desktop service principal, see [Assign RBAC roles to the Azure Virtual Desktop service principal](service-principal-assign-roles.md).

## Create a scaling plan

Now that you've assigned the *Desktop Virtualization Power On Off Contributor* role to the service principal on your subscriptions, you can create a scaling plan. To create a scaling plan:

1. Sign in to the [Azure portal](https://portal.azure.com).

1. In the search bar, type *Azure Virtual Desktop* and select the matching service entry.

1. Select **Scaling Plans**, then select **Create**.

1. In the **Basics** tab, look under **Project details** and select the name of the subscription you'll assign the scaling plan to.

1. If you want to make a new resource group, select **Create new**. If you want to use an existing resource group, select its name from the drop-down menu.

1. Enter a name for the scaling plan into the **Name** field.

1. Optionally, you can also add a "friendly" name that will be displayed to your users and a description for your plan.

1. For **Region**, select a region for your scaling plan. The metadata for the object will be stored in the geography associated with the region. To learn more about regions, see [Data locations](data-locations.md).

1. For **Time zone**, select the time zone you'll use with your plan.

1. For **Host pool type**, select the type of host pool that you want your scaling plan to apply to. 

1. In **Exclusion tags**, enter a tag name for VMs you don't want to include in scaling operations. For example, you might want to tag VMs that are set to drain mode so that autoscale doesn't override drain mode during maintenance using the exclusion tag "excludeFromScaling". If you've set "excludeFromScaling" as the tag name field on any of the VMs in the host pool, autoscale won't start, stop, or change the drain mode of those particular VMs.
        
    >[!NOTE]
    >- Though an exclusion tag will exclude the tagged VM from power management scaling operations, tagged VMs will still be considered as part of the calculation of the minimum percentage of hosts.
    >- Make sure not to include any sensitive information in the exclusion tags such as user principal names or other personally identifiable information.

1. Select **Next**, which should take you to the **Schedules** tab. Schedules let you define when autoscale turns VMs on and off throughout the day. The schedule parameters are different based on the **Host pool type** you chose for the scaling plan.

    #### [Pooled host pools](#tab/pooled-autoscale)

    In each phase of the schedule, autoscale only turns off VMs when in doing so the used host pool capacity won't exceed the capacity threshold. The default values you'll see when you try to create a schedule are the suggested values for weekdays, but you can change them as needed. 
    
    To create or change a schedule:
    
    1. In the **Schedules** tab, select **Add schedule**.
    
    1. Enter a name for your schedule into the **Schedule name** field.
    
    1. In the **Repeat on** field, select which days your schedule will repeat on.
    
    1. In the **Ramp up** tab, fill out the following fields:
    
        - For **Start time**, select a time from the drop-down menu to start preparing VMs for peak business hours.
    
        - For **Load balancing algorithm**, we recommend selecting **breadth-first algorithm**. Breadth-first load balancing will distribute users across existing VMs to keep access times fast.
            
            >[!NOTE]
            >The load balancing preference you select here will override the one you selected for your original host pool settings.
    
        - For **Minimum percentage of hosts**, enter the percentage of session hosts you want to always remain on in this phase. If the percentage you enter isn't a whole number, it's rounded up to the nearest whole number. For example, in a host pool of seven session hosts, if you set the minimum percentage of hosts during ramp-up hours to **10%**, one VM will always stay on during ramp-up hours, and it won't be turned off by autoscale.
        
        - For **Capacity threshold**, enter the percentage of available host pool capacity that will trigger a scaling action to take place. For example, if two session hosts in the host pool with a max session limit of 20 are turned on, the available host pool capacity is 40. If you set the capacity threshold to **75%** and the session hosts have more than 30 user sessions, autoscale will turn on a third session host. This will then change the available host pool capacity from 40 to 60.
    
    1. In the **Peak hours** tab, fill out the following fields:
    
        - For **Start time**, enter a start time for when your usage rate is highest during the day. Make sure the time is in the same time zone you specified for your scaling plan. This time is also the end time for the ramp-up phase.
    
        - For **Load balancing**, you can select either breadth-first or depth-first load balancing. Breadth-first load balancing distributes new user sessions across all available session hosts in the host pool. Depth-first load balancing distributes new sessions to any available session host with the highest number of connections that hasn't reached its session limit yet. For more information about load-balancing types, see [Configure the Azure Virtual Desktop load-balancing method](configure-host-pool-load-balancing.md).
    
        > [!NOTE]
        > You can't change the capacity threshold here. Instead, the setting you entered in **Ramp-up** will carry over to this setting.
    
        - For **Ramp-down**, you'll enter values into similar fields to **Ramp-up**, but this time it will be for when your host pool usage drops off. This will include the following fields:
    
          - Start time
          - Load-balancing algorithm
          - Minimum percentage of hosts (%)
          - Capacity threshold (%)
          - Force logoff users
    
        > [!IMPORTANT]
        > - If you've enabled autoscale to force users to sign out during ramp-down, the feature will choose the session host with the lowest number of user sessions to shut down. Autoscale will put the session host in drain mode, send all active user sessions a notification telling them they'll be signed out, and then sign out all users after the specified wait time is over. After autoscale signs out all user sessions, it then deallocates the VM. If you haven't enabled forced sign out during ramp-down, session hosts with no active or disconnected sessions will be deallocated.
        > - During ramp-down, autoscale will only shut down VMs if all existing user sessions in the host pool can be consolidated to fewer VMs without exceeding the capacity threshold.
    
        - Likewise, **Off-peak hours** works the same way as **Peak hours**:
    
          - Start time, which is also the end of the ramp-down period.
          - Load-balancing algorithm. We recommend choosing **depth-first** to gradually reduce the number of session hosts based on sessions on each VM.
          - Just like peak hours, you can't configure the capacity threshold here. Instead, the value you entered in **Ramp-down** will carry over.
    
    #### [Personal host pools](#tab/personal-autoscale)
    
    In each phase of the schedule, define whether VMs should be deallocated based on the user session state. 
    
    To create or change a schedule:
    
    1. In the **Schedules** tab, select **Add schedule**.
    
    1. Enter a name for your schedule into the **Schedule name** field.
    
    1. In the **Repeat on** field, select which days your schedule will repeat on.
    
    1. In the **Ramp up** tab, fill out the following fields:
    
        - For **Start time**, select the time you want the ramp-up phase to start from the drop-down menu.
        
        - For **Start VM on Connect**, select whether you want Start VM on Connect to be enabled during ramp up. 
    
        - For **VMs to start**, select whether you want only personal desktops that have a user assigned to them at the start time to be started, you want all personal desktops in the host pool (regardless of user assignment) to be started, or you want no personal desktops in the pool to be started.
    
        > [!NOTE]
        > We highly recommend that you enable Start VM on Connect if you choose not to start your VMs during the ramp-up phase.
    
        - For **When disconnected for**, specify the number of minutes a user session has to be disconnected before performing a specific action. This number can be anywhere between 0 and 360.
    
        - For **Perform**, specify what action the service should take after a user session has been disconnected for the specified time. The options are to either deallocate (shut down) the VMs, hibernate the personal desktop, or do nothing.
         
        - For **When logged off for**, specify the number of minutes a user session has to be logged off before performing a specific action. This number can be anywhere between 0 and 360.
    
        - For **Perform**, specify what action the service should take after a user session has been logged off for the specified time. The options are to either deallocate (shut down) the VMs, hibernate the personal desktop, or do nothing.
    
    1. In the **Peak hours**, **Ramp-down**, and **Off-peak hours** tabs, fill out the following fields:
    
        - For **Start time**, enter a start time for each phase. This time is also the end time for the previous phase.
        
        - For **Start VM on Connect**, select whether you want to enable Start VM on Connect to be enabled during that phase. 
    
        - For **When disconnected for**, specify the number of minutes a user session has to be disconnected before performing a specific action. This number can be anywhere between 0 and 360.
    
        - For **Perform**, specify what action should be performed after a user session has been disconnected for the specified time. The options are to either deallocate (shut down) the VMs, hibernate the personal desktop, or do nothing.
         
        - For **When logged off for**, specify the number of minutes a user session has to be logged off before performing a specific action. This number can be anywhere between 0 and 360.
    
        - For **Perform**, specify what action should be performed after a user session has been logged off for the specified time. The options are to either deallocate (shut down) the VMs, hibernate the personal desktop, or do nothing.
    ---

1. Select **Next** to take you to the **Host pool assignments** tab. Select the check box next to each host pool you want to include. If you don't want to enable autoscale, unselect all check boxes. You can always return to this setting later and change it. You can only assign the scaling plan to host pools that match the host pool type specified in the plan.

    > [!NOTE]
    > - When you create or update a scaling plan that's already assigned to host pools, its changes will be applied immediately.

1. After that, you'll need to enter **tags**. Tags are name and value pairs that categorize resources for consolidated billing. You can apply the same tag to multiple resources and resource groups. To learn more about tagging resources, see [Use tags to organize your Azure resources](../azure-resource-manager/management/tag-resources.md).

    > [!NOTE] 
    > If you change resource settings on other tabs after creating tags, your tags will be automatically updated.

1. Once you're done, go to the **Review + create** tab and select **Create** to deploy your host pool.

## Edit an existing scaling plan

To edit an existing scaling plan:

1. Sign in to the [Azure portal](https://portal.azure.com).

1. In the search bar, type *Azure Virtual Desktop* and select the matching service entry.

1. Select **Scaling plans**, then select the name of the scaling plan you want to edit. The overview blade of the scaling plan should open.

1. To change the scaling plan host pool assignments, under the **Manage** heading select **Host pool assignments**.

1. To edit schedules, under the **Manage** heading, select **Schedules**.

1. To edit the plan's friendly name, description, time zone, or exclusion tags, go to the **Properties** tab.

## Assign scaling plans to existing host pools

You can assign a scaling plan to any existing host pools in your deployment. When you assign a scaling plan to your host pool, the plan will apply to all session hosts within that host pool. The scaling plan also automatically applies to any new session hosts you create in the assigned host pool.

If you disable a scaling plan, all assigned resources will remain in the state they were in at the time you disabled it.

### Assign a scaling plan to a single existing host pool
To assign a scaling plan to an existing host pool:

1. Open the [Azure portal](https://portal.azure.com).

1. In the search bar, type *Azure Virtual Desktop* and select the matching service entry.

1. Select **Host pools**, and select the host pool you want to assign the scaling plan to.

1. Under the **Settings** heading, select **Scaling plan**, and then select **+ Assign**. Select the scaling plan you want to assign and select **Assign**. The scaling plan must be in the same Azure region as the host pool and the scaling plan's host pool type must match the type of host pool that you're trying to assign it to.

> [!TIP]
> If you've enabled the scaling plan during deployment, then you'll also have the option to disable the plan for the selected host pool in the **Scaling plan** menu by unselecting the **Enable autoscale** checkbox, as shown in the following screenshot.
>
> [!div class="mx-imgBorder"]
> ![A screenshot of the scaling plan window. The "enable autoscale" check box is selected and highlighted with a red border.](media/enable-autoscale.png)

### Assign a scaling plan to multiple existing host pools

To assign a scaling plan to multiple existing host pools at the same time:

1. Open the [Azure portal](https://portal.azure.com).

1. In the search bar, type *Azure Virtual Desktop* and select the matching service entry.

1. Select **Scaling plans**, and select the scaling plan you want to assign to host pools.

1. Under the **Manage** heading, select **Host pool assignments**, and then select **+ Assign**. Select the host pools you want to assign the scaling plan to and select **Assign**. The host pools must be in the same Azure region as the scaling plan and the scaling plan's host pool type must match the type of host pools you're trying to assign it to.

## Next steps

Now that you've created your scaling plan, here are some things you can do:

- [Enable diagnostics for your scaling plan](autoscale-diagnostics.md)

If you'd like to learn more about terms used in this article, check out our [autoscale glossary](autoscale-glossary.md). For examples of how autoscale works, see [Autoscale example scenarios](autoscale-scenarios.md). You can also look at our [Autoscale FAQ](autoscale-faq.yml) if you have other questions.
