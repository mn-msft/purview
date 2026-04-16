## 1. Spot check DLPRuleMatch events (broadly)

```kusto
CloudAppEvents
| where TimeGenerated >= ago(90d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType =~ "DLPRuleMatch"
| where isnotempty(ObjectName)
| extend ObjectName = url_decode(ObjectName)
| extend Raw = parse_json(RawEventData)
| extend
    RawIncidentId       = tostring(Raw.IncidentId),
    RawUserId           = tostring(Raw.UserId),
    RawCreationTime     = todatetime(Raw.SharePointMetaData.ItemCreationTime),
    RawLastModifiedTime = todatetime(Raw.SharePointMetaData.ItemLastModifiedTime)
| mv-expand Policy = Raw.PolicyDetails
| extend
    RawPolicyId   = tostring(Policy.PolicyId),
    RawPolicyName = tostring(Policy.PolicyName)
| mv-expand Rule = Policy.Rules
| extend
    RawRuleId   = tostring(Rule.RuleId),
    RawRuleName = tostring(Rule.RuleName)
| project
    DLPTime         = TimeGenerated,
    DLPAction       = ActionType,
    ObjectName,
    DLPUserId       = RawUserId,
    DLPIncidentId   = RawIncidentId,
    DLPPolicyId     = RawPolicyId,
    DLPPolicyName   = RawPolicyName,
    DLPRuleId       = RawRuleId,
    DLPRuleName     = RawRuleName,
    RawEventData,
    Application
```

## 2. Spot check Sharing events (broadly)

```kusto
CloudAppEvents
| where TimeGenerated >= ago(2h)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in~ (
    "CompanyLinkCreated",
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    )
| extend Raw = parse_json(RawEventData)
| extend
    ShareObjectId              = tostring(Raw.ObjectId),
    ShareUserId                = tostring(Raw.UserId),
    SharePermission            = tostring(Raw.Permission),
    ShareSharingLinkScope      = tostring(Raw.SharingLinkScope),
    ShareEV                    = tostring(Raw.EventData),
    ShareTargetUserOrGroupName = tostring(Raw.TargetUserOrGroupName),
    ShareTargetUserOrGroupType = tostring(Raw.TargetUserOrGroupType),
    ShareItemType              = tostring(Raw.ItemType)
| where isnotempty(ShareObjectId)
| project
    ShareTime                  = TimeGenerated,
    ShareAction                = ActionType,
    ShareObjectId,
    ShareItemType,
    ShareUserId,
    SharePermission,
    ShareSharingLinkScope,
    ShareEV,
    ShareTargetUserOrGroupName,
    ShareTargetUserOrGroupType,
    RawEventData,
    Application
```

## 3. Spot check DLPRuleMatch events by specific filepath

Triage specific alerts using specific filepath

```kusto
CloudAppEvents
| where TimeGenerated >= ago(90d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType =~ "DLPRuleMatch"
| where isnotempty(ObjectName)
| extend ObjectName = url_decode(ObjectName)
| extend Raw = parse_json(RawEventData)
| extend
    RawIncidentId       = tostring(Raw.IncidentId),
    RawUserId           = tostring(Raw.UserId),
    RawCreationTime     = todatetime(Raw.SharePointMetaData.ItemCreationTime),
    RawLastModifiedTime = todatetime(Raw.SharePointMetaData.ItemLastModifiedTime)
| where ObjectName contains "<file url from alert>"
| mv-expand Policy = Raw.PolicyDetails
| extend
    RawPolicyId   = tostring(Policy.PolicyId),
    RawPolicyName = tostring(Policy.PolicyName)
| mv-expand Rule = Policy.Rules
| extend
    RawRuleId   = tostring(Rule.RuleId),
    RawRuleName = tostring(Rule.RuleName)
| project
    DLPTime         = TimeGenerated,
    DLPAction       = ActionType,
    ObjectName,
    DLPUserId       = RawUserId,
    DLPIncidentId   = RawIncidentId,
    DLPPolicyId     = RawPolicyId,
    DLPPolicyName   = RawPolicyName,
    DLPRuleId       = RawRuleId,
    DLPRuleName     = RawRuleName,
    RawEventData,
    Application
```

## 4. Spot check Sharing events by specific filepath

Triage specific alerts using specific filepath

```kusto
CloudAppEvents
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in~ (
    "CompanyLinkCreated",
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    )
| extend Raw = parse_json(RawEventData)
| extend
    ShareObjectId              = tostring(Raw.ObjectId),
    ShareUserId                = tostring(Raw.UserId),
    SharePermission            = tostring(Raw.Permission),
    ShareSharingLinkScope      = tostring(Raw.SharingLinkScope),
    ShareEV                    = tostring(Raw.EventData),
    ShareTargetUserOrGroupName = tostring(Raw.TargetUserOrGroupName),
    ShareTargetUserOrGroupType = tostring(Raw.TargetUserOrGroupType),
    ShareItemType              = tostring(Raw.ItemType)
| where ShareObjectId contains "<file url from alert>"
| project
    ShareTime                  = TimeGenerated,
    ShareAction                = ActionType,
    ShareObjectId,
    ShareItemType,
    ShareUserId,
    SharePermission,
    ShareSharingLinkScope,
    ShareEV,
    ShareTargetUserOrGroupName,
    ShareTargetUserOrGroupType,
    RawEventData,
    Application
```

## 5. DLPRuleMatch event + Sharing event correlation

since DLPRuleMatch events always contains objectname while sharing link events does not, pivoted to using Raw.ObjectId for sharing links instead

