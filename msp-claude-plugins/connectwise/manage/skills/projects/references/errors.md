# ConnectWise PSA Project Error Reference

| Error | Cause | Resolution |
|-------|-------|------------|
| Company required | Missing company reference | Include `company: {id: x}` |
| Name required | Missing project name | Provide name field |
| Invalid status | Status doesn't exist | No tool enumerates project statuses; confirm the value in the PSA |
| Invalid manager | Member doesn't exist | Verify member ID |
| Template not found | Invalid projectTemplateId | Find templates with `cw_search_projects` filtered on the template project type |
