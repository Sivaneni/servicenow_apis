# Export SCTASKs by Assignment Group – curl Steps

Goal: get all SCTASKs for an assignment group, create one folder per SCTASK, download the
associated RITM attachment(s), and save the "Request Details" data as a `.json` file.

Run these in **Git Bash**, in order.

## 0. Setup

```bash
export SN="https://dev405731.service-now.com"
export SN_AUTH="admin:YOUR_PASSWORD"
export GROUP="Hardware"          # assignment group name
```

> The API user needs the `snc_basic_auth_api_access` role (or "Web service access only")
> if the instance has the Basic Auth restriction enabled; otherwise you get `401 User is not authenticated`.

## 1. Discover the fields on the "Request Details" tab

"Request Details" is a custom form section, so run this on the **company instance**:

```bash
curl -s -u "$SN_AUTH" -H "Accept: application/json" -G "$SN/api/now/table/sys_ui_element" \
  --data-urlencode "sysparm_query=sys_ui_section.name=sc_task^sys_ui_section.caption=Request Details^ORDERBYposition" \
  --data-urlencode "sysparm_fields=element,type"
```

- **Field names** (e.g. `request_item.cat_item`, `u_business_justification`) → add them to `sysparm_fields` in step 2.
- **A formatter** (e.g. "Variable Editor") → the tab shows catalog variables; use step 5.

## 2. Get all SCTASKs for the assignment group

```bash
curl -s -u "$SN_AUTH" -H "Accept: application/json" -G "$SN/api/now/table/sc_task" \
  --data-urlencode "sysparm_query=assignment_group.name=$GROUP^active=true" \
  --data-urlencode "sysparm_fields=sys_id,number,short_description,state,assignment_group,assigned_to,request_item,request,opened_at,due_date" \
  --data-urlencode "sysparm_display_value=all" \
  --data-urlencode "sysparm_limit=1000"
```

- `-G --data-urlencode` is required: group names with spaces (e.g. `Service Desk`) cause `400 Bad Request` if put in the URL directly.
- Remove `^active=true` to include closed tasks.
- `sysparm_display_value=all` returns both raw and display values; `request_item.value` is the RITM sys_id used below.
- More than 1000 tasks → page with `&sysparm_offset=1000`, `2000`, …

## 3. List the attachments on the RITM

```bash
RITM_ID="29692305c34c131075acf61d0501312a"     # request_item.value from step 2
curl -s -u "$SN_AUTH" -H "Accept: application/json" -G "$SN/api/now/attachment" \
  --data-urlencode "sysparm_query=table_name=sc_req_item^table_sys_id=$RITM_ID" \
  --data-urlencode "sysparm_fields=sys_id,file_name,content_type,size_bytes"
```

## 4. Download an attachment into the SCTASK's folder

```bash
ATT_ID="<sys_id from step 3>"
mkdir -p "sctasks/SCTASK0010002"
curl -s -u "$SN_AUTH" "$SN/api/now/attachment/$ATT_ID/file" \
  -o "sctasks/SCTASK0010002/laptop_request_details.docx"
```

## 5. Get the RITM details and catalog variables

RITM fields:

```bash
curl -s -u "$SN_AUTH" -H "Accept: application/json" \
  "$SN/api/now/table/sc_req_item/$RITM_ID?sysparm_fields=number,cat_item,requested_for,opened_by,opened_at,stage,state,approval,quantity,price,description&sysparm_display_value=true"
```

Catalog variables (question → answer):

```bash
curl -s -u "$SN_AUTH" -H "Accept: application/json" -G "$SN/api/now/table/sc_item_option_mtom" \
  --data-urlencode "sysparm_query=request_item=$RITM_ID^ORDERBYsc_item_option.order" \
  --data-urlencode "sysparm_fields=sc_item_option.item_option_new.name,sc_item_option.item_option_new.question_text,sc_item_option.value" \
  --data-urlencode "sysparm_display_value=true"
```

## 6. Save the result as JSON

```bash
curl -s -u "$SN_AUTH" -H "Accept: application/json" "<url from step 5>" \
  -o "sctasks/SCTASK0010002/request_details.json"
```

## Resulting folder layout

```
sctasks/
  SCTASK0010002/
    laptop_request_details.docx
    request_details.json
```

## Notes

- On the developer instance, set `GROUP` to a group that has tasks (assign SCTASK0010002 to a group first).
- Next step: a Python script that loops over every SCTASK and builds the folders and JSON automatically,
  using the exact fields returned by step 1.


## link

https://discord.com/channels/1492098192316043349/1561024399073615987



