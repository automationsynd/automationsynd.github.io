---
layout: post
title:  "Extending SCCM Software Center with Windows 10 Toast Notifications"
date:   2018-12-26 09:42:46 -0500
categories: sccm
---
*Written by Jeremy Brun*

# The What, Why, and Where

Microsoft SCCM has built-in functionality for notifying the user when new applications or updates become available or are required within Software Center. Unfortunately those functions limit your ability to guide the user towards taking action or inferring a level of importance based on the type of deployment.

As a result, I set out to create an application that extends the notification capabilites of Software Center. My initial goals were...

- Maintain a small footprint so that it does not tie up machine resources.
- Pop a toast notification to let the user know if there is an Operating System Deployment (OSD) available or required on their machine.
- Give the user helpful and actionable information that boosts their confidence in the IT organization supporting them and makes them feel like more of a partner than a subject.

So with those goals in mind I set out to discover what I could accomplish.

So how *do you* go about providing an enhanced UX for informing the user that they should click install? Here are a couple of options that I explored while I was trying to answer this question...

- [Using PowerShell in an SCCM DCM baseline to pop a toast notification](https://smsagent.wordpress.com/2018/06/15/using-windows-10-toast-notifications-with-configmgr-application-deployments/)
    - Great approach, but does not allow you to leverage some of the more advanced features of toast notifications (more on this later).
- [Using PowerShell in an SCCM DCM baseline to pop a custom WPF (Windows Presentation Framework) GUI](https://foxdeploy.com/series/learning-gui-toolmaking-series/)
    - This gives you more control over what content you show and what actions you allow the user to take. But it also steps up the complexity and overhead required for administration. Because it is more complex to administer it can be more prone to mistakes during changes.
    - This could also make more sense in a non Windows 10 environment due to lack of toast notifications functionality.

I'm sure there are many other approaches that could be cobbled together from the corners of the interwebs, but after reading up on [Universal Windows Platform (UWP) toast notifications](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/adaptive-interactive-toasts) it really did seem like the way to go. Not only would I be leveraging a tool that is baked into the OS (and adaptive to the user's personal visual/display settings), but it would also be an easier learning/training curve for our users since they are *already seeing and interacting with toast notifications from other applications*.

# The End Result

The results from the approach outlined below were an immediate and staggering increase in user participation on our first Windows 10 build update. We can only compare to past experiences with monthly security patches at this point. And in those cases we were used to seeing a 5-10% voluntary user participation rate in the availability period leading up to the deadline. That availability period is about 7 days on average. Now, with our first Windows 10 build update we blew past 20% in the first two business days that the update was available.

This was made possible by simply displaying a toast notification reminding the user of the available/required OSD every time they log in to or unlock their machine.

![Example](/assets/example.jpg)

Because this is a *reminder* type notification it will be presented to the user and it will persist on the screen until they initiate an action against it. Their options are as follows.

- Clicking *Dismiss* will dismiss the notification until the next time it is triggered which would be whatever your default trigger cadence is (mine is at login or unlock). This would have the same effect as if the user were to dismiss one or all notifications from within the taskbar Action Center.
- Clicking *Snooze* after selecting a timeframe will trigger a run-once scheduled task to execute at the time the user selected and *only* as that user.
- Clicking on the notification body will open up Software Center to the Operating Systems page. From there the user can select relevant OSD's and choose to install them.

# Notes From the Field

This is a background application that is driven by the system Task Scheduler. Initially after installation of the application there is a global scheduled task that is triggered whenever a user logs in to *OR* unlocks their workstation. This scheduled task is configured to run as the current interactive user.

When the application is invoked via command line with specific arguments it will run and check in WMI for any OSD's that are available or required.

If the application finds applicable OSD's it will evaluate the following criteria.

- Is there an OSD **required** in the future that has not already successfully installed?
    - Yes -> Is there already a task snoozed by the user for a future reminder?
        - Yes -> exit
        - No -> pop a toast notification then exit
    - No -> Is there an OSD **available** that has not already successfully installed?
        - Yes -> Is there already a task snoozed by the user for a future reminder?
            - Yes -> exit
            - No -> pop a toast notification then exit
        - No -> exit

First, I want to be clear that although we are using the UWP toast notification framework, we are not developing a [UWP application](https://docs.microsoft.com/en-us/windows/uwp/get-started/universal-application-platform-guide). UWP is a type of application that is cross-platform and typically it would be delivered via an app store like the Microsoft Store. That being said, the UWP toast notification framework is supported for native desktop applications within Windows 10. My project is a Windows Presentation Framework (WPF) application that has all windows hidden.

As someone with .NET development experience, but no initial understanding of the UWP toast notification framework I leaned ***heavily*** on Microsoft Doc's quick start guides. These were absolute gems.

- [Toast content](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/adaptive-interactive-toasts)
- [Toast UX guidance](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/toast-ux-guidance)
- [Sending local toast notifications from desktop C# apps](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/send-local-toast-desktop)

That last article provides a whole bunch of C# examples and then links to a [working sample on GitHub](https://github.com/WindowsNotifications/desktop-toasts). I went ahead and downloaded this sample code and started playing with it. Here are some lessons I learned through experimentation. Many of these are going to be obvious for the average Windows developer, but it was all new and interesting to me so I want to highlight them.

- In order to pop a toast notification that can programatically react to user interaction you must have an application registered in the local OS notification platform. There is code in the provided sample that demonstrates the steps for accomplishing this.
- One of the *gotchas* that was the most interesting to me was that the application has to have a shortcut in the Start Menu. 🤔 This has to do with the way the OS invokes the application when the user interacts with a toast notification. There are specific properties you must set on the shortcut in order to make this work (see code sample).
    - This is an interesting mind game when the goal is an application that the user doesn't run or interact with directly. What I settled on was to make my appication's EXE open up Software Center when it is invoked from the Start Menu shortcut or elsewhere without any arguments passed to it.

Another tool I found immensely helpful during testing was [Notification Visualizer](https://www.microsoft.com/en-us/p/notifications-visualizer/9nblggh5xsl1). This tool allows you to build out notifications using the [UWP toast notification XML schema](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/toast-xml-schema).

You can mock up your own toast notification similar to the one used in my application by pasting the following XML into the Notification Visualizer app.

```xml
<toast scenario="reminder">
  <visual>
    <binding template="ToastGeneric">
      <text>Windows 10 Semiannual Update 1809</text>
      <text>Required by 5:00 PM, Fri, 1/25/19</text>
      <text>You can visit Software Center at any time to view more details or install updates before the required deadline.</text>
      <image placement="hero" src="https://picsum.photos/364/180?image=1043"/>
    </binding>
  </visual>
  <actions>
    <input id="snoozeTime" type="selection" title="Click Snooze to be reminded in:" defaultInput="15">
      <selection id="15" content="15 minutes"/>
      <selection id="60" content="1 hour"/>
      <selection id="240" content="4 hours"/>
      <selection id="480" content="8 hours"/>
    </input>
    <action activationType="system" arguments="snooze" hint-inputId="snoozeTime" content=""/>
    <action activationType="system" arguments="dismiss" content=""/>
  </actions>
</toast>
```

Once I gained an adequate understanding of UWP toast notifications and their capabilities I started to integrate that with what I knew about the SCCM [Client SDK namespace](https://docs.microsoft.com/en-us/sccm/develop/reference/core/clients/sdk/client-sdk-wmi-classes) within WMI on managed machines. This is the namespace that Software Center already gets most of it's information from that it presents to the user.

Of particular interest was the [`CCM_Program`](https://docs.microsoft.com/en-us/sccm/develop/reference/core/clients/sdk/ccm_program-client-wmi-class) class which has information for all task sequences currently applicable to the local machine.

Once I had task sequence data I could then determine if it is an applicable OSD. Unfortunately there is no "I'm an OSD!" flag on the task sequence. You could use text flags or even the category property to track your OSD task sequences in your environment, but after a lot of noodling around I settled on what *appears* to be the basic way Software Center itself decides. That is to simply check that 1) that it is, indeed, a Task Sequence (`TaskSequence == true`), and 2) that the SCCM task sequence user notification setting is enabled (`NotifyUser == true`). The latter is a default setting for OSD task sequences ([according to this little gem](https://configurationmanager.uservoice.com/users/700390873-ilkka)).

If I'm able to find an OSD task sequence then I evaluate my criteria to determine whether a toast notification should be displayed.

# Future Enhancements

- Extend the existing functionality to regular security update/patch deployments.
- Add notifications for failed deployments that are detected. These notifications would either prompt the user towards contacting our org's Help Desk OR towards submitting a service request in the ticketing system OR *just submit a service request for them*. Because automation and convenience. 😉
- Add a *Schedule* option to available actions. This would allow the user to schedule the update in question to be triggered at a time they select. This would be an enhancement to the business hours scheduling functionality. There are pros and cons to this idea that warrant further consideration.
