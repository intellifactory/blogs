---
title: "CloudSharper 0.9.22.0 released"
categories: "cloudsharper,f#,websharper"
abstract: "This release adds cloud workspace management and bug fixes."
identity: "4075,77501"
---
The dashboard is now showing your cloud-synced workspaces as well as your local ones. Separate icons show the availability of your workspaces in local working directory and the cloud storage. You can access the sync up/down commands on the context menu of the workspaces grid or by the More menu above. If both cloud and local copy exists, sync up or down will overwrite the other copy or if newer files exist at the destination, you will have the choice of keeping the newer version of all files.

![](https://i.imgur.com/BeaLDKY.png)

Opening a workspace clone link will now warn if a local clone already exists and ask for opening the last version or create a new clone. However, cloning information is only stored starting from this release so older cloned workspaces won't get notified about.

## Full change log

 * [#561](https://bitbucket.org/IntelliFactory/cloudsharper/issue/561/): Add cloud workspaces management on the dashboard.
 * [#564](https://bitbucket.org/IntelliFactory/cloudsharper/issue/564/): Detect when user is trying to open an online-only workspace.
 * [#560](https://bitbucket.org/IntelliFactory/cloudsharper/issue/560/): Users should be able to log in with their email addresses as well.
 * [#561](https://bitbucket.org/IntelliFactory/cloudsharper/issue/561/): Add cloud workspaces management on the dashboard.
 * [#424](https://bitbucket.org/IntelliFactory/cloudsharper/issue/424/): Fix for using IF.Build.
 * [#447](https://bitbucket.org/IntelliFactory/cloudsharper/issue/447/): Notify user when the number of synced up workspaces reached the quota of his or her account.
 * [#553](https://bitbucket.org/IntelliFactory/cloudsharper/issue/553/): Deleting a file removes its compiler errors in Notification.
 * [#565](https://bitbucket.org/IntelliFactory/cloudsharper/issue/565/):  Opening a clone URL with existing local workspace.


## Known issues

 * [#566](https://bitbucket.org/IntelliFactory/cloudsharper/issue/566/): Fix uploading a previously deleted cloud workspace.
 * [#568](https://bitbucket.org/IntelliFactory/cloudsharper/issue/561/): List workspaces sometimes get called twice if Retry button on dashboard was used .

Happy coding!