```kusto
let dlp_files = toscalar(
    CloudAppEvents
    | where TimeGenerated >= ago(90d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType =~ "DLPRuleMatch"
    | where isnotempty(ObjectName)
    | extend ObjectName = url_decode(ObjectName)
    | summarize make_set(ObjectName)
    );
CloudAppEvents
| where TimeGenerated >= ago(20m)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in~ (
    "CompanyLinkCreated", 
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    )
| extend Raw = parse_json(RawEventData)
| extend ShareObjectId = tostring(Raw.ObjectId)
| where isnotempty(ShareObjectId)
| where ShareObjectId in (dlp_files)
| extend
    ShareUserId                = tostring(Raw.UserId),
    SharePermission            = tostring(Raw.Permission),
    ShareSharingLinkScope      = tostring(Raw.SharingLinkScope),
    ShareEV                    = tostring(Raw.EventData),
    ShareTargetUserOrGroupName = tostring(Raw.TargetUserOrGroupName),
    ShareTargetUserOrGroupType = tostring(Raw.TargetUserOrGroupType),
    ShareItemType              = tostring(Raw.ItemType)
| project
    TimeGenerated,
    ShareTime                  = TimeGenerated,
    ShareAction                = ActionType,
    ShareObjectId,
    ShareItemType,
    ShareUserId,
    SharePermission,
    ShareSharingLinkScope,
    ShareEV,
    ShareTargetUserOrGroupName,
    ShareTargetUserOrGroupType,
    RawEventData,
    Application
```

## Original correlation rule

working as Sentinel Analytics rule but inconsistent due to correlating on objectname which doesn't always have data from the sharing links side

```kusto
let dlp_files = toscalar(
    CloudAppEvents
    | where TimeGenerated >= ago(90d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType =~ "DLPRuleMatch"
    | where isnotempty(ObjectName)
    | extend ObjectName = url_decode(ObjectName)
    | summarize make_set(ObjectName)
);
CloudAppEvents
| where TimeGenerated >= ago(20m)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in~ (
    "CompanyLinkCreated", 
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    )
| where isnotempty(ObjectName)
| where ObjectName in (dlp_files)
| extend Raw = parse_json(RawEventData)
| extend
    ShareUserId                = tostring(Raw.UserId),
    SharePermission            = tostring(Raw.Permission),
    ShareSharingLinkScope      = tostring(Raw.SharingLinkScope),
    ShareEV                    = tostring(Raw.EventData),
    ShareTargetUserOrGroupName = tostring(Raw.TargetUserOrGroupName),
    ShareTargetUserOrGroupType = tostring(Raw.TargetUserOrGroupType),
    ShareItemType              = tostring(Raw.ItemType)
| project
    TimeGenerated,
    ShareTime             = TimeGenerated,
    ShareAction           = ActionType,
    ShareObjectName       = ObjectName,
    ShareItemType,
    ShareUserId,
    SharePermission,
    ShareSharingLinkScope,
    ShareEV,
    ShareTargetUserOrGroupName,
    ShareTargetUserOrGroupType,
    RawEventData,
    Application
```

## Test correlation query with folder support

```kusto
let dlp_files = toscalar(
    CloudAppEvents
    | where TimeGenerated >= ago(90d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType =~ "DLPRuleMatch"
    | where isnotempty(ObjectName)
    | extend ObjectName = url_decode(ObjectName)
    | summarize make_set(ObjectName)
    );
let dlp_parent_folders = toscalar(
    CloudAppEvents
    | where TimeGenerated >= ago(90d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType =~ "DLPRuleMatch"
    | where isnotempty(ObjectName)
    | extend ObjectName = url_decode(ObjectName)
    | extend ParentFolder = tostring(extract("^(.+/)[^/]+$", 1, ObjectName))
    | where isnotempty(ParentFolder)
    | summarize make_set(ParentFolder)
    );
CloudAppEvents
| where TimeGenerated >= ago(30m)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in~ (
    "CompanyLinkCreated", 
    "CompanyLinkUpdated",
    "SharingLinkCreated", 
    "SharingLinkUpdated",
    "AnonymousLinkCreated", 
    "AnonymousLinkUpdated",
    "SecureLinkCreated", 
    "SecureLinkUpdated",
    "SharingSet", 
    "AddedToSharingLink",
    "AddedToSecureLink"
    )
| extend Raw = parse_json(RawEventData)
| extend
    ShareObjectId              = tostring(Raw.ObjectId),
    ShareItemType              = tostring(Raw.ItemType)
| where isnotempty(ShareObjectId)
| where (ShareItemType =~ "File" and ShareObjectId in (dlp_files))
    or (ShareItemType =~ "Folder" and ShareObjectId in (dlp_parent_folders))
| extend
    ShareUserId                = tostring(Raw.UserId),
    SharePermission            = tostring(Raw.Permission),
    ShareSharingLinkScope      = tostring(Raw.SharingLinkScope),
    ShareEV                    = tostring(Raw.EventData),
    ShareTargetUserOrGroupName = tostring(Raw.TargetUserOrGroupName),
    ShareTargetUserOrGroupType = tostring(Raw.TargetUserOrGroupType)
| project
    TimeGenerated,
    ShareTime                  = TimeGenerated,
    ShareAction                = ActionType,
    ShareObjectId,
    ShareItemType,
    ShareUserId,
    SharePermission,
    ShareSharingLinkScope,
    ShareEV,
    ShareTargetUserOrGroupName,
    ShareTargetUserOrGroupType,
    RawEventData,
    Application
```
