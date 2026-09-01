# ConnectWise PSA Time Entry Error Reference

| Error | Cause | Resolution |
|-------|-------|------------|
| chargeToId required | Missing ticket/project ID | Include chargeToId |
| member required | Missing member reference | Include `member: {id: x}` |
| timeStart required | Missing start time | Include timeStart field |
| Invalid work type | Work type doesn't exist | No tool enumerates work types; omit `workTypeId` and let defaults apply |
| Cannot delete | Time already billed | Cannot delete billed entries |
| Invalid status | Invalid status value | Use Open, Approved, Rejected |
