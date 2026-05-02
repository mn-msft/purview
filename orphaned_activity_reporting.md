> ## overview

<pre style="background: transparent; padding: 0; margin: 0; font-family: 'JetBrains Mono', monospace; line-height: 1.25;">
mbx (reference list)                       CloudAppEvents
┌────────────────────┐                    ┌──────┬────────────────────┬─────────────┐
│ PrimarySmtpAddress │                    │ ...  │ UPN                │ ...         │
├────────────────────┤                    ├──────┼────────────────────┼─────────────┤
│ bob@contoso.com    │ ·················► │      │                    │             │
│ susan@contoso.com  │ ·················► │      │ bob@contoso.com    │             │
│ sam@contoso.com    │ ···· no match ···· │      │                    │             │
└────────────────────┘                    │      │ susan@contoso.com  │             │
                                          │      │ bob@contoso.com    │             │
                                          │      │                    │             │
                                          │      │                    │             │
                                          │      │ bob@contoso.com    │             │
                                          │      │                    │             │
                                          │      │ susan@contoso.com  │             │
                                          └──────┴────────────────────┴─────────────┘
                                                 │
                                                 │ sam@contoso.com never appears
                                                 │ 
                                                 ▼
                                      lookup kind=leftouter
                                                 │
                                                 ▼
                            ┌────────────────────┬──────────────────┐
                            │ PrimarySmtpAddress │ Status           │
                            ├────────────────────┼──────────────────┤
                            │ bob@contoso.com    │ Active           │  ← matched
                            │ susan@contoso.com  │ Active           │  ← matched
                            │ sam@contoso.com    │ Stale / Orphaned │  ← no rows found in CloudAppEvents
                            └────────────────────┴──────────────────┘
</pre>

> ## query concepts

<pre style="background: transparent; padding: 0; margin: 0; font-family: 'JetBrains Mono', monospace; line-height: 1.25;">
mbx (CSV)               CloudAppEvents (30 days, Exchange Online only)
┌────────────────────┐  ┌─────────────────────┬────────────────────┬──────────────────────────────────────────────────────────────┐
│ PrimarySmtpAddress │  │ Timestamp           │ ActionType         │ RawEventData                                                 │
├────────────────────┤  ├─────────────────────┼────────────────────┼──────────────────────────────────────────────────────────────┤
│ bob@contoso.com    │  │ 2026-04-28 10:22:00 │ MailItemsAccessed  │ {"MailboxOwnerUPN":"bob@contoso.com","UserId":"bob@..."}     │
│ susan@contoso.com  │  │ 2026-05-01 08:14:00 │ Send               │ {"MailboxOwnerUPN":"bob@contoso.com","UserId":"bob@..."}     │
│ sam@contoso.com    │  │ 2026-05-02 11:03:00 │ MoveToDeletedItems │ {"MailboxOwnerUPN":"bob@contoso.com","UserId":"bob@..."}     │
└────────────────────┘  │ 2026-04-01 09:45:00 │ MailItemsAccessed  │ {"MailboxOwnerUPN":"susan@contoso.com","UserId":"susan@..."} │
        │               │ ...                 │ ...                │ ...                                                          │
        │               └─────────────────────┴────────────────────┴──────────────────────────────────────────────────────────────┘
        │                                                          │
        │                                                          │ parse_json(RawEventData)
        │                                                          │ extend MailboxOwnerUPN, UserId
        │                                                          │ extend ResolvedUPN = case(...)
        │                                                          │ where ResolvedUPN in (mbx)
        │                                                          │ project ResolvedUPN, Timestamp, ActionType
        │                                                          │ summarize arg_max(Timestamp, ActionType) by ResolvedUPN
        │                                                          │
        │                                                          ▼
        │                           activeMailboxes (derived, one row per mailbox)
        │                           ┌───────────────────┬─────────────────────┬────────────────────┐
        │                           │ ResolvedUPN       │ LastSeen            │ LastOperation      │
        │                           ├───────────────────┼─────────────────────┼────────────────────┤
        │                           │ bob@contoso.com   │ 2026-05-02 11:03:00 │ MoveToDeletedItems │
        │                           │ susan@contoso.com │ 2026-04-01 09:45:00 │ MailItemsAccessed  │
        │                           └───────────────────┴─────────────────────┴────────────────────┘
        │                                                          │
        └──────────────────────────────────────────────────────────┘
                                    │
                                    │ lookup kind=leftouter
                                    │ mbx is left side — all CSV rows preserved
                                    │ sam@contoso.com has no match → null LastSeen
                                    │
                                    ▼
