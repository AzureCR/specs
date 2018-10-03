# Auto Purge

## Overview

As customers automate their container builds, we're finding customers are filling their container registries with an increasing percentage of images that will never be used, or are simply used for a short period of time.
However, within that percentage, there are a set of images that are critically important to maintain for production deployments. 

Auto Purge provides a means to automatically delete images based on a set of policies. 

## Auto-Purge Scenarios: 

### 1.0 Default policy
 By default, images are automatically deleted after __ days

In this case, images can be deleted sooner. However, if they still exist after __ days, they are automatically deleted.
### 2.0 Quarantine failed

Images that failed a vulnerability scan and are kept in quarantine are automatically deleted after __ days.

Images can be deleted sooner, but are deleted if they reach __ days.

### 3.0 Maintain Released Images

Images released to production are maintained for __ days (months)

In this case, images must be maintained for at least __ months/days and must not be deleted, even if an explicit delete is attempted. To delete a released image, the policy must be changed for the particular image. 

### 4.0 Policy Extension/Update
Certain expiration date policies will need to be extended. 

- 4.1 **Extension**

  For the "release" policy, the expiration date may be set to +6 months. If the image is still running in production for 6 months and 1 day, auto-purge shouldn't delete the image. 

- 4.2 **Policy Update**

  Customers may change the default policy attributes. For instance, Teja may have initially configured release images to be maintained for 6 months. A few months later, it's determined released images will be maintained for 1 year. 

To support this, a policy can be re-assigned to an image. Each time the policy is assigned, the expiration date and attributes are re-applied to the image, copied from the registries policy settings. 

### 5.0 Rolling Expiration Date for Released Images
While an image may be determined to be maintained for 1 year, post production deployment, it's rarely known how long an image will remain in production. 

To automate the [4.1 Scenario](#4.1-Extension), the auto-purge policy will have the ability to monitor nodes, re-applying release policies to images found running. 

Example: 
- An image was deployed on 1/1/18, and set to expire on 7/1/19
- On 1/2/18, the image is found running. The expiration date will be extended to 7/2/19. 
- On 2/1/18, the policy was extended to maintain images for 1 year. 
- On 2/2/18, the image is found running. The expiration date is re-applied using the new 1 year policy. The expiration date is now set to 2/2/19.
- The daily update will continue, updating the release policy for all images found running. 

## Requirements

With the above scenarios, we identify the following requirements:

### 1.0 Expiration Dates

To support individual image expiration dates, new and old policies, images expiration dates are applied to image:tags and manifests. By setting the expiration date to a specific tag and/or manifest, ACR knows exactly when an action must be taken, regardless of changes in policy. This also decouples any need to know when a policy was set, and what the retention policy was at the time it was set.

### 2.0 Policy Management

A given registry can apply a set default policies. The policies are stored at the registry level. The policies are simply named policies with a set of attributes. As a named policy is assigned to a tag or manifest, the attributes are copied to the image. This allows a policy to change without impacting previously set expiration dates.

As an example:

| Policy | Policy | Unit | Can Delete Prior | Recycle |
|---|---|---|---|---|
| *Default* *| 5 | days | True | True |
| *Recycle Bin* *| 15 | days | True | N/A |
| *Quarantine Failed* * | 5 | days | True | False |
| *Untagged* *| 5 | days | True | True |
| Released | 180 | days | False | True |

> **Default**, **Recycle Bin**, **Quarantine Failed** and **Untagged** are system policies and can't be removed. **Released** may be renamed by a customer for individual **Dev**, **Test**, **Prod** policies. 

> **Untagged** are images who's tags have been removed, but the manifest remains.

A customer can add additional policies. The above are likely the defaults that originate with each registry. 

A policy can be set to never expire, and `CanDeletePrior = true` as well. In this case, the policy applied to the specific image would require an update prior to deletion. 

If the policy is set to never expire, but `CanDeletePrior = false`, the image can be deleted at any time, but will not auto-purge.

### 3.0 Image Pinning

Some images can be marked `CanDeletePrior=false`, avoiding deletion until a given date. To delete images with this attribute, the `CanDeletePrior` attribute must first be removed.  

### 4.0 Never Delete

An image can be set to never delete. This means the image has no expiration date, but is "pinned". 
This surfaces as `CanDeletePrior=false` with no expiration date.

### 5.0 Delete By

Some images can be deleted sooner, but will certainly be deleted if they still exist when they reach their expiration date. If an image has an expiration date, but isn't "pinned", it can be deleted sooner.

### 6.0 Recycle Bin

To avoid images being accidentally deleted, deleted images will be placed in a recycle bin for __ days.

A policy is provided for the length of the recycle bin.

### 7.0 Empty Recycle Bin

Customers may need to free up deleted images, reducing their image sizes. Just as customers can do with their desktops, they can manually empty the recycle bin.

### 8.0 Scheduled Purging

ACR will run daily jobs to move images that have reached their expiration date into the recycle bin. It will also delete images in the recycle bin that have reached their expiration date.

As purging is a lower prioirty background process, the granularity of image purging is daily. While most purging will occur daily, the SLA for purging will be within 2 days of an images expiration date to allow for higher-priority system processes.

### 9.0 Deleting images older than __

Regardless of whether a customer has yet enabled auto-purge policies, customers need the ability to delete images based on a date. Running a delete will be a manual job, as the policies will support automated purging. However, a manual delete will not delete images which are required to be maintained until a given date, such as **released** images. To delete images prior to their expiration date, they must be un-pinned `CanDeletePrior=true`

### 10.0 Logging

As policies are updated, an updatedBy user/datetime is applied. If updated by auto-purge, the user will be "system". If a user manually updates a policy, a log will reflect who initiated the change, along with the pre/post values.

### 11.0 Roles
To differentiate administrative policies from routine policies, ACR will support the following new roles

| Role | Action |
| --- | --- |
| acr-purge-policy-writer | Users and Services can update and create default purge policies, such as **Default**, **Recycle Bin**, ... |
| acr-image-purge-writer | Users and services can update individual image policies. This supports batch jobs that must re-apply the released image policy, extending the expiration date. It also supports individual adminstrative users who need to remove the `CanDeletePrior` attribute |