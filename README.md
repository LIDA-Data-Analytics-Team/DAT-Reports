# Power BI Reports

A collection of Power BI reports used and / or created by DAT to provide Management Informatino on DAT activity and LASER.

## Dashboard v2
Used to provide activity and cost to researchers who are using LASER.  

As a first design principle, views are created on the SQL Database to use as data sources within the report. In this way data can be pre-processed outside of the report and within the database. 

### Refresh 
Set to automatically refresh from Power BI Workspace three times a day; 09:00, 13:00 & 17:00.  

Uses SQL Server Authentication against Prism database. the contained user has been made a member of the _db_datareader_ role.  The details are in DAT's KeePass.  

### Row Level Security
Uses [Row Level Security](https://learn.microsoft.com/en-us/power-bi/enterprise/service-admin-rls) to filter visible projects to just those that the viewer is a amember of. Workspace members assigned Admin, Member, or Contributor have edit permission for the dataset and, therefore, RLS doesn’t apply to them.
- _DashboardUsers_ role created in Power BI report, with a filter on data source _vw_LaserActiveGroupMembers_ of `[UserPrincipalName] = userprincipalname()`. 
- Many:many relationship established in report model between _vw_LaserActiveGroupMembers_ and _vw_LaserUsageCosts_ on Projectnumber, with 'Apply security filter in both directions' checked.
- AAD Groups _LRDP-All-Citrix-SafeRoom-Users_ and _LRDP-All-Citrix-Users_ added as members of the role from Power BI workspace. 

In this way all LASER users (ie members of _LRDP-All-Citrix-SafeRoom-Users_ and _LRDP-All-Citrix-Users_) can access the Dashboard as members of the _DashboardUser_ role, and the content is filtered to projects they are a member of. 

### Datasources

#### DimDate
Used because "dates are hard". Provides mapping for dates stored in various formats. Useful when date hierarchies are required.  

Relationships created in Power BI to join to date fields in other data sources.  

#### vw_LaserUsageCosts  
Uses `[dbo].[tblLaserUsageCosts]` as a primary source table.  

Left joins to `[dbo].[vw_AllProjects]` on ProjectNumber, which is extracted as a substring from `[ResourceGroup]` using the following logic:  
```sql
-- most resource groups follow standard naming convention that includes project number. The are a couple of exceptions...
, case when substring([ResourceGroup], 14, 7) = 'picanet'
        then 'p0001'
    when substring([ResourceGroup], 14, 8) = 'picnetv2'
        then 'p0001'
    else substring([ResourceGroup], 14, 5)
    end as [Projectnumber]
```
All records with a Resource Group that doesn't contain a ProjectNumber on the Prism record (as extracted above) are flagged as 'Infra' in the field `[Project]`, which otherwise is a concatenation of ProjectNumber & ProjectName. 

`[ResourceName]` is taken from `[ResourceId]` as the last token in the string.  

`[ResourceType]` & `[ResourceCategory]` are taken as substrings from the source field `[ResourceType]`. Eg:  
> Source ResourceType = microsoft.compute/virtualmachines  
> View ResourceType = virtualmachines  
> View Resource Category = compute  

#### vw_LaserActiveGroupMembers  
Uses `[dbo].[tblLaserAADGroupMembers]` as a primary source table.

Left joins to `[dbo].[vw_AllProjects]` on ProjectNumber, which is extracted as a substring from `[GroupDisplayName]` using the following logic:  
```sql
-- most AD group names follow standard naming convention that includes project number. The are a couple of exceptions...
, case when [GroupDisplayName] like '%picanet%'
        then 'p0001'
    when [GroupDisplayName] like '%picnetv2%'
        then 'p0001'
    else substring([GroupDisplayName], 5, 5)
    end as [Projectnumber]
```
`[UserPrincipalName]` is used in the Row Level Security filter to determine which `[ProjectNumber]` the viewer is a member of and therefore which project details should be permitted for viewing.