Final Result
┌────────────────────┬──────────────────┬─────────────────────┬────────────────────┐
│ PrimarySmtpAddress │ Status           │ LastSeen            │ LastOperation      │
├────────────────────┼──────────────────┼─────────────────────┼────────────────────┤
│ sam@contoso.com    │ Stale / Orphaned │ null                │ null               │  ← no match
│ susan@contoso.com  │ Active           │ 2026-04-01 09:45:00 │ MailItemsAccessed  │
│ bob@contoso.com    │ Active           │ 2026-05-02 11:03:00 │ MoveToDeletedItems │
└────────────────────┴──────────────────┴─────────────────────┴────────────────────┘
</pre>

> ## query for mailbox activity

### create reference list (table) in memory

```kusto
let mbx = datatable(PrimarySmtpAddress: string)
[
    "bob@contoso.com",
    "susan@contoso.com",
    "sam@contoso.com"
];
mbx
```

### read reference list from external source

```kusto
let mbx = materialize(
    externaldata (PrimarySmtpAddress: string) [
    @"https://demo.blob.core.windows.net/demo/mailboxes.csv"
    h@"?<sas_token>"
    ]
    with (format='csv', ignorefirstrecord=true)
    | where PrimarySmtpAddress != "PrimarySmtpAddress"
    | project PrimarySmtpAddress = tolower(PrimarySmtpAddress)
    );
let activeMailboxes =
    CloudAppEvents
    | where Timestamp >= ago(30d)
    | where Application == "Microsoft Exchange Online"
    | extend Raw = parse_json(RawEventData)
    | extend MailboxOwnerUPN = tostring(Raw.MailboxOwnerUPN)
    | extend UserId = tostring(Raw.UserId)
    | project-away Raw
    | extend ResolvedUPN = tolower(case(
        isnotempty(MailboxOwnerUPN), MailboxOwnerUPN,
        isnotempty(UserId), UserId,
        ""
        ))
    | where isnotempty(ResolvedUPN)
    | where ResolvedUPN in (mbx)
    | project ResolvedUPN, Timestamp, ActionType
    | summarize arg_max(Timestamp, ActionType) by ResolvedUPN
    | project ResolvedUPN, LastSeen = Timestamp, LastOperation = ActionType;
mbx
| lookup kind=leftouter activeMailboxes on $left.PrimarySmtpAddress == $right.ResolvedUPN 
| extend Status = case(
    isnotempty(LastSeen), "Active",
    "Stale / Orphaned"
    )
| project PrimarySmtpAddress, Status, LastSeen, LastOperation
| sort by Status asc, LastSeen asc
```

> ## query w/ watchlist

```kusto
let mbx = materialize(
    _GetWatchlist('bw_senders')
    | project PrimarySmtpAddress = tolower(PrimarySmtpAddress)
    );
let activeMailboxes =
    CloudAppEvents
    | where TimeGenerated >= ago(30d)
    | where Application == "Microsoft Exchange Online"
    | extend Raw = parse_json(RawEventData)
    | extend MailboxOwnerUPN = tostring(Raw.MailboxOwnerUPN)
    | extend UserId = tostring(Raw.UserId)
    | project-away Raw
    | extend ResolvedUPN = tolower(case(
        isnotempty(MailboxOwnerUPN), MailboxOwnerUPN,
        isnotempty(UserId), UserId,
        ""
        ))
    | where isnotempty(ResolvedUPN)
    | where ResolvedUPN in (mbx)
    | project ResolvedUPN, TimeGenerated, ActionType
    | summarize arg_max(TimeGenerated, ActionType) by ResolvedUPN
    | project ResolvedUPN, LastSeen = TimeGenerated, LastOperation = ActionType;
mbx
| lookup kind=leftouter activeMailboxes on $left.PrimarySmtpAddress == $right.ResolvedUPN
| extend Status = case(
    isnotempty(LastSeen), "Active",
    "Stale / Orphaned"
    )
| project PrimarySmtpAddress, Status, LastSeen, LastOperation
| sort by Status asc, LastSeen asc
```

