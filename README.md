# Membership-Retention 

## Notes for ETL to Power BI / Summary of how membership retention was calculated and created

### Salesforce data was used from the TP Membership Renewal Forecast. 
Filters set for extract:
* Membership Name = Access Membership, Cultural, Family, Student
* Finish Date is between 20220601 and 20291201 (to give Lapse Date of 20220701 to 20300101)


### Power BI
##### 1. Import data to Power BI through Microsoft Exchange Online (Analytics email inbox),importing most recent Salesforce report (attachment) in .csv format

= Exchange.Contents("user@email.com)

= Source{[Name="Mail"]}[Data]

= Table.SelectRows(Mail1, each ([Subject] = "Report results (TP Membership Renewal Forecast)"))

= Table.Sort(#"Filtered Rows",{{"DateTimeReceived", Order.Descending}})

= Table.FirstN(#"Sorted Rows",1)

= Table.ExpandTableColumn(#"Kept First Rows", "Attachments", {"AttachmentContent"}, {"AttachmentContent"})

= Table.SelectRows(#"Expanded Attachments", each [Attributes]?[Hidden]? <> true)

= Table.AddColumn(#"Filtered Hidden Files1", "Transform File", each #"Transform File"([AttachmentContent]))

= Table.SelectColumns(#"Invoke Custom Function1", {"Transform File"})

= Table.ExpandTableColumn(#"Removed Other Columns1", "Transform File", Table.ColumnNames(#"Transform File"(#"Sample File")))

= Table.TransformColumnTypes(#"Expanded Table Column1",{{"Status", type text}, {"Membership Name", type text}, {"Membership Purchase Name", type text}, {"Subscription History Name", type text}, {"Finish Date", type date}})



#### SF retention raw 
##### 2. Add Lapse Date column = Finish Date + 30 days, remove Finish Date column

= Table.AddColumn(#"Changed Type", "Lapsed Date", each Date.AddDays([Finish Date], 30))

= Table.TransformColumnTypes(#"Added Lapsed Date",{{"Lapsed Date", type date}})

= Table.SelectColumns(#"Changed Type1",{"Status", "Membership Name", "Membership Purchase Name", "Subscription History Name", "Lapsed Date"})


##### 3. Duplicate SF retention raw dataset twice raw. Rename tables: SF retention multi, SF retention unique 

##### 4. Transformations for SF retention multi; keep duplicates of column Membership Purchase Name (MMS######)

= let columnNames = {"Membership Purchase Name"}, addCount = Table.Group(#"Removed Other Columns", columnNames, {{"Count", Table.RowCount, type number}}), selectDuplicates = Table.SelectRows(addCount, each [Count] > 1), removeCount = Table.RemoveColumns(selectDuplicates, "Count") in Table.Join(#"Removed Other Columns", columnNames, removeCount, columnNames, JoinKind.Inner)


#### 5. Duplicate SF retention multi. Rename tables: SF retention multi (renewed/not renewed), SF retention multi (new/renewal)

#### SF retention multi (renewed/not renewed)
##### 6. Sort rows by Subscription History Name (MSA######) descending (this orders memberships from most recent subscription to oldest subscription)

= Table.Sort(#"Kept Duplicates MMS",{{"Subscription History Name", Order.Descending}})


##### 7. Assign an index to all rows (each row is numbered from 0, at increments of 1), group rows by Membership Purchase Name (MMS######)

= Table.AddIndexColumn(#"Sorted Subscriptions", "Index", 0, 1, Int64.Type)

= Table.Group(#"Added Index", {"Membership Purchase Name"}, {{"Count", each _, type table[Membership Purchase Name=nullable text, Membership Name=nullable text, Status=nullable text, Subscription History Name=nullable text, Lapsed Date=nullable date, New vs Renew=nullable text, Index=number]}})

##### 8. Assign an index to all rows (each row with matching MMS###### is numbered from 1, at increments of 1), name this index Subscription Number. Expand grouped rows

= Table.AddColumn(#"Grouped Rows", "Custom", each Table.AddIndexColumn([Count], "Subscription Number", 1, 1))
  
= Table.SelectRows(#"Added Subscription Number", each true)

= Table.SelectColumns(#"Filtered Rows1",{"Membership Purchase Name", "Custom"})

= Table.ExpandTableColumn(#"Removed Other Columns2", "Custom", {"Status", "Membership Name", "Subscription History Name", "Lapsed Date", "Index", "Subscription Number"}, {"Status", "Membership Name", "Subscription History Name", "Lapsed Date", "Index", "Subscription Number"})

= Table.TransformColumnTypes(#"Expanded Custom",{{"Subscription Number", Int64.Type}, {"Lapsed Date", type date}})

= Table.SelectColumns(#"Changed Type2",{"Membership Purchase Name", "Status", "Membership Name", "Subscription History Name", "Lapsed Date", "Subscription Number"})

= Table.ReorderColumns(#"Removed Other Columns3",{"Status", "Membership Name", "Membership Purchase Name", "Subscription History Name", "Lapsed Date", "Subscription Number"})

= Table.Sort(#"Reordered Columns",{{"Membership Purchase Name", Order.Ascending}})

##### 9. Add column Renew vs. Not Renewed. Most recent subscription has Subscription Number = 1, (therefore it is the only one that has not yet been renewed)

= Table.AddColumn(#"Sorted Rows1", "Renewed vs. Not Renewed", each if [Subscription Number] = 1 then "Not Renewed" else "Renewed")
= Table.SelectColumns(#"Added Renewed vs. Not Renewed",{"Status", "Membership Name", "Membership Purchase Name", "Subscription History Name", "Lapsed Date", "Renewed vs. Not Renewed"})

#### SF retention multi (new/renewal)

* Sort rows by Subscription History Name (MSA######) ascending (this orders memberships from oldest subscription to most recent subscription)

* Assign an index to all rows (each row is numbered from 0, at increments of 1), group rows by Membership Purchase Name (MMS######) 
* Assign an index to all rows (each row with matching MMS###### is numbered from 1, at increments of 1), (name this index Subscription Number), expand grouped rows
* Add column New vs. Renewal. Most recent subscription has Subscription Number =1, (therefore it is the newest)



#### SF retention unique

* Sort rows by Subscription History Name (MSA######) ascending. This orders memberships from oldest subscription to most recent subscription 
* Keep unique values of column Membership Purchase Name (MMS######) 

* Add column Renew vs. Not Renewed. Code: all = "Not Renewed"

* Add column New vs. Renewal. Code: all = "New"

* Append tables as new 

= Table.Combine({#"SF retention unique", #"SF retention multi"})

SF retention multi (new/renewal) and SF retention multi (renewed/not renewed) on column Subscription History Name and name it SF retention multi
Combine tables as new:

SF retention unique and SF retention multi and rename table as SF retention.


Link 'SF retention'[Lapsed Date] to 'Date'[Date] in Model View in Power BI.


An example of new/renewal and renewed/not renewed membership categorisation using this method:

|Membership  |Subscription History |Expiry    |New/Renewal  |Renewed/Not renewed | Notes                                                                    |
| -----      | -----               | -----    | ----        | ---                | ---                                                                      |
|MMS001234   |MSA33333             |01/2023   |Renewal			|Not renewed         | A current membership that is a renewal of a previous subscription.       |
|MMS001234   |MSA22222				     |01/2022		|Renewal			|Renewed             | A non-current membership that was a renewal of a previous subscription.  |
|MMS001234   |MSA11111     				 |01/2021		|New					|Renewed             | The first year of a non-current membership that has since been renewed.  |
|MMS005678	 |MSA44444				     |01/2023		|New					|Not renewed         | The first year of a (current or non-current) membership.                 |