> ## lightweight option w/ toscalar

only returns mailboxes with activity, no additional info

```kusto
let mbx_set = toscalar(
    externaldata (PrimarySmtpAddress: string) [
    @"https://demo.blob.core.windows.net/demo/mailboxes.csv"
    h@"?<sas_token>"
    ]
    with (format='csv', ignorefirstrecord=true)
    | where PrimarySmtpAddress != "PrimarySmtpAddress"
    | project PrimarySmtpAddress = tolower(PrimarySmtpAddress)
    | summarize make_set(PrimarySmtpAddress)
    );
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application == "Microsoft Exchange Online"
| extend Raw = parse_json(RawEventData)
| extend MailboxOwnerUPN = tostring(Raw.MailboxOwnerUPN)
| extend UserId = tostring(Raw.UserId)
| project-away Raw
| extend ResolvedUPN = tolower(case(
    isnotempty(MailboxOwnerUPN), MailboxOwnerUPN,
    isnotempty(UserId), UserId,
    ""
    ))
| where isnotempty(ResolvedUPN)
| where ResolvedUPN in (mbx_set)
| project ResolvedUPN
| distinct ResolvedUPN
// | summarize EventCount = count() by ResolvedUPN
```

> ## query for mailbox traffic

```kusto
let mbx = materialize(
    externaldata (PrimarySmtpAddress: string) [
    @"https://demo.blob.core.windows.net/demo/mailboxes.csv"
    h@"?<sas_token>"
    ]
    with (format='csv', ignorefirstrecord=true)
    | where PrimarySmtpAddress != "PrimarySmtpAddress"
    | project PrimarySmtpAddress = tolower(PrimarySmtpAddress)
    );
let activeMailboxes =
    EmailEvents
    | where Timestamp >= ago(30d)
    | extend RecipientEmailAddress = tolower(RecipientEmailAddress)
    | where RecipientEmailAddress in (mbx)
    | project RecipientEmailAddress, Timestamp
    | summarize LastSeen = max(Timestamp) by RecipientEmailAddress
    | project RecipientEmailAddress, LastSeen;
mbx
| lookup kind=leftouter activeMailboxes on $left.PrimarySmtpAddress == $right.RecipientEmailAddress
| extend Status = case(
    isnotempty(LastSeen), "Active",
    "Stale / Orphaned"
    )
| project PrimarySmtpAddress, Status, LastSeen
| sort by Status asc, LastSeen asc
```

> ## query for group mail traffic

```kusto
let groups = materialize(
    externaldata (PrimarySmtpAddress: string) [
    @"https://demo.blob.core.windows.net/demo/groups.csv"
    h@"?<sas_token>"
    ]
    with (format='csv', ignorefirstrecord=true)
    | where PrimarySmtpAddress != "PrimarySmtpAddress"
    | project PrimarySmtpAddress = tolower(PrimarySmtpAddress)
    );
let activeGroups =
    EmailEvents
    | where Timestamp >= ago(30d)
    | extend DistributionList = tolower(DistributionList)
    | where DistributionList in (groups)
    | project DistributionList, Timestamp
    | summarize LastSeen = max(Timestamp) by DistributionList
    | project DistributionList, LastSeen;
groups
| lookup kind=leftouter activeGroups on $left.PrimarySmtpAddress == $right.DistributionList
| extend Status = case(
    isnotempty(LastSeen), "Active",
    "Stale / Orphaned"
    )
| project GroupSmtpAddress = PrimarySmtpAddress, Status, LastSeen
| sort by Status asc, LastSeen asc
```
